# Auditoría de readiness para producción — Padle

- **Repo:** `/home/clio/dev/Padle` (deploy: `https://mdqclio.github.io/Padle/`)
- **Fecha:** 2026-06-06
- **Stack:** PWA de archivo único — HTML + CSS + JS vanilla (sin build step) · Firebase Realtime Database (compat SDK 10.12.5 vía CDN gstatic) · Service Worker propio (`sw.js`) · `manifest.webmanifest` · Hosting GitHub Pages.
- **Qué es:** app para que un grupo de amigos trackee turnos de pádel (reservados vs. jugaron → créditos/débitos) con sync en tiempo real y un audit log a nombre de cada device.

**Leyenda:** 🟢 ok · 🟡 mejorable · 🔴 bloqueante

---

## 1) Frontend comprimido / sin secretos de cliente — 🟢

El frontend **no expone secretos de terceros**. Lo único sensible visible es `firebaseConfig` en `index.html` (líneas 501-509), que incluye `apiKey: "AIzaSy...KDqc"`. Esa apiKey **NO es un secreto**: la API key web de Firebase es un identificador público de proyecto, está diseñada para vivir en el cliente y no otorga permisos por sí sola (los permisos los dan las reglas de RTDB, ver punto 2). No hay tokens privados, service-account keys ni claves de terceros en el bundle.

- **Riesgo:** Bajo en cuanto a secretos. Salvedad menor: el frontend **no está minificado/comprimido** (index.html ~41 KB sin minificar) y depende de GitHub Pages para servir gzip/brotli (lo hace por defecto). No es bloqueante para esta escala.

## 2) RLS / Reglas de Firebase (`firebase-rules.json`) — 🔴

**Las reglas NO son seguras: dan acceso abierto total.** El archivo `firebase-rules.json` declara:

```json
"jugadores": { ".read": true, ".write": true },
"turnos":    { ".read": true, ".write": true },
"devices":   { ".read": true, ".write": true },
"audit":     { ".read": true, "$eventId": { ".write": "!data.exists()" } }
```

- `jugadores`, `turnos` y `devices` son **lectura y escritura públicas para cualquiera en internet** (`.read:true / .write:true`), sin autenticación. Cualquier persona que conozca la `databaseURL` (visible en el cliente) puede leer, modificar o **borrar toda la base** con un simple `PUT`/`DELETE` HTTP, sin abrir la app.
- **No existe aislamiento por usuario (RLS).** No hay `auth != null`, no hay `auth.uid`, no hay scoping por device. La identidad es un `deviceId` en `localStorage` (auto-generado, falsificable) y un nombre de texto libre — puro honor system, sin valor de seguridad.
- **No hay validación de esquema** (`.validate`): nada impide escribir tipos arbitrarios, payloads gigantes o estructuras inesperadas que rompan el render del cliente.
- El único acierto: `/audit/$eventId` es **append-only** (`".write": "!data.exists()"`), no se puede modificar ni borrar un evento existente. Pero como `audit` es de **lectura pública**, expone todo el historial; y al no estar protegida la escritura por auth, cualquiera puede inyectar eventos falsos (solo no puede sobrescribir los existentes).

Esto es el equivalente a la base en **modo prueba/abierto**, exactamente como sugiere el README ("modo de prueba está bien al principio"). Para producción es un **bloqueante de datos**: cualquiera puede vaciar o vandalizar la DB.

- **Riesgo:** Alto/crítico. Pérdida total de datos, manipulación de balances, inyección de audit falso, scraping del historial. Sin auth real no hay forma de RLS.

## 3) Git sin secretos — 🟢

Revisado `git log -p --all`: el único match es la `apiKey` pública de Firebase (commit `e8475e1 config: pegar firebaseConfig real`), que como se explicó no es un secreto. **No hay** service-account JSON, `.env`, private keys ni client_secret en el historial. El `.gitignore` cubre `.env`, `.env.local`, `dist/`, `node_modules/`, `.claude/`.

- **Riesgo:** Bajo. La apiKey pública no requiere rotación; si se quisiera, se restringe por dominio/HTTP referrer en Google Cloud Console, no rotándola.

## 4) APIs: auth / permisos / validación — 🔴

No hay backend ni API propia: el cliente habla **directo** contra RTDB. Por lo tanto auth/authz/validación recaen 100% en las reglas, que están abiertas (punto 2).

