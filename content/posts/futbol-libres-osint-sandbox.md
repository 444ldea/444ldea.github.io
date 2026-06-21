---
title: "futbol-libres.su — OSINT Pasivo y Análisis Dinámico en Sandbox"
date: 2026-06-21
draft: false
summary: "Investigación completa sobre un sitio de streaming ilegal de la Copa Mundial FIFA 2026. OSINT pasivo (WHOIS, DNS, headers, HTML) seguido de análisis dinámico en REMnux con tcpdump y tshark. Se identificaron dos correos del operador, dos backends de streaming ocultos bajo base64, una red de popunders RTB con Shadow DOM anti-adblocker, y múltiples errores de OPSEC."
tags: ["OSINT", "análisis-dinámico", "REMnux", "sandbox", "DNS", "fingerprinting", "investigación-defensiva", "tcpdump", "tshark"]
categories: ["Investigación", "Seguridad"]
toc: true
---

| Campo | Detalle |
|---|---|
| **Tipo** | OSINT Pasivo + Análisis Dinámico en Sandbox |
| **Objetivo** | `https://futbol-libres.su/` |
| **Fecha** | 21 de junio de 2026 |
| **Analista** | Felipe A. |
| **Entorno dinámico** | REMnux 7 en VirtualBox (modo NAT, snapshot pre-análisis) |
| **Clasificación** | Portafolio profesional — uso educativo |

---

## 1. Resumen ejecutivo

`futbol-libres.su` es un sitio de streaming ilegal de contenido deportivo (principalmente fútbol y eventos de la Copa Mundial FIFA 2026) que opera bajo el TLD `.su` (Soviet Union) para evadir jurisdicciones con aplicación efectiva de DMCA/copyright. La infraestructura está alojada en hosting bulletproof en Ucrania (Virtual Systems LLC / vsys.host) con TTL DNS de 60 segundos para rotación rápida de IPs y resistencia a takedowns.

El modelo de negocio descansa en una red publicitaria de popunders (acscdn.com / adexchangerapid.com) que monetiza el tráfico a través de subastas RTB en tiempo real. En el momento del análisis, los anunciantes eran casas de apuestas (jugabet.com, betano.com) y plataformas de inversión (etoro.com), además de marketplaces (aliexpress.com). No se detectó malware ni exploits activos, aunque la infraestructura de redirección podría ser weaponizada si el operador vende impresiones a actores maliciosos.

Se identificaron dos correos electrónicos atribuibles al operador, dos backends de streaming ocultos bajo codificación base64, y una arquitectura deliberadamente diseñada para maximizar la dificultad de análisis y bloqueo.

---

## 2. Metodología

La investigación se dividió en tres fases:

**Fase 1 — OSINT Pasivo:** recopilación de información sin interactuar con el sitio. Incluye WHOIS, resolución DNS, análisis de headers HTTP, lectura del HTML fuente, Certificate Transparency logs, y consultas a APIs públicas de DNS (Google DNS over HTTPS). Esta fase no genera tráfico hacia el objetivo y no expone la IP del analista.

**Fase 2 — Análisis Dinámico:** navegación controlada en un entorno aislado (REMnux 7 en VirtualBox, modo NAT) con captura de red simultánea mediante tcpdump. Se tomó un snapshot de la VM antes del análisis para poder restaurar el estado limpio. Se usó Firefox con DevTools abierto. Se abrió el sitio, se interactuó con él para disparar los popunders, y se analizó el PCAP resultante con tshark.

**Fase 3 — Análisis de componentes:** descarga y análisis manual de scripts clave (`aclib.js`, `smallscripts.js`) y del iframe `/agenda/` para entender la arquitectura completa.

> **Nota técnica:** Las fases 1 y 3 pueden ejecutarse desde cualquier entorno. La fase 2 requiere sandbox aislado porque el código JavaScript del sitio se ejecuta en el navegador. REMnux en modo NAT garantiza que si el sitio intentara comprometer la máquina, el impacto quedaría contenido dentro de la VM.

---

## 3. OSINT Pasivo — Fase 1

### 3.1 WHOIS y registrador

```
Dominio:        futbol-libres.su
Registrador:    TCINET (administrador del TLD .su, Russia)
Estado:         REGISTERED, DELEGATED
Creado:         2020-10-08
Expira:         2027-10-08
Email registrante: martinjoe19411@gmail.com
```

