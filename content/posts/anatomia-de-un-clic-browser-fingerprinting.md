---
title: "Anatomía de un clic: qué filtra tu dispositivo con solo abrir un enlace"
date: 2026-06-10
draft: false
summary: "Demo educativa de exposición de datos del lado del cliente. Un servidor Flask mínimo captura IP, GeoIP, User-Agent y fingerprint completo del navegador (canvas, audio, WebRTC, GPU) con un solo clic, sin pedir ningún permiso. El objetivo es entender qué se puede obtener de verdad — y qué es mito."
tags: ["web-security", "fingerprinting", "python", "flask", "privacidad", "investigación-defensiva"]
categories: ["Proyecto", "Seguridad"]
toc: true
---

> Demo educativa de **exposición de datos del lado del cliente** (browser fingerprinting).
> Un experimento para entender —y aprender a defenderse de— lo que un simple enlace
> puede averiguar sobre quien lo abre, sin pedir un solo permiso.

---

## Aviso ético y legal

Este proyecto se construyó y se probó **exclusivamente contra mis propios dispositivos, con mi consentimiento, en un entorno controlado**, con fines educativos y de documentación.

**No es una herramienta de doxxing.** Es una demostración defensiva: el objetivo es *exponer* cuánta información filtra un navegador para luego explicar **cómo protegerse**. Capturar datos de terceros sin su consentimiento es ilegal en la mayoría de jurisdicciones y va en contra del propósito de este trabajo.

---

## Qué demuestra este proyecto

La gente asume que un enlace "solo abre una página". En realidad, el navegador entrega de forma **pasiva y silenciosa** (sin prompts, sin clicks de aceptación) una cantidad enorme de datos que, combinados, permiten **re-identificar un dispositivo concreto** entre visitas distintas — la base del rastreo publicitario moderno.

Este repositorio implementa un servidor de captura mínimo que documenta, paso a paso:

- **Qué** se puede obtener de verdad con un clic (y qué es mito).
- **Cómo** se combinan esas señales en una huella única estable.
- **Por qué** la geolocalización por IP "falla" en datos móviles (y eso es un hallazgo, no un error).
- **Cómo defenderse** de cada técnica.

---

## Arquitectura

El flujo completo de una captura, de principio a fin:

```
  Víctima (consentida)            Servidor Flask              Servicios externos
 ┌────────────────────┐         ┌──────────────┐            ┌──────────────────┐
 │  Abre  /link        ├────────▶│  1. Lee       │           │                  │
 │                     │         │  cabeceras    ├──────────▶│  ipinfo.io       │
 │                     │         │  (IP, UA…)    │  GeoIP    │  (ciudad, ISP)   │
 │                     │◀────────┤  2. Sirve     │◀──────────┤                  │
 │  Ejecuta JS oculto  │  HTML   │  página +     │           └──────────────────┘
 │  (fingerprinting)   │  + JS   │  id de fila   │
 │                     ├────────▶│  3. POST      │
 │                     │/collect │  /collect     │           ┌──────────────────┐
 │  Redirige a Google  │◀────────┤  actualiza    │           │  cloudflared      │
 └────────────────────┘   302   │  la fila      │           │  túnel HTTPS      │
                                 └──────┬───────┘            │  público          │
                                        │                    └──────────────────┘
                                  ┌─────▼──────┐
                                  │  logs.db    │  ◀── panel /logs (protegido por token)
                                  │  (SQLite)   │
                                  └────────────┘
```

1. **`GET /link`** — Captura del lado servidor (IP, GeoIP, User-Agent, idioma, referer), inserta una fila y devuelve una página mínima con un `id` embebido.
2. **JavaScript del cliente** — recolecta el fingerprint del navegador de forma silenciosa.
3. **`POST /collect`** — recibe esos datos y **actualiza la misma fila** por su `id`.
4. **Redirección** a un destino legítimo (Google), para no levantar sospechas.
5. **`GET /logs?token=…`** — panel de administración protegido para revisar lo capturado.

La exposición a internet se hace con un **túnel de Cloudflare** (`cloudflared`), que además aporta **HTTPS automático** — requisito para que funcionen APIs como `crypto.subtle` (la huella) o WebRTC.

---

## Qué se captura

### Lado servidor (cabeceras HTTP)

