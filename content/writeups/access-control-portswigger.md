---
title: "Access Control — Tres labs, un concepto"
date: 2026-06-10
draft: false
summary: "Vertical, horizontal, context-dependent. Tres labs de PortSwigger que son el mismo error con distinta forma: la app asume que no vas a encontrar algo en vez de verificar que tienes permiso para verlo."
tags: ["web-security", "access-control", "portswigger", "bug-bounty", "aprendizaje"]
categories: ["Seguridad"]
toc: false
---

*PortSwigger Web Security Academy | Junio 2026*

---

Los writeups de laboratorios los escribo en segunda persona. Es la forma en que naturalmente me hablo a mí mismo para fijar lo que acabo de hacer — no un tutorial, sino una nota hacia adentro. Si estás leyendo esto, bienvenido a mi proceso.

---

Los tres tipos que existen: **vertical** (usuario normal → admin), **horizontal** (tus datos → datos de otro), **context-dependent** (hacer algo fuera del orden permitido). Estos tres labs son verticales. Recuerda eso porque te va a servir para clasificar lo que encuentres.

---

## Lab 1 — robots.txt

Encontraste el panel admin leyendo `/robots.txt`. El dev lo puso ahí para que Google no lo indexara y sin querer expuso la ruta.

Lo importante no es el paso a paso, es el concepto: **esconder una URL no es protegerla**. Eso tiene nombre — *security through obscurity* — y es un error de diseño, no una feature de seguridad.

En cualquier target real, `/robots.txt` es lo primero que miras. Siempre.

---

## Lab 2 — JavaScript en el código fuente

La URL del admin tenía un sufijo random y parecía impredecible. Pero estaba hardcodeada en el JS que la app le manda a todos los usuarios.

```javascript
var isAdmin = false;
if (isAdmin) {
    // /admin-8nijd8  ← ahí estaba
}
```

El bloque no se ejecuta, pero el código se descarga igual. Eso fue suficiente.

Lo que hiciste fue Ctrl+U, Ctrl+F "admin", y apareció. Dos teclas.

**Checklist para cualquier target:**

```
Ctrl+U → Ctrl+F: admin, api, internal, token, key, dashboard, href
```

Es trivial y la gente no lo hace. Tenemos una posible ventana.

---

## Lab 3 — Cookie `Admin: false`

Este lab fue más enredado de lo esperado por dos razones que vale la pena recordar.

**Kaspersky bloqueó el lab.** El antivirus intercepta tráfico HTTPS y choca con el certificado de Burp. La solución es desactivar el escaneo de tráfico cifrado en Kaspersky mientras testeas. Sin eso, los labs no cargan.

**Burp no capturaba nada.** Brave como browser por defecto, pero Burp solo intercepta tráfico que pasa por él. Si el browser no está configurado para usar Burp como proxy, HTTP History queda vacío. La solución fue usar el browser interno de Burp (Proxy → Open Browser). Ojo con esto en targets reales — si algo no aparece en HTTP History, primero verifica que el tráfico esté pasando por Burp.

**Sin Burp, directo en DevTools → Application → Cookies:**

| Cookie | Valor |
|--------|-------|
| `Admin` | `false` |
| `session` | `8dRyCp5NI...` |

`false` por `true`. El servidor confió y dio acceso de admin.

El error del dev fue dejarle al cliente la decisión de quién es admin. **Los roles siempre se validan en el servidor, nunca en el browser.** Cuando veas una cookie con un booleano o un valor tipo `role=user`, `isAdmin=0`, `type=regular` — está claro.

---

## Lo que une los tres

Los tres labs son el mismo error con distinta forma:

> La app *asume* que no vas a encontrar o modificar algo, en vez de *verificar* que tienes permiso.

Esa distinción es fundamental. Cuando estés en un target real y veas algo "escondido" — una ruta, un parámetro, una cookie — pregúntate si está *escondido* o si está *protegido*. Son cosas distintas.

---

*Próximo: IDOR — mismo concepto pero horizontal.*
