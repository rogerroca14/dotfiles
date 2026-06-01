# Arquitectura · High-level

Documento de referencia para explicar **qué es** y **cómo se construye/despliega** una app agéntica en Databricks Apps usando este stack. Sin tutorial paso-a-paso · solo el panorama.

---

## 1. Qué tipo de app es

**Agente conversacional con tool-calling sobre datos Databricks.**

- Usuario chatea en lenguaje natural.
- Un LLM decide qué herramienta (tool) llamar.
- Las herramientas son funciones Python que consultan UC, llaman APIs, ejecutan lógica.
- El LLM interpreta el resultado y responde.

Aplica a: copilots analíticos, asistentes de pricing, investigación de fraude, support copilots, agentes de gobierno de datos.

> No es: dashboard, ETL, modelo offline, chatbot scripted con árboles de decisión.

---

## 2. Stack en una imagen

```
┌──────────────────────────────────────────────────────────────────┐
│  USUARIO (browser)                                                │
└──────────────────────────┬───────────────────────────────────────┘
                           │ HTTPS + OAuth (X-Forwarded-User)
                           ▼
┌──────────────────────────────────────────────────────────────────┐
│  DATABRICKS APPS · serverless container                           │
│  ┌────────────────────┐    ┌─────────────────────────────────┐   │
│  │ Frontend           │ ─► │ Backend                          │   │
│  │ Next.js chat       │    │ FastAPI + MLflow AgentServer    │   │
│  │ Vercel AI SDK      │    │ OpenAI Agents SDK loop          │   │
│  └────────────────────┘    └────────────┬────────────────────┘   │
└──────────────────────────────────────────┼───────────────────────┘
                                           │
        ┌──────────────────────────────────┼─────────────────────┐
        ▼                                  ▼                     ▼
┌───────────────────┐         ┌──────────────────────┐  ┌────────────────┐
│ Model Serving     │         │ SQL Warehouse        │  │ MLflow         │
│ Claude / GPT etc. │         │ (consulta UC tables) │  │ tracing + eval │
│ Pay-per-token     │         │ Auto-start serverless│  │                │
└───────────────────┘         └──────────────────────┘  └────────────────┘
```

Todo arriba vive en el workspace Databricks · no hay nubes terceras.

---

## 3. Capas y responsabilidades

| Capa | Función | Tecnología |
|---|---|---|
| **UI** | Captura input, renderiza respuestas streaming | React + Next.js + Vercel AI SDK |
| **Proxy** | Forwarding del frontend al backend | Express dentro del mismo container |
| **Agent loop** | Decide cuándo llamar tools, cuándo responder | OpenAI Agents SDK (`Runner.run`) |
| **Tools** | Funciones Python tipadas que ejecutan lógica/SQL | `@function_tool` decorator |
| **LLM** | Razonamiento + selección de tools | Databricks Model Serving (Claude, GPT, Llama, etc.) |
| **Data** | Verdad agregada del negocio | Unity Catalog + SQL Warehouse |
| **Tracing** | Observabilidad de cada turno | MLflow autolog → experimento UC |
| **Auth** | Identidad del usuario y del app | OAuth U2M (dev) / Service Principal (prod) |
| **Deploy** | Empaquetado y configuración | Databricks Asset Bundles (DAB) |

---

## 4. Flujo de un request

```
1. Usuario escribe pregunta en el chat
2. Next.js → POST proxy → FastAPI /invocations
3. AgentServer invoca handler @invoke()
4. create_agent() arma:
     instructions (system prompt + contexto dinámico cacheado)
     tools (lista de funciones @function_tool)
     model_settings (tool_choice=required en iter 0)
5. Runner.run() corre el loop:
     - LLM call → tool_calls
     - ejecuta tool(s) en Python
     - tool query Warehouse / API / lo que sea
     - resultado vuelve al LLM
     - repite hasta que LLM responde sin tools
6. Response devuelve a UI con artifacts (tablas, charts, metadata)
7. MLflow registra todo el turno como trace
```

Latencia típica: 2-6 segundos para query analítica simple. 10-20s si el LLM necesita 3+ iteraciones.

---

## 5. Cómo se desarrolla

| Paso | Quién | Herramientas |
|---|---|---|
| Definir scope (qué responde, qué no) | Producto + Tech | Documento + system prompt draft |
| Diseñar tools (1 por familia de pregunta) | Tech + dominio | JSON schemas / type hints |
| Escribir tools en Python | Dev | OpenAI Agents SDK + `databricks-sql-connector` |
| Probar local | Dev | `uv run start-app` + browser localhost |
| Iterar prompts y tools | Dev + dominio | Logs locales + MLflow traces |
| Evaluar accuracy | Dev | MLflow `agent-evaluate` + scorers custom |
| Deploy a dev / staging / prod | Dev | DAB (`databricks bundle deploy`) |

**Filosofía:** loop tight. Cambias 1 tool, reinicias local, probas 1 pregunta. Si va, deployas. Sin merge largos.

---

## 6. Cómo se despliega

```
1. Code en git (branch feature)
2. databricks bundle validate → catch config errors
3. databricks bundle deploy → sube código + crea/actualiza recursos UC
4. databricks bundle run <app_name> → arranca app
5. databricks apps logs --follow → monitorea
```

