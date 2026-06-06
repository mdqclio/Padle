# Remediación — Padle

- **Repo:** `/home/clio/dev/Padle` (deploy: `https://mdqclio.github.io/Padle/`)
- **Fecha:** 2026-06-06
- **Base:** `docs/auditoria/Padle.md`
- **Alcance de esta tanda:** SOLO arreglos seguros y reversibles en código. **No** se desplegó, **no** se reescribió historial, **no** hubo force-push.

**Leyenda:** ✅ hecho en código · 📄 documentado · ⏳ pendiente del dueño

---

## Contexto crítico (leer antes que nada)

La app **no usa Firebase Auth**. La "identidad" es un `deviceId` (uuid en `localStorage`, falsificable) + un nombre de texto libre. El cliente habla **directo** contra RTDB.

Por eso **no se puede** simplemente poner `auth != null` en las reglas: rompería toda la app (ninguna request está autenticada). Lo que sí se hizo es **endurecer la forma de los datos** sin cambiar el modelo de acceso. Esto **reduce el daño** (no se puede escribir basura, tipos arbitrarios, payloads gigantes ni campos no esperados; `/audit` sigue append-only) pero **NO cierra el agujero**: un actor anónimo todavía puede leer todo, sobrescribir nodos con datos *bien formados* y borrar. **El cierre real es cambio de arquitectura** (Auth + reglas por uid) → ver sección ⏳.

---

## ✅ Hecho en código (esta tanda)

### ✅ 1. `firebase-rules.json` endurecido con `.validate` (mitiga puntos 2 y 4)

Se mantuvo `.read:true / .write:true` en `jugadores`, `turnos`, `devices` (obligatorio sin Auth) y se agregó:

- **`.read:false / .write:false` por defecto** en la raíz: lo no declarado explícitamente queda denegado (antes, cualquier nodo nuevo heredaba acceso abierto de facto). Ahora solo los 4 nodos conocidos son accesibles.
- **`.validate` de esquema y tipos** en cada nodo, derivado de las escrituras reales del cliente (`index.html`):
  - `jugadores/$jid` → exige `{ id (string, == $jid, ≤64), nombre (string, 1–80) }`; **rechaza cualquier otro campo** (`"$other": {".validate": false}`).
  - `turnos/$tid` → exige `id` (== `$tid`) y `fecha` (string ≤32); valida `createdAt` numérico y `reservados`/`jugaron` como listas de strings ≤64; rechaza campos extra.
  - `devices/$did` → exige `name` (string 1–80); valida `firstSeen` numérico; rechaza extra.
  - `audit/$eventId` → **append-only conservado** (`".write": "!data.exists()"`) + `.validate` que exige `ts` numérico, `action/entity/device` strings acotados, etc.; rechaza campos extra fuera del set esperado (`before`/`after` se dejan libres por ser snapshots arbitrarios).
- **Comentario honesto** (clave `//` dentro de `rules`, ignorada por Firebase) que documenta que esto NO da aislamiento por usuario y que el cierre real requiere Auth.

**Límite de longitud de strings** = tope implícito al tamaño de payload por campo, mitigando parte del flood del punto 7 (no lo elimina).

**Verificación:** `node -e "require('./firebase-rules.json')"` → parsea OK, único top-level key `rules`. (El `//` es comentario válido de reglas RTDB.)

**⚠️ Estas reglas NO están desplegadas.** Desplegarlas es tarea del dueño (ver ⏳).

### ✅ 2. Content-Security-Policy vía `<meta http-equiv>` (mitiga puntos 5 y 6)

GitHub Pages no permite headers HTTP, así que la CSP va por `<meta>` en `<head>` de `index.html`. Es **segunda línea de defensa** sobre el `escapeHtml()` ya existente:

```
default-src 'self';
script-src  'self' 'unsafe-inline' https://www.gstatic.com;   (SDK Firebase compat)
style-src   'self' 'unsafe-inline' https://fonts.googleapis.com;
font-src    'self' https://fonts.gstatic.com;
img-src     'self' data:;
connect-src 'self' https://*.firebaseio.com wss://*.firebaseio.com
                   https://*.firebasedatabase.app wss://*.firebasedatabase.app;
manifest-src 'self'; worker-src 'self';
object-src 'none'; frame-ancestors 'none'; base-uri 'self'; form-action 'none'
```

- `'unsafe-inline'` en `script-src`/`style-src` es **necesario**: la app tiene un `<script>` y un `<style>` inline y no hay build step para inyectar nonces. Aun así la CSP aporta: limita orígenes de scripts/conexiones, bloquea `object`/`frame`/`base`/`form-action`, y restringe `connect-src` a Firebase RTDB únicamente.
- **Verificación:** inline script re-chequeado con `node --check` (líneas 502–1189) → OK. CSP no rompe los orígenes en uso (gstatic, fonts.googleapis, fonts.gstatic, firebaseio/firebasedatabase).

**Nota:** esta CSP NO es tan estricta como una con nonces (por el `'unsafe-inline'`). Endurecerla del todo requeriría externalizar el JS/CSS inline o introducir un build step con nonces → mejora futura, no bloqueante.

### ✅ 3. `.gitignore` — ya existía y es correcto

