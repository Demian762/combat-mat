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
      /terrain        → {"col,row": terrainType | "#rrggbb" | "piso+encima"}   // tipo con nombre de TERRAIN (paleta Simple, con textura),
                      // color libre (paleta Completa), o dos capas: "stone+blood", "grass+fire" (ver Texturas de terreno)
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
    /{pushId}         → {player, dice, rolls, modifier, total, timestamp, color, duration, by, hidden?}
                      | {kind:'scene', player, from, to, timestamp, by}   // cambio de escena de un jugador ("Olbert cambió de escena 1 a 2")
                      // by = quién tiró: uid, con prefijo 'master:' si era el master. En el Historial de combat-mat
                      // cada uno ve sus propias entradas en blanco (isOwnRoll); log.html las muestra todas iguales
```

### Lo que NO va a Firebase (estado cliente-local)
| Campo | Motivo |
|---|---|
| `ox`, `oy` | Cada cliente navega el mapa independientemente |
| `zoomLevel` | Rueda sobre el mat: 3 niveles sin animación — 1 = 10×10 celdas de 88px, 2 = 20×20 de 44px (default), 3 = 40×40 de 22px. El mat sigue midiendo `VIEW`=880px; cambian `COLS/ROWS` y `CS`, y las fichas escalan con `--cs`. La celda bajo el cursor queda fija. No se guarda |
| `activeScene` | Cada cliente elige qué escena ver |
| mapa on/off | Botón **Mapa**: tapa el mat con `mapa.jpg` (clase `show-map` en `body`); los dados siguen visibles encima y las flechas se ocultan. Rueda = zoom hacia el cursor (1× = imagen entera, máx. `MAP_ZMAX`), arrastrar = mover; vista en `mapView`, nunca sale del borde |
| `selId` | Copia local de `/turn` (la ficha del turno); la escribe el listener de `/turn`, no es una selección aparte |
| `masterSel` | Ficha que eligió el master para tirar (ver Selección / turno). Local a su pestaña: al refrescar o cambiar de escena arranca vacía |
| `activeTerrain` | Pincel de terreno activo |
| `drag` | Estado de arrastre en curso |
| `hoverTok` | Ficha bajo el mouse: muestra `#tok-label` con su nombre arriba de la ficha (debajo si está pegada al borde de arriba) mientras el mouse esté encima; se oculta al arrastrar. Cada uno ve solo la de su propio mouse (`showTokLabel`, se reubica en `layoutStacks`) |
| `pickerStates.player` | Color del picker de nuevo jugador — elección individual, no debe imponerse a otros clientes |
| `maxHp` | HP Máx para el próximo token — cada persona arranca en **10** al cargar (antes `/sala/maxHp`, ya no se usa) |
| `palMode` | Simple/Completa de cada paleta del panel derecho — preferencia de cada persona, en `localStorage` (`palMode`) |
| `leftSplit` | Proporción Combatientes / Baúl de personajes del panel izquierdo (solo master) — en `localStorage` (`leftSplit`) |
| `dDuration` | Duración de la tirada en segundos (barra "Rebote") — preferencia de cada persona, en `localStorage` (`diceDuration`); viaja en cada tirada como `duration` |

---

## Decisiones de diseño clave