**DAB (`databricks.yml`) declara:**
- Nombre y comando del app.
- Recursos que el app necesita (experimento, warehouse, endpoints, Genie spaces, vector indexes).
- Permisos al service principal del app (CAN_USE, CAN_QUERY, CAN_MANAGE).
- Env vars de producción.

DAB es **infra-as-code para Databricks Apps**. Mismo bundle deploya a `dev`, `staging`, `prod` con un `--target`.

---

## 7. Quién gestiona qué

| Componente | Gestiona | Configuración manual |
|---|---|---|
| Container del app | Databricks (serverless) | Nada |
| Modelo LLM | Databricks Model Serving | Solo elegir endpoint en `agent.py` |
| SQL Warehouse | Equipo de data | Auto-start ON + auto-stop razonable |
| Unity Catalog (tablas) | Equipo de gobierno de datos | GRANT SELECT al SP del app |
| MLflow experiment | Dev del agente | Crear 1 vez vía CLI o UI |
| Service Principal del app | Databricks (auto-creado por DAB) | Permisos vía `databricks.yml` |

**Punto importante:** el dev del agente NO toca infra cluster/network. Todo lo de plataforma lo abstrae Databricks Apps. El dev solo se enfoca en lógica + prompts + tools.

---

## 8. Quién paga qué

| Recurso | Costo típico |
|---|---|
| Databricks Apps container | ~$30-50/mes idle, escala con uso |
| LLM (pay-per-token) | $3-15/M tokens · Claude Sonnet ~$3 in / $15 out |
| SQL Warehouse | Lo más caro · ~$0.7/DBU · serverless cobra solo cuando query |
| MLflow + UC storage | Negligible |
| Network / TLS | Cero (incluido) |

App marginal en infra Mibanco es bajo · cargo dominante es **costo de LLM + warehouse**.

---

## 9. Diferencias clave vs alternativas

| Stack | Cuándo | Por qué este es mejor / peor |
|---|---|---|
| **Genie Space directo** | Q&A simple sobre 1 tabla con NL2SQL | Más rápido de armar · menos control de cifras (puede inventar joins) |
| **Streamlit / Gradio + LangChain** | Prototipo rápido stand-alone | OK para POC · más fricción de auth y deploy en Databricks |
| **MLflow ResponsesAgent + chat-next** (este stack) | Producción · agent con tools + UI consistente Databricks-native | Auth nativo + tracing nativo + DAB deploy + chat-next pulido |
| **Chat custom + API REST plana** | Front muy custom | Más trabajo · pierdes la chat-next que ya viene resuelta |

Este stack = sweet spot para **agentes analíticos production-grade en Databricks**.

---

## 10. Patrones de diseño que recomiendo replicar

1. **Tools en Python, no NL2SQL libre.** SQL queries pre-aprobadas en `@function_tool` → control + auditabilidad + cero alucinaciones de joins.
2. **Contrato `ToolResult` con `data_preview` vs `data`.** 25 records al LLM, lo grande al UI. Evita explosiones de contexto.
3. **`ArtifactBuffer` per-request.** Pasado como `Runner.run(context=...)`. Cada tool stashea su artifact, el handler lo surface al UI en `custom_outputs`. Limpio.
4. **System prompt con bloques fijos** (rol, reglas inviolables, conocimiento, formato, scope). Reglas en MAYÚSCULAS, no párrafo.
5. **Contexto dinámico cacheado al startup.** Query 1 vez los periodos disponibles, inyecta al prompt. Evita alucinar fechas.
6. **`tool_choice="required"` en iter 0 + `reset_tool_choice=True`.** Fuerza tool en la primera vuelta · permite respuesta libre después.
7. **MLflow autolog por defecto.** Cada turno queda trazado. Sin custom logging adicional.
8. **DAB para todo lo de plataforma.** No `databricks apps` manual.
9. **Per-product subpackages.** `tools/<producto>/` escala mejor que tools/ flat con prefijos.
10. **Frontend canónico + script de customización.** Clone gitignored + parche regex idempotente.

---

## 11. Cuándo NO usar este stack

- **Q&A puro sobre docs PDF**: usá Knowledge Assistant + Vector Search · este stack es overkill.
- **Pipeline ETL puro**: usá Jobs/Workflows · este stack es para chat.
- **Modelo offline batch**: usá Model Serving + Jobs · no Apps.
- **Frontend muy custom (no tipo chat)**: clonate solo el backend, monta tu frontend aparte.
- **No tenés Databricks**: portate a otro stack · este es Databricks-native.

---

## 12. Próximos puntos de evolución (lo que la plataforma agrega cada release)

- **Supervisor API**: offload del loop al servidor de Databricks. Menos código en tu agent · más declarativo.
- **Background mode**: tasks >30s sin timeout HTTP.
- **Multi-agent supervisor**: 1 agent orquestador + N agents especialistas. Útil cuando crece scope.
- **Lakebase memory**: conversación persistente entre sesiones. Por ahora memoria es per-request.
- **Vector Search nativo en tools**: ya disponible vía MCP.

---

**TL;DR:** un agent en este stack es **Python + system prompt + tools + DAB**. Databricks Apps + serverless + tracing nativo te quitan toda la fricción de infra. El dev escribe lógica · la plataforma escala, autentica, traza y deploya.
