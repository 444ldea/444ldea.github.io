# Contexto del proyecto — Portafolio de seguridad (Hugo)

Este archivo te pone en contexto. Lo lee Claude Code automáticamente al iniciar.

## Qué es

Portafolio personal de un **estudiante de seguridad informática**, construido con Hugo (sitio estático, diseño propio sin tema externo). El objetivo es mostrar proyectos/writeups a reclutadores y a la comunidad.

## El perfil que proyecta

Interés deliberadamente amplio: **blue team, red team, bug bounty, sandboxing, análisis de tráfico, compliance**. La narrativa NO es "experto en todo", sino "curiosidad ofensiva y defensiva en etapa de exploración". El hero lo resume: *"Investigo cómo se rompen los sistemas — y cómo se defienden."* Mantener esa voz: honesta, técnica, sin grandilocuencia.

## Decisiones de diseño (respetarlas al iterar)

- **Estética**: "instrumento de análisis". Fondo slate profundo (no negro puro), acento **ámbar** (color de log/alerta), tipografía **mono** para datos y sans para lectura.
- **Evitar el cliché** de portafolio hacker (negro + verde neón Matrix). Fue una decisión consciente.
- Dark/light automático, responsive, accesible (focus visible, reduced-motion respetado).
- Tokens de color y tipografía viven en `assets/css/main.css` (variables CSS al inicio del archivo).

## Estructura

```
hugo.toml                              config (editar baseURL, author, enlaces)
content/
  about.md                             página "Sobre mí"
  posts/
    deteccion-malware-iptv-android-tv.md   writeup 1 (REAL, completo)
    sandbox-apk-analisis.md                writeup 2 (PLANTILLA, a completar)
layouts/                               plantillas HTML propias
assets/css/main.css                    estilos / identidad visual
static/images/                         diagramas SVG de los writeups
static/files/                          CV (cv.pdf) y descargables
.github/workflows/deploy.yml           deploy automático a GitHub Pages
```

## Los dos writeups

1. **deteccion-malware-iptv-android-tv.md** — caso REAL y completo. Se detectó beaconing C2 (intervalo de 20s, jitter 0) hacia dominios DGA desde una app IPTV pirata (XuperTV) en una smart TV Sony, vía análisis de capturas de OPNsense. Tiene dos diagramas SVG embebidos (metodología y timeline). No reescribir el contenido técnico sin pedir; está verificado contra datos reales de pcap.

2. **sandbox-apk-analisis.md** — PLANTILLA de ejemplo (continuación natural del caso 1: analizar el APK en sandbox aislado). Tiene la estructura lista con secciones a rellenar. Está en `draft: false` pero conviene completarlo con datos reales o ponerlo en `draft: true` hasta entonces. Un writeup con texto de relleno resta credibilidad.

## Pendientes / próximos pasos

- [ ] Completar el segundo writeup (sandbox) con un análisis real, o dejarlo en draft.
- [ ] Editar `hugo.toml`: baseURL real, nombre, enlaces (github/linkedin/email).
- [ ] Reescribir `content/about.md` con la voz del autor.
- [ ] Añadir CV en `static/files/cv.pdf`.
- [ ] (Opcional) capturas reales de OPNsense en el writeup 1, con IP/MAC ofuscadas.
- [ ] Desplegar: el workflow ya existe; falta hacer push y activar Pages (Settings → Pages → Source: GitHub Actions).

## Comandos útiles

```bash
hugo server -D      # preview local con borradores en http://localhost:1313
hugo --minify       # build de producción a public/
hugo new posts/mi-proyecto.md   # nuevo writeup
```

## Notas de trabajo (para mantener al iterar)

- Al añadir writeups, seguir el front matter de los existentes (title, date, summary, tags, categories).
- Los SVG embebidos usan colores hardcodeados que combinan con el fondo oscuro; si se cambian los tokens de color del sitio, revisar que sigan legibles.
- El workflow inyecta baseURL automáticamente desde la config de Pages: no hace falta acertarlo a mano en hugo.toml para que funcione en GitHub.
