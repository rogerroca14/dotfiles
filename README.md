# 🎨 Roger's Dotfiles for Oh My Posh

Este repositorio contiene mi configuración personalizada de terminal usando [Oh My Posh](https://ohmyposh.dev/), diseñada para funcionar tanto en **Windows (PowerShell)** como en **Ubuntu (Bash)**.

## Características

- 💠 Tema inspirado en el estilo Takuya
- 🕒 Hora en formato 12h (AM/PM)
- 🧠 Información del entorno virtual
- 🪟 Icono del sistema operativo (Windows o Linux) dinámico
- 🔤 Nerd Fonts + estilo minimalista
- ⚙️ Scripts para instalación rápida en cada sistema

## Estructura

```markdown
dotfiles/
├── windows/
│   └── oh-my-posh/
│       └── roger-takuya.omp.json
├── ubuntu/
│   └── oh-my-posh/
│       └── roger-takuya.omp.json
└── install.ps1    # Script opcional para aplicar el tema en Windows
```

## ⚙️ Requisitos

- [Oh My Posh](https://ohmyposh.dev/docs/installation)
- [Nerd Font](https://www.nerdfonts.com/font-downloads) (recomendado: Caskaydia Cove Nerd Font)
- PowerShell o Bash según el sistema operativo

---

## 🚀 Instalación en Windows

1. Clona el repositorio:

```powershell
git clone https://github.com/tuusuario/dotfiles.git
```

2. Aplica el tema (ajusta la ruta si es necesario):
```powershell
oh-my-posh init pwsh --config "$env:USERPROFILE\dotfiles\windows\oh-my-posh\roger-takuya.omp.json" | Invoke-Expression
```

## 🐧 Instalación en Ubuntu
1. Clona el repositorio:
```bash
git clone https://github.com/tuusuario/dotfiles.git
```
2. Aplica el tema desde Bash:
```bash
eval "$(oh-my-posh init bash --config ~/dotfiles/ubuntu/oh-my-posh/roger-takuya.omp.json)"
```
> También puedes añadir esa línea al final de tu ~/.bashrc o ~/.bash_profile.
