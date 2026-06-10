---
title: "chilen.sys — Motor de Auditoría de Ciberseguridad Pasiva"
date: 2026-06-10
draft: false
description: "Herramienta de reconocimiento pasivo B2B para auditoría de ciberseguridad empresarial. Analiza más de 40 vectores sin tocar los sistemas del objetivo, genera dos informes PDF profesionales y cuantifica el riesgo con un modelo propio alineado a CVSS."
summary: "Motor de reconocimiento pasivo con 11 módulos OSINT, modelo de riesgo propio y dos tipos de informe PDF. Construido en Python asyncio + ReportLab. El problema que intenté resolver: cómo presentar reconocimiento pasivo de forma que tenga sentido para quien no es técnico."
tags: ["Python", "Ciberseguridad", "OSINT", "asyncio", "ReportLab", "B2B", "SaaS"]
categories: ["Proyecto", "Ciberseguridad"]
toc: true
weight: 1
---

## El problema

Un atacante puede pasar horas recopilando datos sobre una empresa sin tocar ningún sistema — consultando registros DNS, bases de datos de certificados, logs de brechas conocidas, Shodan. Todo eso es público. La mayoría de las empresas medianas no sabe qué está exponiendo.

Quería construir algo que hiciera ese reconocimiento pasivo de forma sistemática y lo tradujera a algo accionable. El reto técnico no era el OSINT en sí — hay herramientas para eso. Era la capa de síntesis: convertir 40 hallazgos dispersos en algo que tenga sentido para alguien que no sabe qué es un registro DMARC.

---

## Qué construí

### Motor de reconocimiento pasivo — 11 módulos OSINT

El núcleo es un motor asíncrono en Python que orquesta 11 módulos especializados, todos en modo pasivo — sin tocar el objetivo:

| Módulo | Qué hace |
|---|---|
| `ct_subdomains` | Enumera subdominios desde Certificate Transparency logs (crt.sh) |
| `dns_recon` | Analiza SPF, DKIM, DMARC, DNSSEC, CAA, MTA-STS, TLS-RPT y BIMI |
| `shodan_recon` | Consulta Shodan InternetDB para puertos abiertos y CVEs (sin API key) |
| `takeover` | Detecta subdominios con CNAMEs a servicios abandonados (GitHub Pages, S3, Heroku, Azure…) |
| `typosquat` | Genera y resuelve dominios lookalike para detectar typosquatting activo con correo |
| `breaches` | Consulta HaveIBeenPwned por brechas del dominio objetivo |
| `fingerprint` | Identifica el stack tecnológico desde headers, cookies y HTML |
| `sourcemaps` | Detecta source maps de JavaScript expuestos que permiten reconstruir el código fuente |
| `html_intel` | Extrae IPs privadas, rutas internas y comentarios de desarrollo del HTML público |
| `privacy` | Detecta trackers de terceros cargados antes del consentimiento (GDPR) |
| `cyber` | Cabeceras de seguridad HTTP, configuración TLS, archivos sensibles expuestos |

### Sistema de intrusividad — Pasivo / Estándar / Agresivo

Uno de los problemas de diseño que tuve que resolver fue el legal: ¿hasta dónde puede llegar la herramienta sin requerir autorización explícita del objetivo? La respuesta fue clasificar cada módulo por nivel de intrusividad (`PASSIVE`, `LIGHT`, `ACTIVE`, `INTENSIVE`) y filtrar en tiempo de ejecución según el modo seleccionado:

```bash
python main.py https://empresa.cl --mode passive    # solo OSINT externo
python main.py https://empresa.cl --mode standard   # + análisis de respuesta HTTP
python main.py https://empresa.cl --mode aggressive # + pruebas activas (requiere autorización)
```

En modo pasivo no se envía ninguna petición directa al objetivo — todo viene de fuentes públicas de terceros. Eso tiene implicaciones bajo la Ley 19.223 / 21.459 de Chile y me importaba dejarlo claro en el diseño, no como nota al pie.