El TLD `.su` fue el ccTLD de la Unión Soviética, asignado en 1990 antes de la disolución de la URSS. Los operadores de sitios de piratería lo eligen precisamente porque las notificaciones DMCA y las órdenes de bloqueo de cortes occidentales no tienen efecto directo sobre TCINET.

El email `martinjoe19411@gmail.com` apareció en el WHOIS. Este es un dato de atribución importante y un error de OPSEC: el operador no usó un servicio de privacidad y Google tiene datos de esta cuenta.

El año de creación (2020) y la expiración (2027) muestran compromiso de largo plazo — no es un sitio creado para una campaña corta.

### 3.2 DNS — Registros A

```bash
dig A futbol-libres.su
```

```
45.11.57.200
185.254.197.23
128.0.104.19
138.226.244.112
194.42.205.89
212.86.116.38
```

Todas las IPs pertenecen a **Virtual Systems LLC** (AS49530), también conocida como `vsys.host`, con sede en Ucrania — hosting de tipo bulletproof que ignora explícitamente notificaciones DMCA y de abuso.

**El TTL de 60 segundos** es una técnica anti-takedown. Si un ISP bloquea una IP, el DNS resuelve a una diferente en menos de un minuto. El operador mantiene un pool de 6+ IPs activas simultáneamente.

### 3.3 DNS — Nameservers

```bash
dig NS futbol-libres.su
```

```
a.p-dns.com
b.p-dns.biz
c.p-dns.biz
d.p-dns.info
```

Cuatro nameservers en tres TLDs diferentes (`.com`, `.biz`, `.info`). Para que el dominio dejara de funcionar habría que dar de baja los cuatro en tres registradores distintos — los intentos de takedown por suspensión de NS son prácticamente inviables.

### 3.4 DNS — Registros adicionales (MX, TXT, SOA)

**MX:** `10 mail.futbol-libres.su` — pero este subdominio no tiene registro A. El correo no está gestionado desde el dominio; el operador probablemente usa ProtonMail directamente.

**TXT:**
```
google-site-verification=2w2FXFQ4BqF6z9RHnFmu5Q53Ut808494H0AVnfsFwQE
```
El operador verificó la propiedad en **Google Search Console**. Aunque el HTML tiene `noindex`, la verificación permite monitorear tráfico orgánico y recibir alertas de penalizaciones. El token está vinculado a la cuenta de Google del operador.

**SOA — hallazgo crítico:**
```
cp.p-dns.com. joezm5a.proton.me. 2026060831 60 3600 604800 86400
```

En formato SOA, el `.` reemplaza al `@`, por lo que esto corresponde a **`joezm5a@proton.me`** — el segundo identificador del operador, deliberadamente privado (ProtonMail cifrado E2E). Contrasta fuertemente con el Gmail del WHOIS:

- `martinjoe19411@gmail.com` → email de registro, error de OPSEC temprano
- `joezm5a@proton.me` → email real de operación, E2E

El serial `2026060831` indica última actualización el 8 de junio de 2026, revisión 31 — confirmación de mantenimiento activo continuo.

### 3.5 Subdominios descubiertos

| Subdominio | Tipo | Destino | Hosting |
|---|---|---|---|
| `www.futbol-libres.su` | CNAME → A | Root domain → 6 IPs | Virtual Systems LLC, UA |
| `cdn.futbol-libres.su` | CNAME | `flsw.b-cdn.net` → `185.93.1.246` | BunnyNet CDN (EU) |
| `mail.futbol-libres.su` | — | NXDOMAIN | — |

`cdn.futbol-libres.su` hace CNAME a **BunnyNet** (bunny.net), CDN comercial legítimo con sede en Eslovenia. TTL de 35 segundos — más agresivo aún que el de las IPs principales. Bloquear el subdominio requiere precisión: no se puede bloquear BunnyNet entero sin afectar miles de sitios legítimos.

### 3.6 HTTP Headers

```bash
curl -sI https://futbol-libres.su/
```

```
HTTP/1.1 200 OK
Server: nginx
X-Powered-By: Engintron
Content-Type: text/html; charset=UTF-8
```

**Ausencias notables:**