### Roles y permisos
- **Master**: la cuenta `MASTER_EMAIL` (hardcodeada en JS). Puede crear personajes sin límite y cambiar de escena libremente.
- **Jugadores** (resto de `ALLOWED_EMAILS`): tienen **un solo personaje a la vez** (`owner` = su uid). El botón de la sección Gestión de Tokens es dinámico (`updateCharBtn`): sin personaje dice **Nuevo personaje**; con personaje, el formulario (Nombre, Clase, Tamaño, HP Máx, Color) muestra siempre sus datos (`syncCharForm`, se recarga solo cuando cambian esos datos, no al moverlo ni con la vida) y el botón dice **Borrar personaje** (pide confirmación) si el formulario coincide, o **Editar personaje** si se tocó algo (`editChar`: actualiza esos campos; la vida se recorta al nuevo máximo; Nombre vacío = no cambia). En el panel de dados, **Nombre** es fijo (no hay desplegable para nadie): el jugador tira como su personaje, y sin personaje no puede tirar (dados, Porcentaje ni Todo o Nada). El master tira como la ficha que tiene seleccionada (`masterSel`, ver Selección / turno) o como **Master** si no tiene ninguna; el nombre cambia solo. El dado toma el color de la ficha si es jugador/NPC; enemigos y Master usan los colores por defecto. No existe "Anónimo".
- Si un jugador cambia de escena, **su personaje viaja con él** (se reubica en la celda libre más cercana) y el cambio se registra en `/sala/rolls` como `kind:'scene'` y se muestra en el **Historial** y en `log.html` como "Olbert cambió de escena 1 a 2" (`dice.html` lo ignora: no tiene `rolls`). Solo se registra si el jugador tiene un personaje propio (con `owner`).
- Un jugador siempre ve la escena donde está su personaje: al cargar, o si su `sceneId` cambia desde otro cliente, su vista lo sigue.
- **Borrar**: el master borra cualquier ficha; un jugador solo su propio personaje (la ✕ del panel izquierdo y la tecla Supr no aparecen/no actúan sobre las demás).
- **Solo master** (clase `master-only`, ocultos para jugadores vía `body:not(.is-master)`): **+ Enemigo**. (Ya no existe "Limpiar todo".)
- Los **personajes que crea el master** (botón **+ NPC**, que para los jugadores dice **+ Jugador**) van **directo al baúl**, sin `owner`; los enemigos se crean en la escena activa como siempre.
- **Vida oculta**: los jugadores no ven HP ni barra de los personajes del master (sin `owner`), ni pueden ajustarla; ven una palabra según `hp/maxHp`: **Entero** ≥90 %, **Golpeado** 51–89 %, **Herido** 20–50 %, **En las últimas** 1–19 %, **Caído** con 0 HP (`hpWord`). Los enemigos no tienen HP.
- **Baúl de personajes** (solo master): el panel izquierdo se divide en **Combatientes** arriba y **Baúl de personajes** abajo, cada uno con su scroll. El divisor entre ambos (`#left-split`) se arrastra; la proporción (15–85 %, default mitad) se guarda en `localStorage` (`leftSplit`). El master arrastra fichas entre ambos: al guardar, la ficha sale de la escena activa (y de su orden; si era la seleccionada, queda sin selección) a `/sala/chest`, invisible para todos; al sacarla, entra en la escena activa (en su celda anterior si está libre y visible) y en el orden donde se soltó. El baúl es global: sirve para pasar enemigos y personajes del master entre escenas. Los personajes de jugadores (con `owner`) no van al baúl. Los jugadores solo ven Combatientes, a altura completa.
- **Vida**: el master ajusta la de cualquiera; un jugador solo la de su propio personaje (`canEditHp`, misma regla que borrar). La de los otros jugadores la ve (número y barra) pero sin botones.
- **Movimiento libre a propósito**: cualquier cliente (master o jugador) puede mover cualquier ficha del mat, incluidos los personajes de otros jugadores y los enemigos. No restringirlo.
- Fuera de eso no hay permisos: cualquier cliente puede pintar terreno, cambiar colores de la grilla, etc.

