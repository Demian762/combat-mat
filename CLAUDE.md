# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Proyecto

Herramientas para sesiones de D&D, hosteadas en GitHub Pages con Firebase Realtime Database como backend compartido en tiempo real.

### Archivos
| Archivo | Descripción |
|---|---|
| `combat-mat.html` | App principal: mat de combate 20×20 + dados 3D integrados |
| `log.html` | Overlay OBS: log de tiradas en tiempo real, fondo transparente |

Sin build tools, sin frameworks, sin Node.js. Cada archivo es HTML/CSS/JS autocontenido con librerías via CDN.

---

## Stack

- **Firebase Realtime Database** — sincronización de estado entre clientes
- **Three.js** (CDN) — dados 3D en `combat-mat.html`
- **GitHub Pages** — hosting estático

---

## Estructura Firebase

```
/sala
  /scenes
    /escena1
      /terrain        → {"col,row": terrainType}
      /enemies        → {id: {id, name, col, row}}
      /order          → [id, id, ...]        // orden de iniciativa
      /turn           → id | null
      /pickerStates   → {bg:{h,s,v}, line:{h,s,v}}
    /escena2 … /escena10   // misma forma
  /players
    /{id}             → {id, name, cls, hp, maxHp, color, sceneId, col, row}
  /uid                → number              // contador global de ids
  /maxHp              → number              // valor por defecto nuevos tokens
  /rolls
    /{pushId}         → {player, dice, rolls, modifier, total, timestamp}
```

### Lo que NO va a Firebase (estado cliente-local)
| Campo | Motivo |
|---|---|
| `ox`, `oy` | Cada cliente navega el mapa independientemente |
| `activeScene` | Cada cliente elige qué escena ver |
| `selId` | Selección local para medir distancias |
| `activeTerrain` | Pincel de terreno activo |
| `drag` | Estado de arrastre en curso |
| `pickerStates.player` | Color del picker de nuevo jugador — elección individual, no debe imponerse a otros clientes |

---

## Decisiones de diseño clave

### Roles y permisos
No existen. Cualquier cliente conectado puede mover cualquier ficha, pintar terreno, ajustar HP, etc.

### Escenas
- 10 escenas fijas pre-creadas: **"Escena 1"** … **"Escena 10"** (nombres no editables desde UI).
- Cada escena tiene su propio terreno, enemigos, orden de turno y colores de grilla.
- El **color picker de fondo y líneas** (`pickerStates.bg` y `pickerStates.line`) es **por escena**.
- El **color de nuevo jugador** (`pickerStates.player`) es **estado local por cliente**, no se sincroniza vía Firebase — cada jugador elige su propio color independientemente.
- Cambiar de escena es **independiente por cliente** — cada uno elige qué escena ver via dropdown. No hay "escena activa global".

### Movimiento de jugadores entre escenas
- Los tokens `type:'player'` tienen un campo `sceneId` que indica en qué escena están físicamente.
- Cada escena tiene un botón **"Traer personajes acá"** que actualiza el `sceneId` de todos los jugadores a esa escena y resetea sus coordenadas `col/row`.
- Los tokens `type:'enemy'` son propios de cada escena; no se transfieren.
- Cuando un cliente ve la Escena 3, solo renderiza los enemigos de esa escena + los jugadores cuyo `sceneId === 'escena3'`.

### Viewport (pan)
`ox` y `oy` son estado cliente-local. Nunca van a Firebase. Cada jugador puede moverse libremente por el mapa sin afectar a los demás.

### Persistencia
Firebase Realtime Database guarda los datos indefinidamente hasta borrado explícito. El terreno pintado, fichas y HP de una sesión persisten en la siguiente.

### Autenticación
Firebase Auth con Google OAuth. Lista de emails permitidos (`ALLOWED_EMAILS`) hardcodeada en JS. Solo los emails de esa lista pueden acceder. `log.html` no requiere auth (es overlay OBS, solo lee `sala/rolls`).

### Anti-loop de sincronización
Cada cliente genera un `CLIENT_ID` aleatorio al cargar. Cada escritura a Firebase incluye `_sender: CLIENT_ID`. Al recibir un update, si `_sender === CLIENT_ID` y el JSON coincide con el último escrito, se ignora para no re-renderizar el propio estado.

---

## Dados (integrado en combat-mat.html)

### Dados soportados
`d4`, `d6`, `d8`, `d12`, `d20`. Sin d10 (geometría compleja).

### Geometrías Three.js
| Dado | Geometría |
|---|---|
| d4 | `TetrahedronGeometry` |
| d6 | `BoxGeometry` |
| d8 | `OctahedronGeometry` |
| d12 | `DodecahedronGeometry` |
| d20 | `IcosahedronGeometry` |

### Flujo de tirada
1. Se generan los resultados numéricos al instante del clic.
2. Se guardan en Firebase **antes** de iniciar la animación.
3. La animación 3D local termina mostrando el número ya registrado (easing out).
4. Los demás clientes reciben el resultado numérico desde Firebase; no ven la animación ajena.

### Ruta Firebase
`/sala/rolls/{pushId}` con forma `{player, dice, rolls, modifier, total, timestamp}`.

---

## log.html

- Lee de `/sala/rolls` en tiempo real (mismo proyecto Firebase, misma sala).
- No necesita room key — sala es única y hardcodeada en el config.
- `body` con `background: transparent` para OBS Browser Source.
- Solo visualización: sin controles ni interacción.
- Muestra las últimas tiradas con animación CSS de entrada (slide-in o fade-in).
- Tipografía grande con sombra/contorno para legibilidad sobre cualquier fondo de stream.

---

## Convenciones visuales (combat-mat.html)

- Fuentes: `'Cinzel'` (headers, labels, números) · `'EB Garamond'` (cuerpo).
- Paleta: `--amber:#c8922a`, `--amber-lt:#f0c060`, `--amber-dim:rgba(200,146,42,0.2)`, `--panel:rgba(8,6,4,0.97)`, `--border:rgba(200,146,42,0.22)`.
- Tamaño de celda: `--cs:44px` en CSS ↔ `CS=44` en JS — deben mantenerse sincronizados.
- Coordenadas siempre enteras. Nunca almacenar posiciones fraccionarias.
- Enemigos: fondo `rgba(90,8,8,0.88)`, borde rojizo. Jugadores: círculo con color configurable.

---

## CDN a incluir en combat-mat.html

```html
<!-- Firebase -->
<script src="https://www.gstatic.com/firebasejs/10.x.x/firebase-app-compat.js"></script>
<script src="https://www.gstatic.com/firebasejs/10.x.x/firebase-database-compat.js"></script>
<script src="https://www.gstatic.com/firebasejs/10.x.x/firebase-auth-compat.js"></script>
<!-- Three.js -->
<script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
```