- **`Content-Security-Policy`: AUSENTE** — el sitio puede cargar recursos de cualquier dominio y ejecutar JS inline sin restricciones.
- **`Strict-Transport-Security`: AUSENTE** — los usuarios en HTTP no son forzados a HTTPS.
- **`X-Frame-Options`: AUSENTE** — puede ser embebido en iframes de otros dominios (clickjacking).
- **`X-Content-Type-Options`: AUSENTE** — expone a MIME sniffing.

**Engintron** es un panel de control para nginx muy usado en hosting cPanel de entornos tolerantes a abusos.

### 3.7 Análisis estático del HTML principal

**Google Analytics:**
```html
gtag('config', 'G-L0N11PVD63');
```
ID vinculado a la cuenta de Google del operador — otro vector de atribución.

**Red de popunders:**
```html
<script src="https://acscdn.com/script/aclib.js"></script>
<script>aclib.runPop({zoneId: '9195464'});</script>
```
El `zoneId: '9195464'` identifica la "zona" del sitio en la red de ads — el identificador con el que el operador cobra.

**iframe /agenda/:**
```html
<iframe src="/agenda/" ...></iframe>
```
El contenido de la agenda deportiva se carga en un iframe separado, aislando el código de la agenda del contexto principal.

---

## 4. Análisis Dinámico — REMnux

### 4.1 Preparación del entorno

- **VM:** REMnux 7 en VirtualBox, modo NAT
- **Snapshot:** tomado antes de iniciar el análisis

> **Incidente durante el análisis:** la VM sufrió un crash al presionar Ctrl+C en el terminal de tcpdump. Se restauró desde el snapshot y se reinstaló Firefox antes de continuar. Esto demuestra la importancia del snapshot pre-análisis.

### 4.2 Captura de red con tcpdump

```bash
# Identificar interfaz (en REMnux no siempre es eth0)
ip a
# En este caso: enp0s3

sudo tcpdump -i enp0s3 -w captura_futbol-libres_$(date +%Y%m%d_%H%M%S).pcap
```

Se generaron tres archivos PCAP:
```
captura_futbol-libres_20260621_180839.pcap
captura_futbol-libres_20260621_181142.pcap
captura_futbol-libres_20260621_181213.pcap  ← principal, 200+ dominios
```

### 4.3 Comportamiento en Firefox

1. La página carga con el listado de partidos en iframe
2. No hay popunders en la carga inicial — aclib.js espera un click del usuario
3. Al hacer el **primer click** en cualquier parte de la página, se disparó el primer popunder

Los navegadores modernos bloquean `window.open()` si no es disparado desde un evento de usuario. El script escucha `click` en todo el documento para garantizar que el primer click del usuario lo dispara.

### 4.4 DevTools — dominios terceros al cargar

```javascript
performance.getEntriesByType('resource')
```

9 dominios terceros contactados en la carga inicial:

| Dominio | Función |
|---|---|
| `cdn.futbol-libres.su` | CDN propio (BunnyNet) |
| `acscdn.com` | SDK de red publicitaria (aclib.js) |
| `usrpubtrk.com` | Tracker silencioso — carga antes de cualquier interacción |
| `ajax.googleapis.com` | jQuery 1.7.1 |
| `www.googletagmanager.com` | Google Tag Manager |
| `www.google-analytics.com` | Google Analytics 4 |
| `fonts.googleapis.com` / `fonts.gstatic.com` | Google Fonts |
| `adexchangerapid.com` | Ad exchange / intermediario RTB |

`usrpubtrk.com` es particularmente notable: carga en silencio en el pageload, sin acción del usuario, y su nombre (`usr pub trk` = user publisher tracker) indica que su función es tracking puro.

### 4.5 Cadena de popunders documentada

| # | Destino | Vertical |
|---|---|---|
| 1 | `jugabet.com` | Casa de apuestas deportivas |
| 2 | `betano.com` | Casa de apuestas deportivas |
| 3 | `etoro.com` | Plataforma de inversión / trading |
| 4 | `aliexpress.com` | E-commerce |

### 4.6 Análisis del PCAP con tshark

```bash
tshark -r captura_futbol-libres_20260621_181213.pcap \
  -Y "dns.flags.response == 1" \
  -T fields -e dns.qry.name \
  | sort -u
```