| Dato | Fuente | Notas |
|---|---|---|
| Dirección IP | `CF-Connecting-IP` | La IP real del cliente tras el túnel |
| Ciudad / País / ISP | GeoIP (ipinfo.io) | Aproximado; ver _Expectativa vs realidad_ |
| Sistema operativo | User-Agent | |
| Navegador | User-Agent | |
| Idioma | `Accept-Language` | |
| Origen | `Referer` | De dónde llegó el clic |

### Lado cliente (JavaScript, sin permisos)

| Dato | API | Disponibilidad |
|---|---|---|
| Resolución y densidad de pantalla | `screen`, `devicePixelRatio` | Universal |
| Zona horaria + offset UTC | `Intl.DateTimeFormat` | Universal |
| Núcleos de CPU | `navigator.hardwareConcurrency` | Universal (Safari capa a 8) |
| RAM aproximada | `navigator.deviceMemory` | Solo Chromium |
| GPU real + fabricante | WebGL `UNMASKED_RENDERER` | Universal |
| Idiomas completos | `navigator.languages` | Universal |
| Plataforma / touch points | `navigator.platform`, `maxTouchPoints` | Universal |
| Tipo de red, downlink, RTT | `navigator.connection` | Solo Chromium |
| Nivel de batería + carga | `navigator.getBattery()` | Solo Chromium |
| **Canvas fingerprint** | Canvas 2D → hash | Universal |
| **Audio fingerprint** | `OfflineAudioContext` → hash | Universal |
| **Fuentes instaladas** | Medición de texto | Universal |
| **IPs por WebRTC** | `RTCPeerConnection` + STUN | Universal (con mitigaciones) |
| **Estado de permisos** | `navigator.permissions.query` | Chromium (lee sin pedir prompt) |
| **Huella combinada** | SHA-256 de señales estables | **El identificador del dispositivo** |

---

## Las técnicas de fingerprinting, explicadas

### Canvas fingerprint
Se dibuja el mismo texto y formas en un `<canvas>` invisible. El resultado en píxeles **varía ligeramente según GPU, drivers, sistema de antialiasing y versión del SO**. Hasheando esa imagen se obtiene un identificador estable del dispositivo.

### Audio fingerprint
Se genera una onda con `OfflineAudioContext` y se procesa con un `DynamicsCompressor`. Cómo el hardware/software procesa esa señal produce diferencias mínimas pero **consistentes por dispositivo**.

### Fuentes instaladas
Se mide el ancho y alto de una cadena renderizada. Si al pedir una fuente concreta las dimensiones cambian respecto a la fuente base, esa fuente **está instalada**. La lista de fuentes delata el SO y el software instalado.

### WebRTC
`RTCPeerConnection` puede revelar direcciones IP (locales o, vía un servidor STUN, la pública) incluso **detrás de algunos proxies/VPN**. Históricamente fue una técnica clásica para des-anonimizar usuarios de VPN.

### Permissions API
Permite **leer el estado** de permisos (cámara, micrófono, geolocalización, notificaciones) — `prompt` / `granted` / `denied` — **sin disparar ningún popup**. Revela qué permisos has concedido en el pasado.

---

## La huella: re-identificación sin cookies

El verdadero objetivo no es la lista de datos sueltos, sino **combinarlos en un único hash SHA-256**. Se calcula **solo con las señales estables** (User-Agent, pantalla, zona horaria, idiomas, núcleos, RAM, GPU, plataforma, canvas, audio y fuentes), excluyendo a propósito las que cambian entre visitas (batería, red, viewport).

> **Resultado:** si abres dos enlaces distintos desde el mismo dispositivo y navegador, la `huella` es **idéntica**. Eso es re-identificación **sin cookies** — exactamente cómo te rastrea la publicidad aunque borres tus cookies o uses modo incógnito.

**Matiz importante:** la huella identifica **dispositivo + navegador**, no el dispositivo por sí solo. Abrir con Chrome y luego con Firefox en el mismo teléfono produce huellas distintas (cambia el User-Agent y el canvas).

---

## Expectativa vs realidad

El mayor valor educativo del proyecto está en desmontar lo que prometen los tutoriales de YouTube:

| Lo que se promete | Lo que pasa de verdad |
|---|---|
| "Su ubicación exacta" | En **datos móviles** la IP geolocaliza al **gateway del operador (CGNAT)**, normalmente en otra ciudad. No es tu casa. |
| "El modelo exacto del teléfono" | Chrome en Android **congeló el User-Agent** (UA Reduction). Se obtiene un "Android 10" genérico aunque tengas Android 14. iPhone nunca dio el modelo. |
| "Su número / IMEI / contactos" | **Imposible** desde un navegador. Mito total. |
| "GPS preciso automático" | Solo con la API de geolocalización **y permiso explícito** (el popup). Aquí se omitió para mantener todo silencioso. |
| IP, ISP, idioma, zona horaria, pantalla, GPU, huella | Esto sí, y es bastante. |