### Selección / turno
- **Turno** = `/scenes/{escena}/turn`: una sola ficha por escena, compartida por todos. Todos la ven igual: brillo que late en el mat y pelotita del panel izquierdo. Así el turno "avanza" por la lista de Combatientes, sea de un jugador o de un NPC/enemigo.
- **Jugadores**: seleccionan cualquier ficha (clic en el mat, en su pelotita, o al empezar a arrastrarla) y con eso pasan el turno.
- **Master**: tiene una selección propia, `masterSel`, **local a su pestaña** (se pierde al refrescar o cambiar de escena). Solo puede elegir fichas sin `owner` (enemigos y NPCs). Elegir una también pasa el turno a ella, así los jugadores la ven marcada, y el master tira los dados como esa ficha. Los personajes de jugadores **no** los selecciona (clic no hace nada), pero sí los puede mover.
- `masterSel` siempre es igual a `turn` o `null`: si el turno se va a otra ficha (un jugador elige otra, o la ficha se borra o va al baúl), se limpia y el master vuelve a tirar como **Master**. Si un jugador elige un NPC, el turno pasa a ese NPC pero no se vuelve la selección del master.
- **Pelotita del panel izquierdo**: una por ficha y para todos muestra el turno. El master la puede clickear solo en enemigos y NPCs; en personajes de jugadores es informativa (`.litem-cb.ro`). Para tirar como un NPC que eligió un jugador, el master hace clic en la ficha del mat.
- **Puede no haber ninguna seleccionada** (`turn = null`), y es un estado normal. Ya no rige la vieja regla de "siempre tiene que haber una".
- Deseleccionar: clic en la pelotita encendida (`toggleTurn`), que deja `turn = null` y limpia `masterSel`. Clic en la ficha ya elegida en el mat no la deselecciona.
- Nada elige una ficha automáticamente: crear un token no lo selecciona, y borrar la activa o guardarla en el baúl deja `turn = null` (no pasa a la siguiente).
- Tecla Supr: el jugador borra la ficha del turno si es suya; el master borra su `masterSel` (nunca la que eligió un jugador).

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
- **Mapa** (arriba): destino Terreno / Fondo / Líneas, modos **Prefabricado** (internamente `'simple'`) y **Completa**.
  - Prefabricado en Terreno: pestañas por sección (`TERR_CATS`, la elegida en `localStorage` `terrCat`) — **Naturaleza** (hierba, arena, tierra, nieve, barro, paja, árbol, vacío, fuego, hielo), **Fluidos** (agua, lava, ácido, sangre, nube), **Construcción** (pared, ladrillo, puerta, cerca, tejado, piedra, madera, metal), **Objetos** (trampa, escombros, cofre, oro, armero, silla, mesa). Cada muestra muestra su textura (`terrBg`). Las pestañas van en 2×2. Todo terreno nuevo va en una sección, sin límite: la grilla de muestras (`#map-swatches`) siempre mide 5 columnas × 3 filas y tiene su propia barra vertical (también en Fondo/Líneas).
  - Prefabricado en Fondo/Líneas: los colores `sw` de todo menos Objetos, aplicados a la grilla.
  - **Panel derecho sin sobrante ni scroll, medido para la vista del master** (la más alta, con mail y Cerrar sesión): Prefabricado (pestañas 2×2 + grilla fija de 3 filas de 28px con scroll propio, 142px) define la altura de la paleta y el selector Completa (`#map-sq` 124px de alto) no la pasa. Queda ~1px: si se agrega algo al panel hay que sacar alto de otro lado (flechas, márgenes) o volverá el scroll.
  - **Pincel** (`#brush-size`, entre las flechas y Limpiar terreno, solo con destino Terreno): 1×1 (por defecto, no se guarda), 3×3 o 5×5 (`brushSize`). `paintCell` pinta (o borra) el cuadrado centrado en la celda clickeada y lo escribe en una sola `update` a `/terrain`.
  - Completa en Terreno = pincel de color libre (se guarda el hex en `/terrain`). El botón de terreno actual (último pincel elegido, `lastBrush`: terreno con nombre o color libre; clic = prender/apagar) y el ✕ borrar van en `#terr-tools`, a la izquierda de las flechas de pan; ambos solo con destino Terreno, en los dos modos.
- **Color** del nuevo personaje (dentro de Gestión de Tokens): Simple = 10 colores vivos (`TOK_COLORS`), sin borrar. Completa = selector HSV.
- Los nombres de las muestras (y del ✕ borrar terreno) van en `data-tip`, no en `title`: `#ui-tip` los muestra al instante al pasar el mouse.