El PCAP contenía **más de 200 dominios únicos**. Todos verificados en VirusTotal: **ninguno marcado como malicioso** en el momento del análisis. Los popunders apuntaban a anunciantes legítimos, aunque el canal es intrínsecamente inseguro.

---

## 5. Análisis del iframe /agenda/

### 5.1 Estructura y metadata

```bash
curl -sL 'https://futbol-libres.su/agenda/' \
  --user-agent 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36' \
  -o agenda.html
```

```html
<meta http-equiv="cache-control" content="no-cache">
<meta name="robots" content="nofollow, noindex">
```

**Contenido en el momento del análisis (21/06/2026):**

```
Copa Chile: Huachipato vs Puerto Montt — 17:30
Copa Mundial: Bélgica vs Irán — 20:00
Copa Chile: Deportes Temuco vs Deportes Concepción — 20:00
Copa Chile: Universidad Chile vs Santiago Wanderers — 22:30
Copa Mundial: Uruguay vs Cabo Verde — 23:00
Copa Chile: O'Higgins vs Unión Española — 01:00
Copa Mundial: Nueva Zelanda vs Egipto — 02:00
```

El sitio estaba transmitiendo **partidos de la Copa Mundial FIFA 2026 en tiempo real** — incluyendo Bélgica vs Irán, Uruguay vs Cabo Verde, y Nueva Zelanda vs Egipto. Contratos de broadcasting exclusivos valorados en miles de millones de dólares.

**jQuery legacy:**
```html
<script src="https://ajax.googleapis.com/ajax/libs/jquery/1.7.1/jquery.min.js"></script>
```
Versión de 2011 con múltiples CVEs conocidas. Riesgo bajo en este contexto porque el HTML lo genera el propio operador.

### 5.2 Backends de streaming ocultos

Los links de cada partido apuntan a:
```
https://futbol-libres.su/eventos.html?r=<BASE64>
```

El parámetro `r` es una URL completa codificada en base64.

### 5.3 Decodificación base64

```python
import base64
base64.b64decode("aHR0cHM6Ly9sYXRhbXZpZHpzLm9yZy9jYW5hbC5waHA/c3RyZWFtPW1heDE=").decode()
# → https://latamvidzs.org/canal.php?stream=max1
```

**Backend 1: `latamvidzs.org`** — `/canal.php?stream=NOMBRE_CANAL` — Eastern Europe, probable bulletproof hosting.

**Backend 2: `latamvidzfy.org`** — `/CANAL.php` — detrás de **Cloudflare** (AS13335). IP real del servidor completamente oculta.

La separación en dos backends con arquitecturas distintas garantiza redundancia: si uno cae, el otro continúa.

| Base64 | URL decodificada |
|---|---|
| `aHR0cH...bWF4MQ==` | `https://latamvidzs.org/canal.php?stream=max1` |
| `aHR0cH...Y2hpbGU` | `https://latamvidzs.org/canal.php?stream=tntsportschile` |
| `aHR0cH...czIucGhw` | `https://latamvidzfy.org/dsports2.php` |
| `aHR0cH...cy5waHA=` | `https://latamvidzfy.org/dsports.php` |
| `aHR0cH...ZXkyI=` | `https://latamvidzs.org/canal.php?stream=disney2` |
| `aHR0cH...cHIucGhw` | `https://latamvidzfy.org/zdfpr.php` |
| `aHR0cH...bi5waHA=` | `https://latamvidzfy.org/rcn.php` |
| `aHR0cH...NtNi5waHA=` | `https://latamvidzfy.org/m6.php` |
| `aHR0cH...dml4MQ==` | `https://latamvidzs.org/canal.php?stream=vix1` |
| `aHR0cH...ZWNlLnBocA==` | `https://latamvidzfy.org/trece.php` |
| `aHR0cH...b3gxdXMucGhw` | `https://latamvidzfy.org/fox1us.php` |
| `aHR0cH...cmFjb2x0dg==` | `https://latamvidzs.org/canal.php?stream=caracoltv` |
| `aHR0cH...cnN0ZXByLnBocA==` | `https://latamvidzfy.org/daserstepr.php` |
| `aHR0cH...dGVsZW11bmRvdXNh` | `https://latamvidzs.org/canal.php?stream=telemundousa` |
| `aHR0cH...emUyLnBocA==` | `https://latamvidzfy.org/caze2.php` |