**Diferencias por navegador observadas en pruebas reales:**

| Señal | Android Chrome | iPhone Safari |
|---|---|---|
| RAM, tipo de red, batería | disponibles | `null` (Safari no implementa esas APIs) |
| GPU | "Adreno / Mali…" | "Apple GPU" (genérico) |
| Núcleos | real | capado a 8 por privacidad |
| WebRTC (IP local) | a menudo ofuscada (`.local` por mDNS) | ofuscada |

> Que Safari y los móviles modernos devuelvan menos datos **no es un fallo del proyecto: es la prueba de que las defensas de privacidad funcionan.**

---

## Seguridad del propio servidor de captura

Una reflexión que se volvió parte central del proyecto:

> **Una herramienta para espiar visitantes puede ser atacada por un visitante. Quien construye el panel de captura también es un blanco.**

Por eso el código aplica dos defensas concretas:

**1. Prevención de inyección SQL (lista blanca de columnas).**
Los nombres de columna en SQL no se pueden parametrizar con `?`. Como las claves del JSON las controla el cliente, construir el `UPDATE` con ellas directamente sería inyectable. La solución: solo se aceptan claves que estén en una **lista blanca fija** (`CAMPOS_CLIENTE`); cualquier otra se ignora.

**2. Prevención de XSS almacenado (escapado de salida).**
El User-Agent, la GPU o las fuentes son **datos controlados por el visitante**. Un atacante puede enviar un User-Agent como `<script>…</script>`. Si el panel `/logs` los inyectara sin escapar, ese script se ejecutaría **en el navegador del administrador** al revisar los logs — XSS almacenado contra el propio operador. Se neutraliza escapando toda la salida con `html.escape`.

El panel `/logs` además está protegido por un **token** vía variable de entorno.

---

## Lo que no puede obtener un navegador

Para cerrar la desinformación, estas cosas son **imposibles** desde una página web:

- Número de teléfono / IMEI / número de serie
- Contactos, SMS, llamadas
- Archivos del dispositivo
- Dirección MAC
- Nombre real o identidad
- Otras apps instaladas

---

## Cómo defenderse

| Amenaza | Mitigación |
|---|---|
| Geolocalización por IP | VPN de confianza (cambia la IP y el ISP visibles) |
| Fugas de IP por WebRTC | Desactivar WebRTC o usar extensiones que lo bloqueen |
| Canvas / Audio fingerprint | Tor Browser, Brave, Firefox con `resistFingerprinting` |
| User-Agent y APIs invasivas | Navegadores que los reducen o falsean; mantener todo actualizado |
| Permisos | Revisar y revocar; negar por defecto |
| Rastreo general | Modo estricto anti-rastreo, bloqueadores, compartimentar navegadores |

---

## Stack técnico

Python 3 + Flask · waitress · SQLite · requests · JavaScript vanilla · cloudflared · Web APIs (Canvas, WebGL, WebAudio, WebRTC, Permissions, Network Information, Battery)

---

## Instalación y uso

```bash
pip install flask waitress requests

# Token del panel (PowerShell)
$env:LOGS_TOKEN = "un-token-secreto-y-largo"

python app.py
```

Exponer con Cloudflare:

```bash
cloudflared tunnel --url http://localhost:5000
```

| Ruta | Función |
|---|---|
| `GET /link` | Página de captura (redirige tras capturar) |
| `POST /collect` | Recibe el fingerprint del cliente |
| `GET /logs?token=…` | Panel de administración (protegido) |

---

## Aprendizajes

- La superficie de exposición de un navegador es **mucho mayor** de lo que parece, pero también **mucho menor** que los mitos populares.
- El fingerprinting funciona **sin cookies y sin permisos**; combatirlo requiere medidas activas.
- La geolocalización por IP en móvil es engañosa (gateway del operador, no ubicación física).
- Construir herramientas ofensivas obliga a pensar en defensa: el propio servidor de captura es un blanco (SQLi, XSS almacenado).
- Documentar la brecha **expectativa vs realidad** es tan valioso como la captura en sí.
