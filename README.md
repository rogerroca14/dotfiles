# Roger Takuya OMP Theme
Tema personalizado de Oh My Posh basado en `takuya`, optimizado para PowerShell en Windows y Bash en Ubuntu.

## Instalación
### Windows (PowerShell)
```powershell
oh-my-posh init pwsh --config "$env:LOCALAPPDATA\Programs\oh-my-posh\themes\roger-takuya.omp.json" | Invoke-Expression
````

### Ubuntu (Bash)
1. Abrir terminal
2. Clonar repo
```bash
git clone https://github.com/rogerroca14/dotfiles.git ~/.dotfiles
```
3. Verificar
```bash
ls ~/.dotfiles/roger-takuya.omp.json
```
4. Actualiza tu configuración de Oh My Posh para que use este archivo. Si estás usando Bash, abre o edita tu archivo ~/.bashrc:
```bash
nano ~/.bashrc
```
Y busca la línea donde cargas Oh My Posh, debería ser algo como:
```bash
eval "$(oh-my-posh init bash --config ~/.dotfiles/roger-takuya.omp.json)"
```
4. Guarda y cierra (Ctrl + O, luego Enter, y Ctrl + X), y recarga la terminal:
```bash
eval "$(oh-my-posh init bash --config ~/.dotfiles/roger-takuya.omp.json)"
```

## Requisitos
* [Oh My Posh](https://ohmyposh.dev)
* Nerd Font (como Fira Code o Cascadia Code)

## Nota
> El ícono del sistema cambia según el SO. Tema compartido entre Windows y Ubuntu.