La codificación base64 oculta los dominios backend a usuarios casuales, sistemas de filtrado de URLs, y herramientas automáticas de DMCA que rastrean links directos.

### 5.4 Script smallscripts.js

**`popUp()` / `popUpscroll()`** — abre los players en ventanas sin chrome de navegador. Efecto secundario: oculta el dominio real del player al usuario (sin barra de dirección visible).

**`guardaHorario()` / `horaHuso()`** — convierte horarios de partidos a la zona horaria local del visitante usando `new Date().getTimezoneOffset()`. Subproducto: fingerprinting pasivo de zona geográfica del visitante.

**`formatoRegion()`** — detecta si el visitante usa formato 12h o 24h según el timezone offset. Los offsets de Australia, EEUU y Japón reciben AM/PM; el resto 24h. Revela que la audiencia objetivo incluye explícitamente Latinoamérica, EEUU y Australia.

---

## 6. Análisis de aclib.js

Obtenido de: `https://acscdn.com/script/aclib.js`

### 6.1 Obfuscación

El script está ofuscado con una herramienta tipo obfuscator.io. Patrón característico:

```javascript
(function(_0x25fe34, _0x4b7020) {
    const _0x2fbdfd = _0x166c;
    const _0x5cf930 = _0x25fe34();
    while (!![]) {
        try {
            const _0x1efc7d = parseInt(...) / 0x1 * ...
            if (_0x1efc7d === _0x4b7020) break;
            else _0x5cf930['push'](_0x5cf930['shift']());
        } catch (_0x41a641) {
            _0x5cf930['push'](_0x5cf930['shift']());
        }
    }
}(_0x336f, 0x99213));
```

Esta IIFE inicial es el deofuscador de strings: toma un array de strings encriptadas y las reordena mediante un proceso matemático. El resultado es que los strings literales no aparecen directamente en el source — evadiendo adblockers basados en patrones de texto.

### 6.2 Fingerprinting del visitante

La función `_0x2f757e` construye un string de fingerprint concatenando:

```
navigator.language + navigator.platform + navigator.vendor
+ navigator.cookieEnabled + navigator.languages.join()
+ screen.width + "x" + screen.height
+ new Date().getTimezoneOffset()
+ navigator.doNotTrack + navigator.hardwareConcurrency
+ screen.colorDepth
```

Más agresiva aún es la función que usa la **Client Hints API moderna**:
```javascript
navigator.userAgentData.getHighEntropyValues([
    'platform', 'bitness', 'platformVersion', 'fullVersionList'
])
```

Disponible solo en Chromium, permite extraer la **versión exacta del OS y del navegador** — información que el User-Agent normal deliberadamente oculta por privacidad.

### 6.3 Mecanismo de popunder

1. Registra listeners en `document` para `visibilitychange` y `click`
2. Al primer click, el handler hace `fetch()` al ad server para obtener la URL destino
3. Si la respuesta es válida, ejecuta `window.open(url, '_blank')`
4. Registra la impresión vía un segundo fetch

El parámetro `delay` configurable permite que el operador establezca un tiempo mínimo en página antes de que el click dispare el popunder — filtrando rebotes.

### 6.4 Shadow DOM anti-adblocker

Los overlay ads se inyectan dentro de un **Shadow DOM**:
```javascript
const shadowRoot = container.attachShadow({mode: 'open'});
shadowRoot.appendChild(adElement);
```

El Shadow DOM es un estándar web de encapsulación. Los selectores CSS del DOM principal no pueden acceder al contenido dentro de un shadow — los filtros de EasyList y uBlock Origin basados en CSS son completamente inefectivos contra esto. Es una de las técnicas más sofisticadas actualmente en uso por redes de ads para evadir bloqueadores.

### 6.5 URL parameter shuffling

Antes de enviar requests al ad server:

1. Toma todos los query parameters de la URL
2. Los convierte a un array de pares `[key, value]`
3. Aplica Fisher-Yates shuffle (barajado aleatorio)
4. Codifica el resultado en base64
5. Genera un nombre de parámetro aleatorio de 24 caracteres
6. Añade: `URL?<random24chars>=<base64(shuffled_params)>`

