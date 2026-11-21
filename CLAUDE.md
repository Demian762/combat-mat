# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Proyecto

Herramientas para sesiones de D&D, hosteadas en GitHub Pages con Firebase Realtime Database como backend compartido en tiempo real.

### Archivos
| Archivo | Descripción |
|---|---|
| `combat-mat.html` | App principal: mat de combate 20×20 + dados 3D integrados |
| `log.html` | Overlay OBS: log de tiradas en tiempo real, fondo transparente |
| `dice.html` | Overlay OBS: solo dados 3D animados, fondo transparente |
| `mapa.jpg` | Mapa del mundo que muestra el botón **Mapa** (junto al desplegable de escenas) en lugar del mat |

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
      /terrain        → {"col,row": terrainType | "#rrggbb"}   // tipo con nombre (paleta Simple) o color libre (paleta Completa)
      /enemies        → {id: {id, name, size, col, row}}
      /order          → [id, id, ...]        // orden de iniciativa
      /turn           → id | null
      /pickerStates   → {bg:{h,s,v}, line:{h,s,v}}
      /arrows
        /{CLIENT_ID}  → {fromId, toCol, toRow}  // flecha de arrastre en curso, una por cliente (onDisconnect remove)
    /escena2 … /escena10   // misma forma
  /players
    /{id}             → {id, name, cls, hp, maxHp, color, size, sceneId, col, row, owner?}   // owner = uid del jugador que lo creó (no se pone si lo crea el master)
  /chest
    /{id}             → token completo (enemy o player sin owner, sin sceneId)   // baúl del master: fuera de toda escena
  /uid                → number              // contador global de ids
  /rolls
    /{pushId}         → {player, dice, rolls, modifier, total, timestamp, color, duration, hidden?}
                      | {kind:'scene', player, from, to, timestamp}   // cambio de escena de un jugador ("Olbert cambió de escena 1 a 2")
