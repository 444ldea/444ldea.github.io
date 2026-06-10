# Portafolio de seguridad — sitio Hugo

Sitio estático construido con [Hugo](https://gohugo.io). Diseño propio (sin tema externo), enfocado en un perfil de seguridad ofensiva y defensiva.

## Requisitos

- **Hugo Extended** v0.120 o superior. Es importante la versión *extended* porque el CSS se procesa con Hugo Pipes (minify + fingerprint).

Instalación de Hugo:

- macOS: `brew install hugo`
- Windows: `winget install Hugo.Hugo.Extended` o `choco install hugo-extended`
- Linux (Debian/Ubuntu): descarga el `.deb` *extended* desde las [releases de GitHub](https://github.com/gohugoio/hugo/releases), o `sudo snap install hugo`
- Verifica con: `hugo version` (debe decir `extended`)

## Levantar el sitio en local

Desde la carpeta del proyecto:

```bash
hugo server -D
```

Abre http://localhost:1313. El flag `-D` muestra también borradores (`draft: true`). Los cambios se recargan solos.

## Generar el sitio para publicar

```bash
hugo --minify
```

Genera la carpeta `public/` con el sitio estático listo para subir.

## Qué editar antes de publicar

1. **`hugo.toml`** — cambia `baseURL`, `title`, `author` y tus enlaces (`github`, `linkedin`, `email`).
2. **`content/about.md`** — reescribe la página "Sobre mí" con tu voz.
3. **`static/files/cv.pdf`** — coloca tu CV con ese nombre exacto (el botón ya apunta ahí).
4. **Hero de la home** — el titular vive en `layouts/index.html` (o añade `heroTitle` y `heroSubtitle` en `[params]` de `hugo.toml`).

## Añadir un nuevo writeup

```bash
hugo new posts/mi-nuevo-proyecto.md
```

Edita el archivo creado en `content/posts/`, pon `draft: false` cuando esté listo. Soporta Markdown, bloques de código con resaltado, tablas e imágenes/SVG embebidos (en `static/images/`).

## Estructura

```
hugo.toml                  configuración
content/
  about.md                 página "Sobre mí"
  posts/                    writeups (un .md por proyecto)
layouts/                    plantillas HTML (diseño propio)
assets/css/main.css         estilos (identidad visual)
static/
  images/                   diagramas SVG
  files/                    CV y descargables
```

## Publicar gratis

- **GitHub Pages**: sube el repo, usa la GitHub Action oficial de Hugo (`actions/hugo`). Pon `baseURL` con tu URL de Pages.
- **Netlify / Cloudflare Pages**: conecta el repo, comando de build `hugo --minify`, directorio de salida `public`. Detectan Hugo solos.

---

Diseño: estética de instrumento de análisis (slate + ámbar de log), mono para datos y sans para lectura. Responsive, dark/light automático, accesible (focus visible, reduced-motion).

## Deploy automático (ya configurado)

El repo incluye `.github/workflows/deploy.yml`: cada `git push` a `main` construye y publica el sitio en GitHub Pages automáticamente. Para activarlo la primera vez:

1. Sube el proyecto a un repo de GitHub.
2. En el repo: **Settings → Pages → Build and deployment → Source: GitHub Actions**.
3. Haz un push a `main`. La pestaña **Actions** mostrará el build; al terminar, tu sitio queda publicado.

No necesitas configurar `baseURL` a mano: el workflow lo inyecta desde la config de Pages al construir.