Resultado: cada request al ad server tiene una URL diferente, impidiendo detección por firmas de URL en IDS/IPS o filtros de proxy.

### 6.6 Sistema de cooldown y retry

**Cooldown:** después de un popunder exitoso, el flag `inCooldown` se activa por un tiempo configurable. El usuario no recibe otro popunder hasta que expire.

**Retry con backoff exponencial:** delay inicial de 12 segundos, multiplicado por 5 en cada fallo:
- 1er fallo: 12s
- 2do fallo: 60s
- 3er fallo: 300s
- 4to fallo: 1500s
- Máximo: 7200s (2h)

---

## 7. Mapa de infraestructura completo

```
futbol-libres.su (TLD: .su / TCINET, Russia)
│
├── Nameservers (propios del operador)
│   ├── a.p-dns.com
│   ├── b.p-dns.biz
│   ├── c.p-dns.biz
│   └── d.p-dns.info
│
├── IPs del dominio principal (rotación, TTL 60s)
│   ├── 45.11.57.200    ┐
│   ├── 185.254.197.23  │ Virtual Systems LLC
│   ├── 128.0.104.19    │ (vsys.host, Ucrania)
│   ├── 138.226.244.112 │ Bulletproof hosting
│   ├── 194.42.205.89   │ AS49530
│   └── 212.86.116.38   ┘
│
├── cdn.futbol-libres.su
│   └── CNAME → flsw.b-cdn.net → 185.93.1.246 (BunnyNet CDN, EU)
│
├── Scripts de terceros
│   ├── acscdn.com/script/aclib.js       ← SDK popunders
│   ├── usrpubtrk.com                    ← Tracker silencioso
│   ├── adexchangerapid.com              ← Ad exchange RTB
│   ├── ajax.googleapis.com              ← jQuery 1.7.1
│   ├── www.googletagmanager.com         ← GTM
│   ├── www.google-analytics.com         ← GA4 (G-L0N11PVD63)
│   ├── fonts.googleapis.com             ← Google Fonts
│   └── fonts.gstatic.com
│
└── iframe /agenda/
    ├── Backends de streaming (via Base64 en param ?r=)
    │   ├── latamvidzs.org               ← /canal.php?stream=CANAL
    │   │   IPs: 31.42.184.180, 128.0.104.49, 62.182.85.23, 128.0.104.23
    │   └── latamvidzfy.org              ← /CANAL.php
    │       IPs: 104.21.9.115, 172.67.159.198
    │       (Cloudflare AS13335 — IP real oculta)
    └── cdn.futbol-libres.su/agenda/smallscripts.js (BunnyNet CDN)
```

---

## 8. OPSEC del operador

### Medidas efectivas

- **TLD .su** — jurisdicción rusa, immune a DMCA occidental. Renovado hasta 2027.
- **Bulletproof hosting (vsys.host)** — ignoran explícitamente abuse reports.
- **TTL 60s** — rotación de IPs en 1 minuto.
- **4 nameservers en 3 TLDs** — takedown requeriría coordinar tres registradores distintos.
- **ProtonMail como email operativo** — `joezm5a@proton.me`, cifrado E2E.
- **Base64 para ocultar backends** — reduce exposición automática a scrapers de DMCA.
- **noindex/nofollow** — no aparece en búsquedas de Google.
- **Cloudflare para `latamvidzfy.org`** — IP real completamente oculta.
- **BunnyNet CDN** — scripts servidos desde CDN legítima.

### Errores de OPSEC

- **Gmail expuesto en WHOIS:** `martinjoe19411@gmail.com`. Google tiene datos de esta cuenta.
- **Google Search Console verificado:** el token TXT vincula el dominio a una cuenta de Google.
- **Google Analytics activo (G-L0N11PVD63):** Google tiene todos los datos de tráfico. Una solicitud legal podría revelar el propietario.
- **SOA con email de ProtonMail expuesto:** `joezm5a` es un identificador con posibles correlaciones en otras plataformas.
- **zoneId de la red de ads expuesto:** `9195464` está vinculado a una cuenta en la red de popunders, que tiene datos KYC del publisher para poder pagarle.

---

## 9. Cadena RTB — Real Time Bidding