- **Sin autenticación:** no hay Firebase Auth (anónima ni federada). Toda operación es anónima.
- **Sin autorización:** cualquiera escribe en cualquier nodo.
- **Sin validación server-side:** la validación de entrada vive solo en el cliente (ej. `maxlength="40"`, `.trim()` en `saveName`), trivial de saltear pegándole directo a la REST API de RTDB.

- **Riesgo:** Alto. Cualquier cliente HTTP puede corromper datos saltando toda la lógica del frontend.

## 5) Hosting / entornos / env vars — 🟡

- **Hosting:** GitHub Pages (estático, HTTPS por defecto, CDN). Adecuado y barato para esta escala.
- **Entornos:** **no hay separación dev/staging/prod**. Un solo proyecto Firebase (`padle-82f32`), una sola DB, un solo deploy. Cualquier prueba impacta la base "de producción".
- **Env vars:** al ser sitio estático sin build, la config está hardcodeada en `index.html`. No hay mecanismo de inyección de variables por entorno; aceptable dado que la apiKey es pública, pero impide tener configs separadas por entorno sin editar el HTML.
- **Headers de seguridad:** GitHub Pages **no permite** custom headers → no hay CSP, HSTS, X-Frame-Options, etc. Sin CSP la superficie XSS es mayor (ver punto 6).

- **Riesgo:** Medio. Falta de aislamiento de entornos y ausencia de CSP/headers de seguridad.

## 6) Login / sesiones / vulnerabilidades — 🟡

- **No hay login ni sesiones.** La "identidad" es `deviceId` (uuid en localStorage) + nombre libre. No es autenticación; es etiquetado cosmético para el audit log. No hay nada que robar (ni cookies ni tokens de sesión), pero tampoco hay nada que impida suplantar a otro escribiendo su nombre.
- **XSS / `innerHTML`:** el código usa `innerHTML` extensivamente para renderizar datos provenientes de la DB (turnos, jugadores, log). **El mitigante es bueno:** existe `escapeHtml()` (línea 548) y se aplica de forma consistente sobre todos los datos del usuario interpolados (`escapeHtml(j.nombre)`, `escapeHtml(r.nombre)`, `escapeHtml(e.device_name)`, `escapeHtml(describeAction(e))`, etc.). No detecté interpolación de strings de usuario sin escapar en los templates. **Pero:** dado que la DB es escribible por cualquiera (punto 2), un atacante podría intentar inyectar payloads; el escaping actual los neutraliza, aunque sin CSP (punto 5) no hay segunda línea de defensa. Un descuido futuro en un template sería XSS persistente directo.

- **Riesgo:** Medio. Sin auth real no hay no-repudio; el XSS está mitigado hoy pero es frágil sin CSP y con DB abierta.

## 7) Rate limiting — 🔴

**No existe rate limiting de ninguna clase.** No hay backend que lo aplique, las reglas de RTDB no limitan frecuencia/tamaño, y la DB es escribible anónimamente. Un actor puede:

- Hacer un flood de escrituras (inflar `audit`, crear turnos infinitos).
- Disparar consumo de cuota/facturación de Firebase (RTDB cobra por GB descargados y almacenados).
- Dado que `audit` se lee con `limitToLast(300)` pero los datos crecen sin tope, un flood degrada y encarece.

- **Riesgo:** Alto. Abuso de cuota / costos / vandalismo trivial sin auth ni throttling.

## 8) Caché — Service Worker (`sw.js`) — 🟢

La estrategia de caché está **bien diseñada** para una PWA de este tipo:

- **HTML → network-first** (líneas 41-51): siempre intenta red primero y cae al caché solo offline → los usuarios reciben actualizaciones de `index.html` sin quedar pegados a una versión vieja (problema clásico de PWAs cache-first).
- **Assets estáticos (icon, manifest) → cache-first** (líneas 52-64): correcto, son inmutables.
- **Requests externos (Firebase, fonts, gstatic) se excluyen** explícitamente (`if (url.origin !== self.location.origin) return;`, línea 34): evita cachear datos en tiempo real o el SDK, correcto.
- **Activación limpia caches viejos** (líneas 19-25) y usa `skipWaiting` + `clients.claim`.

