# Space Invaders — instrucciones del proyecto

## Visión general

El proyecto es **un único archivo**: `index.html` (~19 KB). Contiene el HTML, el CSS y todo el JavaScript (vanilla, sin dependencias externas, sin bundler, sin gestor de paquetes, sin tests). No hay repositorio git.

Todo el código vive dentro de una IIFE `(function () { 'use strict'; ... })();` que **no exporta nada**: no hay API global ni forma de inspeccionar el estado del juego desde la consola del navegador. Para probar lógica hay que ejecutar el script en Node con stubs de `document`, `canvas.getContext()` y `requestAnimationFrame`, y manejar el bucle a mano.

## Ejecutar y verificar

- Ejecutar: abrir `index.html` directamente (`file://`) o servirlo con `python3 -m http.server 8123` y visitar `http://localhost:8123/index.html`. No hace ninguna petición de red, así que `file://` funciona igual.
- No hay build, lint ni suite de tests. Para validar la sintaxis del script embebido sin navegador:

  ```bash
  python3 -c "import re;h=open('index.html',encoding='utf-8').read();open('/tmp/g.js','w',encoding='utf-8').write(re.search(r'<script>(.*?)</script>',h,re.S).group(1))" && node --check /tmp/g.js
  ```

- Verificación funcional manual: cargar la página, pulsar `Enter`/`Espacio` y comprobar HUD, movimiento de la flota, disparos, pérdida de vidas y reinicio.

## Entorno de desarrollo

El proyecto vive en WSL (`/home/alex/projects/space-invader`), pero las sesiones de agente se ejecutan en Windows. **No hay `node` en el lado Windows**: todo lo que necesite Node debe invocarse a través de WSL, p. ej. `wsl -e bash -lc "node --check /tmp/g.js"`. Tampoco hay heredoc en PowerShell y las comillas anidadas se corrompen, así que para scripts inline conviene escribir un archivo temporal y ejecutarlo, en lugar de usar `node -e`/`python3 -c` directamente.

`.vscode/mcp.json` define un servidor MCP de Playwright (`@playwright/mcp`, v0.0.83) para verificar el juego en un navegador real. Se lanza con `command: "wsl"` por el motivo anterior y hace `cd` al proyecto para que la raíz del espacio de trabajo sea correcta; si cambia la ruta del proyecto, hay que actualizar ese `cd`.

## Arquitectura

El orden de las secciones dentro del `<script>` es: sprites → constantes → estado → teclado → utilidades → flota → actualización → dibujo → bucle. Mantener ese orden al añadir código.

- **Bucle**: `loop(now)` llama a `update(dt)` y `render()`, y se re-agenda con `requestAnimationFrame`. `dt` está limitado a `0.05` s para que un cambio de pestaña no teletransporte las entidades.
- **Máquina de estados**: la variable `state` vale `title | playing | paused | levelclear | gameover`. Solo el bloque `playing` de `update()` procesa la partida; estrellas y partículas se actualizan siempre. `stateTimer` controla las pausas temporales (p. ej. el cartel de nivel superado).
- **Entidades**: arrays planos de objetos `{ x, y, w, h, ... }` en el ámbito de la IIFE: `invaders`, `playerShots`, `invaderShots`, `particles`, `stars`. Los invasores **nunca se eliminan del array**: se marca `alive = false` y el resto del código filtra por esa bandera. Mantener ese patrón.
- **Colisiones**: AABB mediante `hit(a, b)`.
- **Flota**: `spawnFleet()` crea la rejilla 5×11; la flota se mueve como un bloque usando `fleet.dir`, y `dropFleet()` la baja una fila y cambia la dirección. `fleetSpeed(aliveCount)` acelera según los invasores vivos y el nivel; `invaderFire()` dispara desde el invasor más bajo de una columna aleatoria.
- **Dibujo**: es inmediato, sin caché — `render()` redibuja todo cada frame (fondo, estrellas, suelo, invasores, disparos, nave, partículas) y `drawOverlays()` pinta encima los carteles de título/pausa/nivel/game over. `shake` es la magnitud del temblor de pantalla y se desvanece en `update()`.
- **HUD**: es DOM (no canvas). `updateHUD()` se llama tras cualquier cambio de `score`, `level` o `lives`.

## Convenciones

- **Estilo ES5 deliberado**: `var`, declaraciones de función, sin `let`/`const`, arrow functions, clases ni template literals. Mantenerlo para no mezclar estilos.
- **Sprites**: arrays de strings donde `'1'` es un píxel (`SQUID`, `CRAB`, `OCTOPUS`, `SHIP`). `drawPattern(pat, x, y, cell, color)` los pinta con `fillRect`. El `Math.round()` de las coordenadas es intencional: evita antialiasing y mantiene el look retro nítido. Hacer lo mismo en cualquier dibujo nuevo de píxeles (incluidos los `fillRect` de disparos y partículas).
- **Ajuste del juego**: todos los parámetros están en constantes al inicio del script (`CELL`, `SHIP_CELL`, `COLS`, `ROWS`, `COL_GAP`, `ROW_GAP`, `COL_W`, `ROW_H`, `FLEET_TOP`, `PAD`, `DROP`) o en funciones dedicadas (`fleetSpeed()`, `invaderShotSpeed()`, temporizador de disparo dentro de `update()`). No repartir números mágicos por el código.
- **Campo cuadrado**: el canvas es 800×800 y `W`/`H` se derivan de `canvas.width`/`canvas.height`. El CSS lo mantiene cuadrado y a escala con `--size: min(100%, 800px, 80vh)` y `aspect-ratio: 1 / 1`. Si se cambia el tamaño del canvas, hay que ajustar también `--size` y el layout vertical (`FLEET_TOP`, la posición de la nave en `resetPlayer()`, los offsets de `panel()`/`text()` en `drawOverlays()`).
- **Colores**: centralizados en el objeto `COLORS` (y `--fg`/`--line` en el CSS). No introducir literales de color sueltos.
- **Entrada**: el teclado se mapea por `e.code` (independiente de la distribución) en `keyName()`; el estado vive en el mapa `pressed` (`left`/`right`/`fire`/`pause`/`start`). Los botones táctiles usan `bindHold()` y escriben en ese mismo mapa. Para añadir un control, extender el mapa y `keyName()`, no añadir listeners paralelos.
- **Textos de la interfaz en español** (también los `aria-label`); identificadores y nombres de variables en inglés.
- **`localStorage`** siempre dentro de `try/catch`: bajo `file://` puede lanzar. Clave usada: `si-hi`.
- **Balance**: solo un disparo del jugador en vuelo (`playerShots.length < 1`), enfriamiento de 0.24 s; 3 vidas; velocidad de bala del jugador 720 px/s. Son decisiones deliberadas para acercarse al arcade original.
- **Teclas**: `←`/`A` y `→`/`D` mover, `Espacio`/`↑`/`W` disparar y arrancar, `P` pausa, `Enter` arrancar/reanudar. `e.repeat` se ignora para que mantener la tecla no reinicie la partida, y `window.blur` limpia `pressed`.
