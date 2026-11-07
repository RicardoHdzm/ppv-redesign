# PPV Draft League · Rediseño (Temporada 5 · Evolution)

Documento de traspaso para llevar el rediseño al sitio real (ppvdraftleague.com), con sus archivos y su base de datos.

- **Repositorio:** https://github.com/RicardoHdzm/ppv-redesign. Se puede clonar o bajar como ZIP desde el botón *Code → Download ZIP*.
- **Demo en vivo:** https://ricardohdzm.github.io/ppv-redesign/ (GitHub Pages, rama `main`).

> El rediseño es un **prototipo estático**: todo vive en un solo `index.html` con los datos escritos dentro (copiados del sitio actual). No se conecta a ninguna base de datos. Lo que en el sitio real viene de la BD o de la sesión del usuario aquí está **simulado**. La sección 4 lista cada caso.

---

## 1. Cómo verlo en local

Cualquier servidor estático en la carpeta del repo sirve, por ejemplo:

```bash
npx serve .
```

o `python -m http.server`. Abrir `index.html` con doble clic **no** sirve del todo: el navegador bloquea exportar imágenes y el service worker con `file://`.

## 2. Archivos

| Ruta | Qué es |
|---|---|
| `index.html` | Todo el sitio: HTML, CSS y JS en un solo archivo (~330 KB). |
| `logos/*.webp`, `logos/richampions-vgc.png` | Logos de los 18 equipos. El nombre del archivo es el *slug* del equipo (`nombre-en-minusculas-con-guiones`). Las excepciones están en `LOGO_FILE`. |
| `logos/logo-ppv-white.png` | Logo de la liga en blanco (navbar, footer, imágenes para compartir, overlays). `logo-ppv.png` es el original. |
| `media/hero-web.mp4`, `media/hero-poster.jpg` | Video de fondo del hero y su imagen de respaldo. |
| `meta/` | Favicon, ícono de Apple, íconos de la app (192, 512 y maskable) e imagen para redes (`og-image.jpg`). |
| `manifest.webmanifest`, `sw.js` | App instalable (PWA). El service worker pide la página a la red primero y guarda logos, sprites y fuentes en caché. |
| `.nojekyll` | Solo para GitHub Pages. |

## 3. Cómo está armado `index.html`

- **Navegación por hash, sin recargar:** `#home`, `#standings`, `#matches`, `#h2h` (y `#h2h-<i>-<j>`), `#pronosticos`, `#playoffs`, `#teams`, `#team-<slug>`, `#draft`, `#pokedex`, `#salon` (H.O.F.), `#mi-equipo` y `#media`. La función `route()` está al final del script.
- **Cada página** es un `<main id="page-…">`. Las secciones dinámicas se pintan con funciones `render…()`: `renderStandings`, `renderMatches`, `renderProfile(ti)`, `renderH2H`, `renderPicks`, `renderDex`, `renderDraft`, `renderHof`, `renderMine`, `renderMedia`, `renderTx`, etc.
- **Equipos por índice:** casi todo referencia a un equipo por su posición en `TEAMS` (0–17). Al pasar a la BD, conviene usar un **id estable** en lugar del índice.

## 4. Datos: qué hay en el código y a qué corresponde en la BD

Todos están cerca del inicio del `<script>`. Al conectar el sitio real, cada constante debe venir de la BD o de una API.