```

### Lo que NO va a Firebase (estado cliente-local)
| Campo | Motivo |
|---|---|
| `ox`, `oy` | Cada cliente navega el mapa independientemente |
| `zoomLevel` | Rueda sobre el mat: 3 niveles sin animación — 1 = 10×10 celdas de 88px, 2 = 20×20 de 44px (default), 3 = 40×40 de 22px. El mat sigue midiendo `VIEW`=880px; cambian `COLS/ROWS` y `CS`, y las fichas escalan con `--cs`. La celda bajo el cursor queda fija. No se guarda |
| `activeScene` | Cada cliente elige qué escena ver |
| mapa on/off | Botón **Mapa**: tapa el mat con `mapa.jpg` (clase `show-map` en `body`); los dados siguen visibles encima y las flechas se ocultan. Rueda = zoom hacia el cursor (1× = imagen entera, máx. `MAP_ZMAX`), arrastrar = mover; vista en `mapView`, nunca sale del borde |
| `selId` | Selección local para medir distancias |
| `activeTerrain` | Pincel de terreno activo |
| `drag` | Estado de arrastre en curso |
| `pickerStates.player` | Color del picker de nuevo jugador — elección individual, no debe imponerse a otros clientes |
| `maxHp` | HP Máx para el próximo token — cada persona arranca en **10** al cargar (antes `/sala/maxHp`, ya no se usa) |
| `palMode` | Simple/Completa de cada paleta del panel derecho — preferencia de cada persona, en `localStorage` (`palMode`) |
| `leftSplit` | Proporción Combatientes / Baúl de personajes del panel izquierdo (solo master) — en `localStorage` (`leftSplit`) |
| `dDuration` | Duración de la tirada en segundos (barra "Rebote") — preferencia de cada persona, en `localStorage` (`diceDuration`); viaja en cada tirada como `duration` |

---

## Decisiones de diseño clave

### Roles y permisos
- **Master**: la cuenta `MASTER_EMAIL` (hardcodeada en JS). Puede crear personajes sin límite y cambiar de escena libremente.
- **Jugadores** (resto de `ALLOWED_EMAILS`): pueden crear **un solo personaje a la vez** (`owner` = su uid; el botón "+ Jugador" se deshabilita mientras exista). En el panel de dados, **Nombre** no es un desplegable para ellos: muestra fijo su personaje, y sin personaje no pueden tirar (dados, Porcentaje ni Todo o Nada). Solo el master elige nombre del desplegable, que arranca en **Master** y lista Master y los personajes que creó él (sin `owner`) presentes en la escena, no los de los jugadores. No existe "Anónimo".
- Si un jugador cambia de escena, **su personaje viaja con él** (se reubica en la celda libre más cercana) y el cambio se registra en `/sala/rolls` como `kind:'scene'` y se muestra en el **Historial** y en `log.html` como "Olbert cambió de escena 1 a 2" (`dice.html` lo ignora: no tiene `rolls`). Solo se registra si el jugador tiene un personaje propio (con `owner`).
- Un jugador siempre ve la escena donde está su personaje: al cargar, o si su `sceneId` cambia desde otro cliente, su vista lo sigue.
- **Borrar**: el master borra cualquier ficha; un jugador solo su propio personaje (la ✕ del panel izquierdo y la tecla Supr no aparecen/no actúan sobre las demás).
- **Solo master** (clase `master-only`, ocultos para jugadores vía `body:not(.is-master)`): **+ Enemigo**. (Ya no existe "Limpiar todo".)
- Los **personajes que crea el master** (+ Jugador) van **directo al baúl**, sin `owner`; los enemigos se crean en la escena activa como siempre.
- **Vida oculta**: los jugadores no ven HP ni barra de los personajes del master (sin `owner`), ni pueden ajustarla; ven una palabra según `hp/maxHp`: **Entero** ≥90 %, **Golpeado** 51–89 %, **Herido** 20–50 %, **En las últimas** 1–19 %, **Caído** con 0 HP (`hpWord`). Los enemigos no tienen HP.
- **Baúl de personajes** (solo master): el panel izquierdo se divide en **Combatientes** arriba y **Baúl de personajes** abajo, cada uno con su scroll. El divisor entre ambos (`#left-split`) se arrastra; la proporción (15–85 %, default mitad) se guarda en `localStorage` (`leftSplit`). El master arrastra fichas entre ambos: al guardar, la ficha sale de la escena activa (y de su orden/turno) a `/sala/chest`, invisible para todos; al sacarla, entra en la escena activa (en su celda anterior si está libre y visible) y en el orden donde se soltó. El baúl es global: sirve para pasar enemigos y personajes del master entre escenas. Los personajes de jugadores (con `owner`) no van al baúl. Los jugadores solo ven Combatientes, a altura completa.
- **Vida**: el master ajusta la de cualquiera; un jugador solo la de su propio personaje (`canEditHp`, misma regla que borrar). La de los otros jugadores la ve (número y barra) pero sin botones.
- **Movimiento libre a propósito**: cualquier cliente (master o jugador) puede mover cualquier ficha del mat, incluidos los personajes de otros jugadores y los enemigos. No restringirlo.
- Fuera de eso no hay permisos: cualquier cliente puede pintar terreno, cambiar colores de la grilla, etc.

### Modo prueba
`combat-mat.html?prueba` (como jugador) o `?prueba=master` (como master) usa `ROOT = 'sala/prueba'` en vez de `sala`: misma estructura, datos aparte, cartel "MODO PRUEBA" arriba. El rol sale de la URL, no del email, y solo dentro de la prueba. Todas las rutas de Firebase en `combat-mat.html` se arman con `ROOT` — no escribir `'sala/...'` literal. `log.html` y `dice.html` siguen leyendo solo `sala/rolls`.