Revisado: cubre `.env`, `.env.local`, `dist/`, `build/`, `node_modules/`, `.vscode/`, `.idea/`, `.claude/`, `*.log`, `.DS_Store`. **No se requirió cambio.**

---

## 📄 Documentado (sin tocar código)

- **Service Worker (`sw.js`)** y **caché** ya están bien (punto 8 🟢). Sin cambios.
- **apiKey de Firebase en el cliente** NO es un secreto (identificador público de proyecto). No requiere rotación; si se quiere, se restringe por dominio/HTTP referrer en Google Cloud Console. Documentado en auditoría, sin acción.
- **CSP con `'unsafe-inline'`**: aceptable hoy; el endurecimiento total (nonces/externalizar inline) queda como mejora futura.

---

## ⏳ Pendiente del dueño (acciones fuera de código / arquitectura)

### ⏳ A. Desplegar las reglas endurecidas

Las reglas nuevas están en `firebase-rules.json` pero **NO desplegadas**. Pasos:

1. Revisar el diff de `firebase-rules.json`.
2. En la **consola de Firebase** → Realtime Database → Reglas → pegar el contenido del archivo → **Publicar**.
   - (o con CLI: `firebase deploy --only database` desde un proyecto con `firebase.json` apuntando a `firebase-rules.json`.)
3. **Probar en staging primero si es posible** (ver punto D). Tras publicar, verificar que la app sigue funcionando:
   - crear/editar/borrar un jugador, crear/editar un turno, cambiar nombre de device, ver que el log registra.
   - Las `.validate` pueden rechazar escrituras si el cliente cambió de forma; si algo se rompe, **revertir** pegando las reglas viejas (siguen en el historial de git y en el historial de versiones de reglas de Firebase).

> Importante: esto **endurece** pero **no cierra**. Para cerrar de verdad, hacer B.

### ⏳ B. Firebase Auth para aislamiento real (el cierre verdadero) — cambio de arquitectura

Hoy cualquiera con la `databaseURL` (pública) puede leer/sobrescribir/borrar. Para cerrarlo:

- **Mínimo viable:** activar **Firebase Auth anónima**. En `index.html`, llamar `firebase.auth().signInAnonymously()` al arrancar, esperar el sign-in antes de attachListeners, y cambiar reglas a `".read": "auth != null"` / `".write": "auth != null"`. Esto corta el grueso del abuso anónimo HTTP directo (ya no basta conocer la URL: hay que pasar por el flujo de Auth) y da un `auth.uid` estable por device.
- **Mejor:** **login real** (Google federado). Permite:
  - reglas con `auth.uid` para **aislamiento por usuario** (no-repudio real en el audit, no el `device_name` cosmético actual).
  - distinguir admins de invitados si se quiere (custom claims).
- En ambos casos, reemplazar el `deviceId` de `localStorage` por `auth.uid` como identidad del audit.

**Esfuerzo:** medio. Toca `index.html` (flujo de init), `firebase-rules.json` (de `true` a `auth != null` + scoping por uid) y la consola (habilitar el provider de Auth).

### ⏳ C. Rate limiting + budget alert (puntos 7 y 10)

- **Budget alert** en Google Cloud / Firebase Billing: configurar alerta de presupuesto (ej. avisar a $X) para detectar abuso de cuota / costos silenciosos. **Acción de consola, alta prioridad, bajo esfuerzo.**
- **Rate limiting** real requiere backend o Cloud Functions (RTDB no limita frecuencia nativamente). Las `.validate` de longitud ya acotan tamaño por campo, pero no frecuencia. Atado a B: con Auth se puede limitar por uid y cortar floods anónimos.

### ⏳ D. Monitoreo / observabilidad (punto 10)

- Integrar **error tracker** (Sentry o similar) para no perder fallos silenciosos (`recordChange` hace `console.error`, el SW usa `.catch(() => {})`).
- Considerar **separación de entornos** (dev/staging/prod): hoy hay un solo proyecto Firebase (`padle-82f32`); cualquier prueba impacta "producción". Un proyecto de staging permite probar reglas y Auth sin riesgo.

---

## Resumen

| Acción | Estado | Dónde |
|---|---|---|
| `.validate` + `.read/.write:false` por defecto en reglas | ✅ código | `firebase-rules.json` |
| `/audit` append-only conservado | ✅ código | `firebase-rules.json` |
| CSP vía `<meta>` | ✅ código | `index.html` |
| `.gitignore` | ✅ ya existía | `.gitignore` |
| Comentario honesto "esto no aísla por usuario" | ✅ código | `firebase-rules.json` |
| Desplegar reglas endurecidas | ⏳ dueño | consola Firebase |
| Firebase Auth (anónima mínima / login real) + reglas por uid | ⏳ dueño | arquitectura |
| Budget alert + rate limiting | ⏳ dueño | consola / backend |
| Monitoreo (Sentry) + entornos dev/staging/prod | ⏳ dueño | infra |

**Honestidad:** lo de esta tanda **reduce el daño** (no se escribe basura, no se borran/crean estructuras arbitrarias tan fácil, audit sigue append-only, XSS con doble defensa) pero **NO sustituye** Auth. Mientras la DB siga abierta sin Auth, el cierre real es ⏳ del dueño (acción B).