| Constante | Contenido | Forma |
|---|---|---|
| `TEAMS` | Equipos y fase regular | `{n: nombre, c: "rojo"\|"azul" (conferencia), w, l, d: balance de KOs, h: [tono1, tono2] para el escudo de respaldo}`. Al cargar se calculan `pts = w*3`, `ab` (abreviatura, de `ABBR`), `rank` (posición en su conferencia), `q` (top 4 = playoffs) y `gr` (posición general). |
| `ABBR` | Abreviaturas de 3 letras (#INT, #RCH…) | Mismo orden que `TEAMS`. Convendría guardarlas como campo del equipo. |
| `CONF` | Conferencias | `rojo` → FireRed, `azul` → LeafGreen. Las claves internas son `rojo`/`azul`, aunque los nombres cambien cada temporada (ver sección 7). |
| `RAW_MATCHES` → `MATCHES` | Series de la fase regular | `[semana, iA, iB, "aba" (ganador de cada juego), [ids de replay de YouTube]]`. Al cargar se calculan `ga`, `gb` y la conferencia. |
| `GAMES` | Pokemon usados en cada juego | `[serie][juego][ladoA[], ladoB[]]`, en el mismo orden que `MATCHES`. Alimenta el detalle de serie, el "Pokemon más usado" y el **Pokemon de la semana**. |
| `PROFILES` | Perfil y roster de cada equipo | `[trainer, título, "puntos de draft", movs. FA, favorito, roster[]]`. Cada roster trae `[pokemon, costo, "tipo1/tipo2", habilidades, obtenido, partidas, kills, deaths]`. |
| `CHAMPIONS_ID` | Champions ID por equipo | `{índice: "id"}`. Solo Richampions tiene uno en la demo. |
| `MONS`, `MONS_FULL` | Leaderboard de KOs | `MONS`: top 8 para Home `[pokemon, kos, kd]`. `MONS_FULL`: `[pokemon, partidas, kills, deaths]`. |
| `POOL` | Pool del draft (Pokedex) | `[pokemon, costo, "tipos", id PokeAPI]`. El dueño (`OWNER`) se calcula a partir de los rosters. |
| `DRAFT_ORDER`, `DRAFT_ROUNDS` | Tablero snake del draft | Orden de columnas y picks por ronda. |
| `PLAYOFFS` | Bracket | `A`/`B` con `wc`, `qf`, `sf` y la `final`. Cada cruce trae `{a, b, w: "a"\|"b"}` o `seq` con el ganador de cada juego. `resolvePlayoffs()` avanza a los ganadores. |
| `SPR_IDS`, `BOX` | Sprites | Nombre → id de PokeAPI (incluye formas: Hisuian, Alolan, Rotom, etc.) y nombre del sprite de caja. |
| `NEXT_EVENT` | Cuenta regresiva del hero y del ticker | `{label, at: fecha ISO}`. Hoy es un placeholder ("Draft de la Temporada 5", 21 nov). |

### Lo que está simulado y hay que conectar

| Qué | En la demo | En el sitio real |
|---|---|---|
| **Sesión** | `const ME = 12` (Richampions VGC, trainer "Rick"). | El equipo del usuario logueado. Se usa en el chip de la barra, "Mi equipo", "Tu" en tablas, apodos, Cara a cara y Pronósticos. |
| **Roles** | `const ROLES = {12: ["trainer","cm"]}`. | Roles reales por cuenta. El rol `cm` (community manager) muestra el botón **CM** y la página `#media`. |
| **Apodos de Pokemon** | `localStorage` (`ppv-apodos-v1`), solo en ese navegador. | Tabla de apodos por equipo y Pokemon, editable solo por el dueño. |
| **Pronósticos** | Tus picks en `localStorage` (`ppv-pronosticos-v1`). Los de los demás trainers son **simulados** (`simPick`). | Tabla de picks por usuario y serie, que se cierran antes de cada jornada, más la tabla de aciertos real. |
| **Trades y fichajes** | Se **deducen** comparando el tablero del draft con los rosters actuales (`TX()`). No tienen fecha. | Registrar cada movimiento con fecha y tipo. Así se podrán filtrar "los de esta semana" en la página del CM. |
| **Pokemon de la semana** | El que estuvo en más juegos ganados de la semana; desempata el total de KOs. | Si la BD guarda **KOs por juego**, conviene usar eso. |
| **Power ranking** | Se **calcula** en el navegador (`powerAt(w)`): 3 por serie ganada, 0.6 por juego de diferencia, 1.5 por victoria en las últimas 3 semanas y hasta 4 por la fuerza de los rivales. | Puede quedarse calculado o guardarse por semana. |
| **Récord de temporadas pasadas (H.O.F.)** | Las temporadas 1–3 salen del título de cada trainer; la 4 sale de `PLAYOFFS`. | Tabla de campeones por temporada. |

## 5. Funciones nuevas del rediseño (por página)

- **Home:** hero con video, cuenta regresiva y la sección "Del draft al título". Debajo: tabla por conferencia, Líderes en KOs con el Pokemon de la semana, Power ranking (top 5) y "Juega y compara".
- **Standings:** vistas Por conferencia, General y **Power ranking** (con selector de semana), y el Pokemon Leaderboard ordenable.
- **Matches:** semanas, filtros y el panel de detalle de cada serie, con los juegos, los replays y la sección **Transmisión · OBS** para descargar el overlay (ver sección 6).
- **H2H / Cara a cara:** comparador de dos equipos con números, historial directo, ventaja de tipos y rosters.
- **Pronósticos:** quiniela por semana, revelar resultados y tabla de aciertos.
- **Perfil de equipo:** semana a semana, roster (con apodos), análisis de tipos, **cobertura ofensiva**, trades y fichajes, y resultados.
- **Draft:** mi draft, tablero, **trades y fichajes**, movimientos desde el draft y free agency.
- **Pokedex, Playoffs y H.O.F.:** rediseñados.
- **Media (solo rol CM):** las imágenes de la semana para redes (calendario, resultados, power ranking, Pokemon de la semana, standings, tabla general, líderes, mercado y bracket), con descarga individual o de todas.
- **En todo el sitio:** buscador global (tecla `/` o Ctrl+K), tooltips que explican cada stat, imágenes para compartir en historia o post y app instalable.

## 6. Imágenes para compartir y overlays de OBS

- **Se dibujan en el navegador con `<canvas>`.** La base común es `shareFrame()` y cada tipo de imagen está en `SHARE_KINDS`. Hay dos formatos: historia 1080×1920 y post 1080×1350.
- **Overlay de OBS:** `drawOverlay(i)` genera un PNG de 1920×1080 con el recuadro del juego transparente (`OBS_GAME = {x:160, y:120, w:1600, h:900}`). Usa márgenes de 26px, bordes de color de 4px y el roster con los renders de **Pokemon HOME**.
- **Pantalla de inicio:** `drawIntro(i)` existe, pero su botón está **oculto** por ahora.
- **CORS:** las imágenes externas (sprites de PokeAPI y HOME) se cargan con `crossOrigin="anonymous"`. Si el sitio real sirve sus propios sprites o logos, deben mandar encabezados CORS. Si no, el canvas queda "contaminado" y no se puede exportar.
- **Descarga:** usa un enlace de descarga normal y, en el celular, el menú de compartir del sistema (Web Share API).

## 7. Estilo y reglas de diseño

- **Tokens** (en `:root`):

  | Token | Valor | Uso |
  |---|---|---|
  | `--ink` | `#0B0B0C` | Fondo carbón |
  | `--ink-2` | `#151517` | Paneles |
  | `--ink-3` | `#202024` | Paneles elevados |
  | `--chalk` | `#F2F2F0` | Texto |
  | `--mute` | `#9C9CA4` | Texto secundario |
  | `--rojo` | `#FF4B2B` | FireRed |
  | `--verde` | `#7ED13A` | LeafGreen |
  | `--volt` | `#FFB62E` | Acento amarillo |

- **Fuentes** (Google Fonts):
  - **Big Shoulders Display:** títulos y números.
  - **Barlow Condensed:** etiquetas, botones y navegación.
  - **Barlow:** texto corrido.
- **Reglas:**
  - **Sin acentos** en los textos del sitio. La ñ sí se usa.
  - Fondo carbón. El **color de la conferencia** va en los detalles (franjas, bordes, barras) y se aplica a lo que pertenece a un equipo: barras de KOs, Mejor K/D, récords del H.O.F., etc.
  - El amarillo queda para lo destacado de toda la liga: campeón, MVP, botones y títulos.
  - Términos: "trainer" y "entrenadores" (no coach), "Balance de KOs" (no diferencia de Pokemon), "Series 2-0" (no barridas).
- **Conferencias por temporada.** Ya están aprobadas las siguientes. Se aplican cambiando `--rojo`, `--verde`, `--volt` y los nombres en `CONF`:

  | Temporada | Conferencia 1 | Conferencia 2 | Acento |
  |---|---|---|---|
  | Actual | FireRed `#FF4B2B` | LeafGreen `#7ED13A` | Amarillo `#FFB62E` |
  | Siguiente | HeartGold `#F2B632` | SoulSilver `#B9C4D4` | Crystal `#5CD6F0` |
  | Después | Omega Ruby `#EE3B63` | Alpha Sapphire `#3D7BFF` | Esmeralda `#2ED18A` |

  En la temporada Omega Ruby / Alpha Sapphire, las celdas "Inmune" de la tabla de tipos (`--azul`) pasan a morado `#B58CFF`.

## 8. Créditos y fuentes externas

- **Sprites:** gen 5 y renders de HOME de [PokeAPI/sprites](https://github.com/PokeAPI/sprites). Los sprites de caja son de [Autumnchi/coloured-box-sprites](https://github.com/Autumnchi/coloured-box-sprites).
- **Fuentes:** Google Fonts.
- **Aviso legal en el footer:** proyecto de fans sin fines de lucro, no afiliado a Nintendo, Game Freak ni The Pokemon Company.

## 9. Sugerencias para la integración

1. **Datos:** reemplazar cada constante de la sección 4 por una consulta a la BD, manteniendo la misma forma de datos para que las funciones `render…` sigan funcionando. También se puede adaptar cada función a los modelos reales.
2. **Ids:** cambiar los índices de `TEAMS` por ids reales en las rutas (`#team-…`, `#h2h-…`) y en las relaciones.
3. **Sesión y roles:** conectar la sesión real en lugar de `ME`, y los roles en lugar de `ROLES`. Ocultar `#media` en el servidor si el usuario no es CM.
4. **Fechas en la BD:** guardar fecha en trades y fichajes, y KOs por juego. Con eso se activan los filtros por semana y un Pokemon de la semana más preciso.
5. **Pronósticos y apodos:** moverlos de `localStorage` a tablas reales.
6. **Partir el archivo:** separar `index.html` en componentes o páginas según el stack del sitio real. Las secciones del CSS y del JS ya están marcadas con comentarios `/* ---------- NOMBRE ---------- */`.

El historial completo de cambios está en los commits del repositorio (`git log`).