Salvedad menor: el `CACHE = 'padel-v1'` es estático y los assets cacheados (`./`, `index.html`) por network-first se actualizan solos, pero el `icon.svg`/`manifest` cache-first solo se refrescan bumpeando el `CACHE` (el propio comentario lo documenta). Aceptable.

- **Riesgo:** Bajo. Estrategia sólida; solo recordar bumpear `CACHE` al cambiar assets estáticos.

## 9) Escalabilidad — 🟡

- Para el caso de uso real (un grupo de amigos, decenas de registros) escala de sobra: RTDB + GitHub Pages aguantan esto sin esfuerzo.
- **Límites estructurales:** los listeners usan `.on('value')` sobre nodos **completos** de `jugadores` y `turnos` (líneas 651, 665): cada cambio retransmite todo el nodo a todos los clientes. Con cientos/miles de turnos esto crece linealmente en ancho de banda y memoria del cliente. `computeBalances()` y los renders recorren todo en cada evento (O(turnos×jugadores)). No hay paginación salvo `limitToLast(300)` en audit.
- Sin auth, también es vulnerable a inflado malicioso (punto 7) que rompería estos supuestos de escala.

- **Riesgo:** Medio-bajo a la escala prevista; degradaría si los datos crecieran mucho o ante abuso.

## 10) Monitoreo / alertas — 🔴

**No hay observabilidad alguna.** No hay error tracking (Sentry/similar), ni analytics, ni logging server-side, ni alertas de presupuesto/cuota de Firebase. Los errores se tragan silenciosamente: `recordChange` hace `console.error` (línea 639), el registro del SW usa `.catch(() => {})` (línea 1181) descartando fallos, y varios `.catch(() => {})` en el SW. Si la DB se vacía, se llena de basura o se dispara la factura, **nadie se entera** hasta abrir la app.

- **Riesgo:** Alto. Cero visibilidad ante incidentes, abuso o picos de costo. Sin alertas de billing, el abuso del punto 7 puede generar costos silenciosos.

---

## Tabla resumen

| # | Punto | Estado | Síntesis |
|---|-------|--------|----------|
| 1 | Frontend comprimido / sin secretos cliente | 🟢 | Solo apiKey pública (no es secreto); sin minificar pero OK |
| 2 | RLS / reglas Firebase | 🔴 | `.read:true/.write:true` → DB abierta, sin auth, sin aislamiento por usuario |
| 3 | Git sin secretos | 🟢 | Solo apiKey pública en historial; sin keys privadas |
| 4 | APIs: auth / permisos / validación | 🔴 | Cliente directo a RTDB; toda la seguridad depende de reglas abiertas |
| 5 | Hosting / entornos / env vars | 🟡 | GitHub Pages OK, pero sin dev/staging/prod ni CSP/headers |
| 6 | Login / sesiones / vulns | 🟡 | Sin auth real; XSS mitigado con `escapeHtml` pero frágil sin CSP |
| 7 | Rate limiting | 🔴 | Inexistente; flood y abuso de cuota triviales |
| 8 | Caché (Service Worker) | 🟢 | Network-first HTML + cache-first assets + excluye externos: correcto |
| 9 | Escalabilidad | 🟡 | OK a la escala real; `.on('value')` sobre nodos completos no escala a volumen |
| 10 | Monitoreo / alertas | 🔴 | Cero observabilidad, sin alertas de billing/errores |

---

## Los 3 arreglos más urgentes

1. **Cerrar las reglas de Firebase + agregar autenticación (punto 2 y 4).** Hoy la base es pública R/W: cualquiera puede borrarla. Mínimo viable: activar Firebase Auth (anónima o, mejor, federada/Google) y cambiar las reglas a `auth != null` para lectura/escritura, agregando `.validate` por esquema. Sin esto, no hay producción posible.

2. **Rate limiting + alerta de presupuesto en Firebase (puntos 7 y 10).** Configurar un **budget alert** en Google Cloud / Firebase para detectar abuso de cuota, y endurecer reglas (límites de tamaño con `.validate`, append-only ya presente en audit). Atado al fix #1 (auth corta el grueso del abuso anónimo).

3. **Monitoreo de errores + CSP (puntos 10 y 6/5).** Integrar un error tracker (ej. Sentry) para no perder fallos silenciosos, y servir una Content-Security-Policy (vía `<meta http-equiv>` ya que GitHub Pages no permite headers) como segunda línea de defensa XSS sobre el `escapeHtml` existente.