### Escenas
- 10 escenas fijas pre-creadas: **"Escena 1"** … **"Escena 10"** (nombres no editables desde UI).
- Cada escena tiene su propio terreno, enemigos, orden de turno y colores de grilla.
- El **color picker de fondo y líneas** (`pickerStates.bg` y `pickerStates.line`) es **por escena**.
- El **color de nuevo jugador** (`pickerStates.player`) es **estado local por cliente**, no se sincroniza vía Firebase — cada jugador elige su propio color independientemente.
- Cambiar de escena es **independiente por cliente** — cada uno elige qué escena ver via dropdown. No hay "escena activa global".

### Paletas del panel derecho
Dos paletas separadas, cada una con modo **Simple** (muestras fijas) y **Completa** (selector HSV):
- **Mapa** (arriba): destino Terreno / Fondo / Líneas. Simple = los 14 terrenos de `TERRAIN` (en Fondo/Líneas esos mismos colores se aplican a la grilla). Completa en Terreno = pincel de color libre (se guarda el hex en `/terrain`). El botón de terreno actual (último pincel elegido, `lastBrush`: terreno con nombre o color libre; clic = prender/apagar) y el ✕ borrar van en `#terr-tools`, a la izquierda de las flechas de pan; ambos solo con destino Terreno, en los dos modos.
- **Color** del nuevo personaje (dentro de Nuevo token): Simple = 10 colores vivos (`TOK_COLORS`), sin borrar. Completa = selector HSV.

### Movimiento de jugadores entre escenas
- Los tokens `type:'player'` tienen un campo `sceneId` que indica en qué escena están físicamente.
- Cada jugador mueve su propio personaje al cambiar de escena (ver Roles y permisos). No hay botón para traer a todos.
- Los tokens `type:'enemy'` son propios de cada escena; no se transfieren.
- Cuando un cliente ve la Escena 3, solo renderiza los enemigos de esa escena + los jugadores cuyo `sceneId === 'escena3'`.

### Viewport (pan)
`ox` y `oy` son estado cliente-local. Nunca van a Firebase. Cada jugador puede moverse libremente por el mapa sin afectar a los demás. Se mueve con las flechas del panel derecho o arrastrando el terreno (sin pincel activo y fuera de las fichas): de a casillas enteras, el mapa sigue al cursor (`gridPan`). Clic en las coordenadas del centro de las flechas = `resetView()`: zoom nivel 2 y vista en 0,0.

### Persistencia
Firebase Realtime Database guarda los datos indefinidamente hasta borrado explícito. El terreno pintado, fichas y HP de una sesión persisten en la siguiente.

### Autenticación
Firebase Auth con Google OAuth. Lista de emails permitidos (`ALLOWED_EMAILS`) hardcodeada en JS. Solo los emails de esa lista pueden acceder. `log.html` no requiere auth (es overlay OBS, solo lee `sala/rolls`).

### Anti-loop de sincronización
Cada cliente genera un `CLIENT_ID` aleatorio al cargar. Cada escritura a Firebase incluye `_sender: CLIENT_ID`. Al recibir un update, si `_sender === CLIENT_ID` y el JSON coincide con el último escrito, se ignora para no re-renderizar el propio estado.

---

## Dados (integrado en combat-mat.html)

### Dados soportados
`d4`, `d6`, `d8`, `d10`, `d12`, `d20`.

### Geometrías Three.js
| Dado | Geometría |
|---|---|
| d4 | `TetrahedronGeometry` |
| d6 | `BoxGeometry` |
| d8 | `OctahedronGeometry` |
| d10 | `makeD10Geometry` (trapezoedro pentagonal custom, 10 caras rombo × 2 triángulos) |
| d12 | `DodecahedronGeometry` |
| d20 | `IcosahedronGeometry` |

### Flujo de tirada
1. Se generan los resultados numéricos al instante del clic.
2. Se guardan en Firebase **antes** de iniciar la animación, con `hidden:true`.
3. La animación 3D local termina mostrando el número ya registrado (easing out).
4. Los demás clientes (y `dice.html`) reciben la tirada desde Firebase y reproducen la misma animación con esos resultados.
5. Al asentarse los dados, el cliente que tiró borra `hidden` y el valor aparece en resultado, historial y `log.html`.