```
1. Usuario carga futbol-libres.su
   └─ usrpubtrk.com registra la visita

2. Usuario hace click
   └─ aclib.js intercepta el evento

3. aclib.js envía bid request a acscdn.com con:
   - zoneId del publisher (9195464)
   - Fingerprint del usuario
   - URL actual

4. acscdn.com convoca subasta con adexchangerapid.com (~100ms)

5. El ad server responde con la URL del ganador

6. aclib.js ejecuta window.open(url_ganador)

7. Se abre: jugabet.com / betano.com / etoro.com / aliexpress.com
```

**Implicación de seguridad crítica:** el operador de `futbol-libres.su` no controla qué URL abre el popunder. Si un actor malicioso gana la subasta RTB, puede apuntar el popunder a páginas de phishing, exploits, o distribución de malware — sin que el operador del sitio lo sepa.

---

## 10. Vectores de ataque potenciales

### Contra el visitante

- **Malvertising via RTB** — el vector más probable. Si la red acepta anunciantes maliciosos, los popunders pueden apuntar a exploits o phishing.
- **Clickjacking** — ausencia de `X-Frame-Options` permite embeber la página en un iframe de un sitio malicioso.
- **Fingerprinting extensivo** — aclib.js recopila resolución, timezone, hardware concurrency, y versión exacta de OS/browser via Client Hints.
- **Exposición a Google** — usuarios logueados en Google son correlacionados con la visita vía GA y GTM.

### Contra el operador

- **Correlación de identidades** — gmail + ProtonMail SOA + GA token + zoneId proporcionan múltiples vectores de atribución para una investigación legal coordinada.
- **Suspensión por la red de ads** — las redes publicitarias prohíben monetizar contenido pirata. Si la detectan, suspenden la cuenta y retienen los pagos.

---

## 11. Herramientas utilizadas

| Herramienta | Propósito | Fase |
|---|---|---|
| `jwhois` | Consultas WHOIS | Pasivo |
| `dig` (dnsutils) | Resolución DNS, NS, MX, TXT | Pasivo |
| `curl` | Headers HTTP, descarga de HTML/JS | Pasivo |
| Google DNS over HTTPS | DNS queries sin exponer IP | Pasivo |
| `crt.sh` | Certificate Transparency logs | Pasivo |
| `api.bgpview.io` / `rdap.arin.net` | Lookup de ASN por IP | Pasivo |
| REMnux 7 (VirtualBox, NAT) | Sandbox de análisis dinámico | Dinámico |
| `tcpdump` | Captura de tráfico de red | Dinámico |
| `tshark` | Análisis de PCAP | Dinámico |
| Firefox + DevTools | Navegador y análisis de requests | Dinámico |
| `performance.getEntriesByType` | Enumerar dominios terceros | Dinámico |
| VirusTotal | Reputación de dominios del PCAP | Threat Intel |
| Python 3 (`base64`) | Decodificación de URLs del iframe | Post-análisis |
| Análisis manual de JS ofuscado | Ingeniería inversa de aclib.js | Post-análisis |

---

## 12. Conclusiones

`futbol-libres.su` es un sitio de piratería de streaming deportivo sofisticado, con una arquitectura deliberadamente diseñada para maximizar la resiliencia (bulletproof hosting, TTL corto, NS redundantes, base64 obfuscation) y minimizar la superficie de atribución legal.

En el momento del análisis, el sitio **no distribuye malware directamente**. El riesgo real para el visitante es el fingerprinting extensivo y la exposición potencial a malvertising si la red de ads acepta anunciantes maliciosos — lo que el canal RTB hace posible en cualquier momento sin conocimiento del operador.

No obstante, el operador cometió errores de OPSEC significativos al usar Gmail, Google Analytics, Google Search Console y una red de ads legítima: todos servicios que mantienen registros y están sujetos a órdenes legales. La combinación de esos identificadores proporciona suficientes vectores de atribución para una investigación coordinada.

Esta investigación demuestra que incluso un sitio aparentemente simple implementa múltiples capas de técnicas ofensivas y defensivas: desde la selección de TLD hasta el Shadow DOM, pasando por RTB, fingerprinting, hosting bulletproof, y obfuscación de JavaScript. El análisis completo requirió OSINT, análisis de red, ingeniería inversa de JS, y conocimiento de infraestructura DNS.

---

*Documento generado el 21/06/2026. Todos los datos reflejan el estado del objetivo en esa fecha.*