### Texturas de terreno
- El terreno se dibuja en una capa `<canvas id="terrain-canvas">` dentro de `#grid` (z-index 5, debajo de las fichas), no con un div por celda. `renderTerrain` redibuja las celdas visibles; el canvas se ajusta a `devicePixelRatio × appZoom` para verse nítido.
- Cada terreno con nombre tiene su textura en `TERRAIN_TEX` (ladrillos, adoquines, tablones, follaje, olas, etc.). El azar sale de la coordenada absoluta de la celda (`cellRng(col,row,salt)`): todos ven lo mismo y no cambia con pan ni zoom. Los colores libres de la paleta Completa siguen siendo cuadrados lisos.
- **Sangre** (`blood`) es un terreno más, pero se dibuja como charco con gotas sin fondo: la grilla se ve debajo.
- **Vacío** (`void`) es un abismo/precipicio, no la celda sin terreno del borrador. **Escombros** (`rubble`) marca terreno difícil.
- **Objetos** (`chest` cofre, `gold` oro, `rack` armero, `chair` silla, `table` mesa) van encima, una celda cada uno. Los que miran a sus vecinas reciben `nb` (`objNeighbors`, `OBJ_NB`): las mesas pegadas se unen en una mesa larga (`links`), la silla gira para mirar a la mesa de al lado (respaldo del otro lado) y el armero apoya la tabla contra la pared vecina (`rot`, aplicado en `terrTex`). Sin vecina, sin giro.
- **Capas**: **Fuego**, **Sangre**, **Escombros**, **Trampa**, **Cerca**, **Ácido / Veneno** (`acid`), **Nube mágica** (`cloud`) y los objetos (`TERR_OVER`) van *encima* de un terreno con nombre; la celda guarda `"piso+encima"` en un solo valor (`parseTerr`/`joinTerr`). Solos (`"fire"`, `"trap"`…) no tienen piso: fuego, trampa y cerca ponen su fondo propio (`OVER_BG`: `fireBg`, `trapBg`, `fenceBg`) y llevan la línea de grilla como un piso; las demás no tienen fondo y se ve la grilla. Hay una sola capa de encima por celda (pintar sangre sobre fuego lo reemplaza).
  - Pintar fuego/sangre conserva el piso con nombre; sobre un color libre lo reemplaza (los colores libres nunca llevan nada encima).
  - Pintar un piso con nombre conserva lo de encima; pintar un color libre reemplaza todo.
  - El borrador saca **una capa por pasada** (`terrStroke`, se vacía en cada mousedown): primero lo de encima, con otro clic el piso.
- **Formas por grupos** (`LIQ`): agua, lava, barro, hielo, vacío (pisos) y sangre, ácido, nube mágica (encima) no se dibujan celda por celda: cada grupo de celdas iguales es una sola mancha con esquinas redondeadas (intermedio entre círculo y cuadrado), orilla ondulada y rincones en L rellenos. Es **solo visual**: la celda cuenta entera aunque la esquina se recorte.
  - Forma: campo suave F (ocupación suavizada con una rampa de ancho `w` por eje, `liqConv`) + ruido de borde (`liqNoise`, coordenadas absolutas), contorno F = umbral con marching squares (`liqPath`). D = mismo campo con rampa ancha: agua profunda / centro de la lava. Cada líquido tiene bandas (orilla, cuerpo, fondo), `glow` (halo) y `drops` (gotas fuera del contorno). Regla para `amp`: menor que el primer umbral y `1-amp` mayor que la banda F más alta.
  - Las formas se calculan por bloques de 8×8 (`liqChunk`) y se cachean (`liqCache`, clave = tipo + bloque + zoom + huella de la ocupación): pan y pintar otra cosa las reusan. Se unen en un Path2D por banda (`addPath`) para no tener costuras.
  - Vacío: redondo como los árboles (`round`): se agrupa en copas igual que ellos (`treeCrowns(…,'void')`: 1×1, 2×2, 3×3, al azar en zonas extensas) y cada copa suma una gota circular al campo (`liqRoundPaths`); las vecinas se funden. No usa los bloques cacheados porque depende del agrupado de todo el terreno. Borde de roca tenue, centro más negro y sombra que se derrama sobre el piso vecino. Nube mágica: velo lila con borde difuminado (halo), dibujada **encima de todo** (`top`, después de fuego, escombros y cercas).
  - Pisos con forma apilados (`LIQ_BASE_ORDER`: vacío < barro < agua < hielo < lava; el agua que toca el vacío cae sobre el abismo): cada uno se extiende debajo de las celdas de los de arriba **pegadas a las suyas**, así una esquina redondeada deja ver al otro líquido.
  - Esquina recortada contra un terreno que no es líquido: muestra ese terreno vecino (`under`, por cuadrante: costado, arriba/abajo o diagonal); si no hay, el fondo.
  - **Agua o hielo tocando lava**: costra de obsidiana sobre el borde compartido + vapor que sube (`drawContact`, siempre animado).