### Rebote por persona
Deslizador **Rebote** en el panel de dados: elige la **duración exacta** de la tirada, 1.5–100 s en escala logarítmica (default 6 s). Se guarda en `localStorage` y se escribe en la tirada (`duration`), así los demás clientes y `dice.html` animan lo mismo; tiradas sin `duration` usan el default.
- Más duración → más restitución (`dRestFor`: 1.5 s → 0.30, 2 saltos; 6 s → 0.78; 100 s → 0.96, ~80 saltos). Pasados ~25 s, el resto se logra con cámara lenta.
- Los rebotes son **analíticos** (`dBounces`/`dBounceY`: parábolas exactas, no integración por cuadro). Cada dado estira su calendario natural por `lam = dur/T` para asentarse exactamente en `dur`.
- La animación depende del **tiempo real** (`performance.now()`), no de cuadros: dura lo mismo a 60 o 144 Hz.
- Los dados se desvanecen a los 10 s, o 3 s después de asentarse si eso es más tarde (`fadeAt`, en segundos).

### Revelado al terminar la animación
**Todas** las tiradas (dados, Porcentaje, Todo o Nada) se escriben con `hidden:true` para que todos animen en simultáneo; el cliente que tira borra `hidden` al terminar su animación (o a `duration` + 3 s como respaldo) y recién ahí muestra el resultado local. Los logs (`combat-mat` historial y `log.html`) ignoran entradas `hidden` y las muestran en `child_changed`.

### Ruta Firebase
`/sala/rolls/{pushId}` con forma `{player, dice, rolls, modifier, total, timestamp}`.

Tiradas especiales (botones **Porcentaje** y **Todo o Nada**):
- Porcentaje: `rolls = {d100:[decenas 0–90], d10:[unidades 1–10]}`, `total` 1–100. `d100` es un d10 con caras 0,10…90.
- Todo o Nada: `rolls = {coin:["Todo"|"Nada"]}` (moneda dorada: estrella = Todo, cruz = Nada), `total` = `"Todo"`/`"Nada"`.

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
- Tamaño del mat: `--view:880px` en CSS ↔ `VIEW=880` en JS. Tamaño de celda: `CS = VIEW/COLS` según el zoom; `setZoomLevel` actualiza `--cs` — no fijar 44px a mano ni usar `20*var(--cs)` para alturas (usar `--view`).
- Coordenadas siempre enteras. Nunca almacenar posiciones fraccionarias.
- Enemigos: fondo `rgba(90,8,8,0.88)`, borde rojizo. Jugadores: círculo con color configurable.
- **Tamaño** (desplegable junto a Clase en Nuevo token, default Mediano; vale para jugadores y enemigos): campo `size` = `tiny|small|medium|large|huge|gargantuan` (`SIZES`; sin `size` = Mediano). Ocupa `n×n` casillas según D&D 5e — Diminuto/Pequeño/Mediano 1, Grande 2×2, Enorme 3×3, Gargantuesco 4×4 — con `col,row` = esquina superior izquierda. Diminuto se dibuja a media casilla y Pequeño a ¾ (ocupa 1 casilla igual que Mediano, pero la ficha es más chica); hasta 4 Diminutas solas en una casilla van una por cuadrante. Al crear/sacar del baúl/cambiar de escena se busca lugar para todo el cuadrado (`footprint`, `fitsAt`). Al arrastrar, la casilla por la que se agarró queda bajo el cursor; la distancia se mide entre esquinas.
- Fichas en la misma celda: se abren en abanico tipo mano de cartas (`layoutStacks`: corridas y apenas giradas, en orden de iniciativa), solo visual — `col/row` no cambian. En una pila solo el cuerpo (`.tbody`) recibe el mouse, así cada ficha se puede clickear/arrastrar por su franja visible. Para redibujar una ficha usar `renderTok` (recalcula el abanico); `drawTok` solo dentro de bucles que después llaman `layoutStacks()`.

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