### Motor de riesgo propio

Esta fue la parte más difícil de calibrar. El primer intento era un score numérico simple que resultaba o alarmista o irrelevante dependiendo del objetivo. El problema es que un hallazgo aislado rara vez importa — lo que importa es cómo se encadenan.

`risk_engine.py` agrega los hallazgos de los 11 módulos en un modelo unificado:

- Score 0–100 alineado a CVSS (`critical×25 + high×12 + medium×5 + low×1 + attack_paths×8`)
- Encadenamiento de hallazgos en **rutas de ataque** narrativas: "credenciales filtradas + DMARC ausente = campaña de phishing creíble contra empleados"
- Clasificación por categorías: superficie de ataque, email/dominio, código, privacidad, configuración

El multiplicador de `attack_paths` fue la adición que más cambió la utilidad del score: dos hallazgos que juntos abren un vector real pesan más que cuatro hallazgos aislados de bajo impacto.

### Dos informes PDF

El otro problema de diseño: a quién le entregás el informe. En una empresa mediana, el que recibe el hallazgo técnico (IT) no tiene presupuesto, y el que tiene presupuesto (el dueño) no entiende qué es un CVE. Resolverlo con un solo informe no funciona.

#### Informe Técnico (`pdf_reporter.py`)
11 secciones numeradas + apéndice ordenado por CVSS. Diseño dark con portada de score de riesgo, tablas de evidencia y remediaciones concretas. Para el equipo técnico.

#### Informe Ejecutivo (`pdf_executive.py`)
5 páginas. Veredicto en una frase en la portada, hallazgos narrados como consecuencias reales para el negocio, plan de acción con plazos (esta semana / este mes / este trimestre). Sin jerga técnica. Para el dueño o directivo.

Ambos son white-label: logo, nombre de marca, colores y contacto configurables.

---

## Arquitectura

```
web_auditor/
├── auditor/
│   ├── core.py            # Orquestador principal — asyncio + semáforos
│   ├── intrusiveness.py   # Clasificación Probe/Mode
│   ├── models.py          # AuditResult con los 11 campos de módulos
│   ├── risk_engine.py     # Modelo de riesgo y rutas de ataque
│   ├── pdf_reporter.py    # PDF técnico (ReportLab)
│   ├── pdf_executive.py   # PDF ejecutivo (ReportLab)
│   ├── ct_subdomains.py
│   ├── dns_recon.py
│   ├── shodan_recon.py
│   ├── takeover.py
│   ├── typosquat.py
│   ├── breaches.py
│   ├── fingerprint.py
│   ├── sourcemaps.py
│   ├── html_intel.py
│   ├── privacy.py
│   └── cyber.py
├── main.py
└── landing/
```

**Stack:** Python 3.11+ / asyncio · httpx (HTTP/2) · dnspython · sslyze · beautifulsoup4 + lxml · ReportLab · playwright

---

## Lo que aprendí

Diseñar cada módulo como completamente autónomo —dataclass propia, sin dependencias cruzadas con otros módulos— fue una decisión que pagó dividendos: agregar un módulo nuevo no rompe nada, y testear uno en aislado es trivial. Lo haría así desde el primer día en cualquier proyecto similar.

El modelo de riesgo tardó tres iteraciones en quedar útil. La versión 1 era un score lineal que no distinguía entre un certificado expirado y credenciales filtradas del CEO. La versión 2 introducía pesos por severidad pero seguía sin capturar la relación entre hallazgos. La versión 3, con rutas de ataque encadenadas, fue la primera que generó informes donde los hallazgos contaban una historia en vez de ser una lista.

---

## Estado actual

- Motor v3.0.0 con los 11 módulos funcionando
- Ambos PDFs generados y verificados
- Landing page publicada en Netlify con formulario activo
- Primer cliente prospecto en conversación

**Próximo paso:** automatizar el flujo completo — formulario → análisis → entrega de PDFs sin intervención manual.
