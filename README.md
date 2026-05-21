# Padel · Turnos

App de tracking de turnos de padel para un grupo de amigos. Cualquiera con el
link ve y edita los turnos en tiempo real, y todo cambio queda registrado
en un audit log inmutable a nombre de quien lo hizo.

## Cómo funciona

Cada turno tiene dos listas separadas:

- **Reservados** — los que se anotaron (los que pagan).
- **Jugaron** — los que efectivamente jugaron.

De la diferencia salen los balances:

- Reservó pero no jugó → **+1 crédito** (le deben).
- Jugó sin reservar → **−1 débito** (debe).
- Reservó y jugó → neutro.

## Stack

- HTML vanilla + JS, **sin frameworks ni build step**.
- **Firebase Realtime Database** (sync reactivo entre todos los clientes).
- Deploy: **GitHub Pages** (archivo único `index.html`).
- Mobile-first, dark.

## Setup

### 1. Firebase

1. Crear un proyecto en [console.firebase.google.com](https://console.firebase.google.com).
2. Crear una **Realtime Database** (modo de prueba está bien al principio).
3. En *Project settings → General → Your apps → Web app*, copiar el objeto
   `firebaseConfig`.
4. Pegarlo en `index.html` reemplazando los `"PEGAR_ACA"`.
5. En *Database → Rules*, pegar el contenido de [`firebase-rules.json`](./firebase-rules.json).
   Esa regla hace que `/audit` sea **append-only** (no se puede modificar
   ni borrar lo escrito).

### 2. GitHub Pages

1. *Settings → Pages → Source: Deploy from a branch → `main` / `(root)`*.
2. Guardar. En ~1 min queda en `https://<usuario>.github.io/<repo>/`.

### 3. Instalar como app (PWA)

- **iPhone**: abrí la URL en **Safari** (no Chrome — en iOS sólo Safari instala PWAs) → botón Compartir → *Agregar a pantalla de inicio*.
- **Android**: menú del navegador → *Instalar app* (o *Agregar a pantalla de inicio*).

Una vez instalada se abre fullscreen sin barra del navegador, con su ícono propio y sigue funcionando aunque pierdas señal momentáneamente (gracias al service worker).

## Estructura de datos (RTDB)

```
/jugadores/{jid}            { id, nombre }
/turnos/{tid}               { id, fecha, reservados[], jugaron[], createdAt }
/devices/{deviceId}         { name, firstSeen }
/audit/{autoKey}            { ts, action, entity, entity_id, before, after,
                              device, device_name }
```

## URL del deploy

Pegá acá tu link cuando esté activo:

```
https://mdqclio.github.io/Padle/
```