- **Árboles**: copas redondas (`treeCrowns`/`drawCrowns`), más redondas que los líquidos. Se agrupan recorriendo todo el terreno fila por fila: bloque libre de 3×3 (que no sea parte de uno de 4×4) = copa grande, 2×2 = mediana, si no una copa por celda. En zonas extensas (grupo de ≥ `TREE_WILD_MIN` = 12 árboles) cada copa elige tamaño al azar (30% grande, 40% mediana, 30% chica, la que entre) y varía un poco tamaño y centro: bosque natural. El azar sale de la coordenada; no depende de la vista. Copas superpuestas con sombra abajo a la derecha; fuera de las copas, el terreno vecino (`under`, igual que los líquidos) y, si no hay ninguno, transparente (se ve el fondo). Se dibujan una vez a `terrCrownCv`.
- **Cercas**: se unen con sus vecinas en las 8 direcciones (`drawFences`): dos barandas de centro a centro, sin recortar a la celda, así una hilera horizontal, vertical o diagonal es continua. La diagonal se une solo si las dos celdas de costado no son cerca (una L no forma triángulo). Sin vecinas, horizontal. `TERRAIN_TEX.fence` queda solo para la muestra de la paleta.
- Orden de dibujo (`drawTerrScene`): pisos celda por celda (y fuego/trampa/cerca solos con su fondo; debajo de un líquido o árbol, sus vecinos) → pisos con forma → copas de árbol → línea de grilla tenue (arriba/izquierda, `terrLineColor`, sale del color de Líneas, para poder contar casillas) → obsidiana y vapor → sangre y ácido → lo de encima celda por celda → cercas → nube mágica. La sangre sola no lleva línea.
- Rendimiento: cada celda de piso quieta se dibuja una vez en un canvas chico cacheado (`terrCellBitmap`) y los pisos se juntan en una imagen (`terrBaseCv`) que la animación solo copia. El listener de `/terrain` no redibuja si el contenido no cambió (la escritura propia vuelve igual). Peor caso medido (zoom 3, todo lleno): ~20 ms al pintar o mover, ~9 ms por cuadro.
- La nube mágica se dibuja en el canvas del terreno, así que queda **debajo** de las fichas: marca la zona, no tapa a las criaturas.
- **Animación**: lava, agua, fuego, ácido y nube mágica se animan cuando forman un grupo de celdas vecinas de lado de al menos `TERR_ANIM[tipo]` celdas: el fuego siempre (1), el resto desde 4. Cada capa agrupa por separado (el fuego sobre piedra cuenta con el resto del fuego). Cada cuadro redibuja la escena ya armada (sin recalcular formas) a ~20 cuadros/s, o más lento si un cuadro tarda (intervalo = 4× lo que tardó, máx. 250 ms); se pausa con el mapa del mundo encima o la pestaña oculta. Las texturas animadas llaman a `R()` la misma cantidad de veces con o sin `t`, así la forma no cambia.
- Pendiente: fuego por grupos (llamas repartidas, más altas al centro, chispas); puertas que se orientan solas, sombra de paredes y mezcla entre suelos.

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
5. Al asentarse los dados, el cliente que tiró borra `hidden` y el valor aparece en resultado, historial y `log.html`. Si cierra la pestaña antes, un `onDisconnect` borra `hidden` igual.

`timestamp` es `ServerValue.TIMESTAMP` (hora del servidor). Para saber qué tiradas son nuevas y hay que animar, `combat-mat` y `dice.html` no comparan relojes: ignoran las que llegan antes del primer evento `value` y las de clave push ≤ la última vista al cargar.

Las texturas de los dados se cachean por número+color (`dNumTex`/`dCoinTex`) y al desaparecer cada dado se libera su geometría y materiales (`dDisposeMesh`): no crear texturas sueltas por dado.

Todo texto que venga de Firebase y vaya a `innerHTML` pasa por `esc()`.

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
- **Tamaño** (desplegable junto a Clase en Gestión de Tokens, default Mediano; vale para jugadores y enemigos): campo `size` = `tiny|small|medium|large|huge|gargantuan` (`SIZES`; sin `size` = Mediano). Ocupa `n×n` casillas según D&D 5e — Diminuto/Pequeño/Mediano 1, Grande 2×2, Enorme 3×3, Gargantuesco 4×4 — con `col,row` = esquina superior izquierda. Diminuto se dibuja a media casilla y Pequeño a ¾ (ocupa 1 casilla igual que Mediano, pero la ficha es más chica); hasta 4 Diminutas solas en una casilla van una por cuadrante. Al crear/sacar del baúl/cambiar de escena se busca lugar para todo el cuadrado (`footprint`, `fitsAt`). Al arrastrar, la casilla por la que se agarró queda bajo el cursor; la distancia se mide entre esquinas.
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
