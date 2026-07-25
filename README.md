# 🏓 Ping-Pong Teleco Huelva

Aplicación web para gestionar la liga interna de ping-pong de un grupo de amigos: calendario, registro de resultados, clasificación en vivo con desempates, playoffs y estadísticas. Pensada para usarse desde el móvil, sin registro ni instalación.

**🔗 Web en producción:** https://pcresp0.github.io/pingpong-teleco-huelva/

---

## Índice

- [Funcionalidad](#funcionalidad)
- [Arquitectura](#arquitectura)
- [Estructura del repositorio](#estructura-del-repositorio)
- [Modelo de datos](#modelo-de-datos)
- [Lógica de la liga](#lógica-de-la-liga)
- [Persistencia y sincronización](#persistencia-y-sincronización)
- [Autenticación de escritura](#autenticación-de-escritura)
- [Interfaz y diseño](#interfaz-y-diseño)
- [Build y despliegue](#build-y-despliegue)
- [Decisiones técnicas](#decisiones-técnicas)
- [Limitaciones conocidas](#limitaciones-conocidas)

---

## Funcionalidad

| Área | Descripción |
|---|---|
| **Liga** | Round-robin a una vuelta entre 7 jugadores fijos: 21 partidos repartidos en 7 jornadas. |
| **Resultados** | Registro con validación de marcador según reglas reales de tenis de mesa. Editables y borrables por el administrador. |
| **Clasificación** | Recalculada en cada render a partir de los partidos jugados. Incluye PJ, PG, PP, PF, PC, diferencia, ratio y puntos. |
| **Desempates** | Enfrentamiento directo primero; ratio puntos a favor / en contra como segundo criterio. |
| **Playoffs** | Se desbloquean al completar la liga regular. Semifinales 1º–4º y 2º–3º, final y partido por el 3.er puesto. |
| **Estadísticas** | Totales, medias, mejor racha, máximo anotador, ranking de regularidad (desviación típica) y victorias más abultadas. |
| **Curiosidades** | Datos generados dinámicamente a partir de los resultados reales. |
| **Sobre la web** | Sección dentro de la propia app que explica cómo está construida. |

---

## Arquitectura

Es una **SPA estática**: no hay backend propio, ni base de datos, ni proceso de servidor. El navegador descarga tres archivos (`index.html`, `bundle.js`, `styles.css`) y a partir de ahí toda la lógica —cálculo de clasificación, validaciones, desempates, generación de playoffs— se ejecuta en el cliente.

```
┌──────────────────────────────┐
│  Navegador (móvil)           │
│                              │
│   React 18 SPA               │
│   ├─ estado en memoria       │
│   ├─ cálculo de clasificación│
│   ├─ validación de marcadores│
│   └─ hash SHA-256 (WebCrypto)│
│              │               │
│              ▼               │
│   localStorage (caché local) │
└──────────────┼───────────────┘
               │ fetch (GET/PUT)
               ▼
      ┌────────────────────┐
      │  jsonblob.com      │  ← estado compartido del grupo
      │  (blob JSON)       │
      └────────────────────┘

      ┌────────────────────┐
      │  GitHub Pages      │  ← sirve los archivos estáticos
      └────────────────────┘
```

**Stack:**

| Herramienta | Versión | Papel |
|---|---|---|
| React | 18 | Capa de UI y gestión de estado (hooks: `useState`, `useEffect`, `useCallback`, `useRef`) |
| esbuild | — | Bundler: compila JSX y empaqueta React + app en un único `bundle.js` minificado |
| Tailwind CSS | 3 | Utilidades de estilo, compiladas y purgadas en build time |
| Web Crypto API | nativa | Hash SHA-256 para la contraseña de administrador |
| jsonBlob | — | Almacén JSON público y gratuito para el estado compartido |
| GitHub Pages | — | Hosting estático con despliegue automático desde `main` |

---

## Estructura del repositorio

```
pingpong-teleco-huelva/
├── index.html      # Documento base (~1 KB)
├── bundle.js       # App compilada: React + lógica + vistas (~190 KB)
├── styles.css      # Tailwind purgado (~14 KB)
└── README.md
```

**`index.html`** — Mínimo deliberadamente. Contiene:
- Metadatos para PWA-like en iOS (`apple-mobile-web-app-capable`, `viewport-fit=cover` para respetar el notch).
- Favicon en SVG inline vía data-URI (sin petición extra).
- Precarga de fuentes desde Google Fonts (Bebas Neue para display, Inter para texto, JetBrains Mono para datos).
- CSS crítico inline: animaciones (`ppth-bounce`, `ppth-fadein`), textura de puntos y un placeholder `#root:empty::before` que muestra "Cargando…" antes de que React monte.
- Un único `<script src="./bundle.js">`.

**`bundle.js`** — Salida de esbuild. Incluye React, ReactDOM y todo el código de la app en un solo archivo, minificado y con `process.env.NODE_ENV` fijado a `"production"` para eliminar los warnings de desarrollo de React.

**`styles.css`** — Salida de la CLI de Tailwind con `--minify`, escaneando el fuente JSX para incluir solo las clases realmente usadas.

> **Nota:** el fuente JSX no está versionado en este repositorio; solo el artefacto compilado. Ver [Decisiones técnicas](#decisiones-técnicas).

---

## Modelo de datos

Todo el estado de la liga es un único objeto JSON serializable:

```jsonc
{
  "players": [
    { "id": "p1", "name": "Paloma" },
    { "id": "p2", "name": "Patricia" }
    // … 7 en total, ids estables p1–p7
  ],
  "matches": [
    {
      "id": "m_ms0poubz_l167p",  // uid: prefijo + timestamp base36 + random
      "leg": 1,                  // siempre 1 (liga a una vuelta)
      "round": 1,                // jornada 1–7
      "homeId": "p2",
      "awayId": "p7",
      "homeScore": null,         // null mientras no se juega
      "awayScore": null,
      "played": false,
      "playedAt": null           // timestamp: se usa para calcular rachas
    }
    // … 21 partidos
  ],
  "playoffs": null               // o { seeds, semi1, semi2, final, third }
}
```

Los **ids de jugador son estables** (`p1`…`p7`) y están precalculados en el bundle, de modo que el calendario viene ya generado por defecto: nadie tiene que pulsar "generar calendario" la primera vez que abre la app.

El campo `playedAt` no es decorativo: es lo que permite ordenar cronológicamente los partidos de cada jugador para calcular la **mejor racha de victorias consecutivas**.

---

## Lógica de la liga

### Generación del calendario (round-robin)

Se usa el **algoritmo del círculo** (círculo de Berger). Con número impar de jugadores (7) se añade un `null` como "bye", de modo que en cada jornada uno descansa:

```js
function generateRoundRobinRounds(ids) {
  let arr = [...ids];
  if (arr.length % 2 !== 0) arr.push(null);   // bye
  const n = arr.length;
  const rounds = [];
  for (let r = 0; r < n - 1; r++) {
    const roundPairs = [];
    for (let i = 0; i < n / 2; i++) {
      const a = arr[i], b = arr[n - 1 - i];
      if (a !== null && b !== null) {
        // se alterna local/visitante por jornada para repartir el orden
        roundPairs.push(r % 2 === 0 ? [a, b] : [b, a]);
      }
    }
    rounds.push(roundPairs);
    const fixed = arr[0];                      // el primero queda fijo
    const rest = arr.slice(1);
    rest.unshift(rest.pop());                  // el resto rota
    arr = [fixed, ...rest];
  }
  return rounds;
}
```

Resultado: 7 jornadas × 3 partidos = **21 partidos**, cada jugador contra los otros 6 exactamente una vez.

### Validación de marcadores

Aplica las reglas reales de tenis de mesa, no una aproximación:

```js
function isValidScore(a, b) {
  const x = Number(a), y = Number(b);
  if (!Number.isInteger(x) || !Number.isInteger(y)) return false;
  if (x < 0 || y < 0 || x === y) return false;        // no hay empate posible
  const max = Math.max(x, y), min = Math.min(x, y);
  if (max < 11) return false;                          // nadie ha llegado a 11
  if (max === 11) return min <= 9;                     // 11-10 es imposible: sería deuce
  return min === max - 2 && min >= 10;                 // deuce: 12-10, 13-11, 14-12…
}
```

El matiz importante: **el partido termina en el instante en que alguien llega a 11 con 2 de ventaja**. Por eso `12-8` es inválido (habría acabado en 11-8) y `11-10` también (a 10-10 se entra en deuce). Solo se puede superar 11 si hubo empate a 10, y entonces la diferencia debe ser exactamente 2.

El botón de guardar permanece deshabilitado mientras el marcador no sea válido, con un texto de ayuda que explica la regla.

### Clasificación y desempates

2 puntos por victoria, 0 por derrota. El orden se resuelve en cascada:

1. **Puntos de liga.**
2. **Enfrentamiento directo** entre los jugadores empatados.
3. **Ratio** puntos a favor / puntos en contra (no la diferencia: el ratio penaliza más las palizas encajadas).
4. **Puntos a favor** como último recurso.

```js
function headToHead(x, y) {
  const meetings = regular.filter(m =>
    (m.homeId === x.id && m.awayId === y.id) ||
    (m.homeId === y.id && m.awayId === x.id)
  );
  if (meetings.length === 0) return 0;
  let xWins = 0, yWins = 0;
  meetings.forEach(m => {
    const winnerId = m.homeScore > m.awayScore ? m.homeId : m.awayId;
    if (winnerId === x.id) xWins++; else if (winnerId === y.id) yWins++;
  });
  return yWins - xWins;   // positivo ⇒ y va por delante
}
```

El ratio maneja el caso límite de división por cero: si un jugador aún no ha encajado ningún punto, se le asigna `Infinity` si ha anotado alguno, y `0` si no ha jugado.

### Playoffs

Se habilitan cuando los 21 partidos están jugados. Se toman los 4 primeros de la clasificación y se cruzan **1º vs 4º** y **2º vs 3º**. Al resolverse ambas semifinales, la app genera la final (ganadores) y el partido por el 3.er puesto (perdedores) en una sola operación.

### Ranking de regularidad

Mide la **desviación típica de la diferencia de puntos** de cada jugador en sus partidos. Un valor bajo significa resultados parejos partido a partido; uno alto, altibajos (palizas y palizones). Solo se incluyen jugadores con 2 o más partidos, porque con uno solo la desviación es trivialmente 0.

```js
function stdDev(arr) {
  const n = arr.length;
  const mean = arr.reduce((s, x) => s + x, 0) / n;
  const variance = arr.reduce((s, x) => s + (x - mean) ** 2, 0) / n;
  return Math.sqrt(variance);
}
```

---

## Persistencia y sincronización

No hay backend, pero el grupo necesita **un estado compartido**: si Pablo apunta un resultado desde su móvil, Rocío tiene que verlo desde el suyo.

La solución es **jsonblob.com**, un servicio gratuito que permite crear, leer y actualizar documentos JSON públicos vía REST sin autenticación:

```js
const BLOB_API = "https://jsonblob.com/api/jsonBlob";

// POST → crea el blob; el id viene en la cabecera Location
// GET  /{id} → lee el estado actual
// PUT  /{id} → sobrescribe el estado
```

**Flujo de arranque (`init`)**:

1. Se busca el id del blob en el query string (`?liga=ID`).
2. Si no está ahí, se busca en `localStorage` (`ppth_blob_id`).
3. Si se encuentra por cualquiera de las dos vías → se descarga el estado remoto, se cachea en `localStorage` y se fija el id en la URL con `history.replaceState` (así el enlace queda listo para compartir).
4. Si no hay id en ningún sitio → se crea un blob nuevo con el estado por defecto (7 jugadores + 21 partidos vacíos).
5. Si la red falla en cualquier punto → se entra en **modo local**: se carga la última copia de `localStorage` y se muestra un aviso visible de que los cambios no se están sincronizando.

**Flujo de escritura (`persist`)**: se actualiza el estado de React, se escribe en `localStorage` como caché inmediata, y se lanza el `PUT` al blob. El icono de recargar de la cabecera gira mientras la petición está en vuelo.

**Lectura manual**: el botón ↻ de la cabecera fuerza un `GET` para traer los cambios que hayan hecho otros.

La sincronización es **last-write-wins**, sin resolución de conflictos ni polling: no hay push en tiempo real, así que quien tenga la app abierta debe recargar para ver cambios ajenos. Para el volumen real de uso (7 personas, un partido cada varios minutos) es más que suficiente.

---

## Autenticación de escritura

Toda operación que modifica datos —guardar resultado, editarlo, borrarlo, generar playoffs, reiniciar la liga— pasa por un mismo *gate*:

```js
const pendingActionRef = useRef(null);

function requireAuth(action) {
  if (authed) { action(); return; }
  pendingActionRef.current = action;   // se aparca la acción
  setPwOpen(true);                     // se abre el modal
}
```

La acción pendiente se guarda en un `useRef` (no en estado: no debe provocar re-render) y se ejecuta tal cual si la contraseña es correcta. Esto evita duplicar lógica: cada handler solo se envuelve en `requireAuth(...)` y el modal es agnóstico de qué se está autorizando.

La verificación usa la **Web Crypto API** del navegador:

```js
const ADMIN_PASSWORD_HASH = "1b5c3adff…";   // SHA-256, 64 hex

async function sha256Hex(text) {
  const enc = new TextEncoder().encode(text);
  const buf = await crypto.subtle.digest("SHA-256", enc);
  return [...new Uint8Array(buf)].map(b => b.toString(16).padStart(2, "0")).join("");
}
```

En el repositorio **solo vive el hash**, nunca la contraseña en claro. Una vez validada, se marca `ppth_authed` en `localStorage` para no volver a pedirla en ese dispositivo; el candado 🔓 de la cabecera permite cerrar la sesión manualmente.

> ⚠️ Esto es una **barrera de conveniencia, no seguridad real**. Al ser una app puramente cliente, cualquiera con conocimientos podría escribir directamente contra el blob o parchear el JavaScript en memoria. Cumple su función —evitar ediciones accidentales o bromas— pero no protege frente a un atacante motivado. Con un backend real esto se resolvería validando en servidor.

---

## Interfaz y diseño

**Sistema de color** — Paleta cerrada definida como constante única (`C`), inspirada en una mesa de ping-pong:

| Token | Hex | Uso |
|---|---|---|
| `boardDark` | `#0E3A30` | Verde mesa: cabecera, tarjetas de partido, nav inferior |
| `boardDarker` | `#092D25` | Degradados y overlays |
| `cream` | `#FAF6EC` | Fondo general |
| `ball` | `#FF6A34` | Naranja pelota: acento principal, acciones |
| `gold` | `#E3B23C` | Oro: ganadores, playoffs, podio |
| `win` / `loss` | `#2E8B5B` / `#C24A32` | Semántica de victoria y derrota |

**Tipografía** — Tres roles diferenciados: *Bebas Neue* condensada para titulares (cabecera, marcadores grandes, nombre del campeón), *Inter* para texto e interfaz, y *JetBrains Mono* para todo lo tabular (marcadores, clasificación, ratios) para que las cifras se alineen verticalmente.

**Navegación** — Barra inferior fija estilo app nativa con las tres vistas frecuentes (Partidos, Clasificación, Playoffs) más un botón "Más" que abre un *drawer* lateral con el resto. La pestaña de Playoffs muestra un indicador cuando aún está bloqueada.

**Detalles de UX móvil**:
- `env(safe-area-inset-top/bottom)` en cabecera y nav para respetar notch y barra de gestos.
- Feedback táctil con `active:scale-95` en todos los pulsables.
- Inputs de marcador con `inputMode="numeric"` para que salga el teclado numérico.
- Nombres largos truncados con `truncate` + `min-w-0` en flex, para que "Miguel Ángel" nunca rompa el layout.
- Tablas con `overflow-x-auto`; donde una tabla no cabría bien (victorias más abultadas) se usa lista en lugar de tabla.
- Animaciones envueltas en `@media (prefers-reduced-motion: reduce)`.
- Confirmación en dos pasos para toda acción destructiva.

---

## Build y despliegue

El proyecto se compila fuera del repositorio y se sube el resultado. El pipeline es:

```bash
# 1. Bundle de la app
npx esbuild src/main.jsx \
  --bundle \
  --minify \
  --jsx=automatic \
  --define:process.env.NODE_ENV='"production"' \
  --outfile=dist/bundle.js \
  --loader:.jsx=jsx

# 2. CSS purgado
npx tailwindcss \
  -i ./input.css \
  -o ./dist/styles.css \
  --content "./src/**/*.jsx" \
  --minify
```

Los artefactos (`index.html`, `bundle.js`, `styles.css`) se publican en la raíz de `main` mediante la [Contents API](https://docs.github.com/rest/repos/contents) de GitHub.

**Despliegue** — GitHub Pages está configurado en modo *legacy* sirviendo desde `main` / `root`. Cada commit dispara una reconstrucción automática, sin GitHub Actions ni workflow propio: el sitio ya son archivos estáticos listos para servir.

**Verificación previa** — Antes de publicar, el bundle se somete a una prueba de humo en un DOM headless con **jsdom**, simulando `crypto.subtle` y con `fetch` deshabilitado para forzar el modo local. Se comprueba que React monta sin errores y se simulan interacciones reales (introducir un marcador inválido y comprobar que el botón queda deshabilitado, completar el flujo de contraseña, borrar un resultado y verificar el estado resultante). Esto detecta errores de runtime que un simple `esbuild` sin errores no revela.

---

## Decisiones técnicas

**Por qué se publica el bundle y no el fuente.** La primera versión cargaba React, ReactDOM y Babel Standalone desde CDN y transpilaba el JSX en el navegador. Funcionaba en escritorio pero **fallaba en móvil con datos**: ~1.5 MB de dependencias más compilación en cliente daban pantalla en blanco. Precompilar redujo la carga a ~200 KB y eliminó tanto la dependencia de CDNs de terceros como el coste de transpilar en cada visita. El repositorio contiene el artefacto que realmente se sirve.

**Por qué jsonBlob y no Firebase / Supabase.** Ambos habrían dado tiempo real y reglas de seguridad de verdad, pero exigen cuenta, proyecto, claves y configuración. Para una liga de 7 amigos durante un viaje, el coste de montarlo superaba el beneficio. jsonBlob no requiere absolutamente nada y encaja con la premisa de "sitio estático sin backend".

**Por qué el calendario va precalculado en el bundle.** Evita que el primer usuario tenga que pulsar "generar calendario" y garantiza que todos vean exactamente el mismo emparejamiento y orden de jornadas, sin depender de quién abrió la app primero.

**Por qué ratio y no diferencia de puntos como desempate.** Con partidos a 11, la diferencia acumulada premia desproporcionadamente al que juega muchos partidos ajustados. El ratio (a favor / en contra) refleja mejor la solidez relativa.

**Por qué los jugadores no son editables.** El grupo es cerrado y conocido. Bloquearlo elimina toda una clase de errores (renombrar a alguien a mitad de liga, borrar un jugador con partidos jugados y dejar referencias huérfanas) sin perder nada útil.

---

## Limitaciones conocidas

- **Sin tiempo real.** Hay que recargar (o pulsar ↻) para ver cambios de otros. No hay WebSocket ni polling.
- **Last-write-wins.** Si dos personas guardan a la vez, el último gana silenciosamente. Improbable en la práctica dado el patrón de uso.
- **La contraseña no es seguridad real.** Ver [Autenticación de escritura](#autenticación-de-escritura).
- **El blob es público.** Cualquiera con el id podría leer o escribir. No hay datos sensibles: solo nombres de pila y marcadores.
- **El estado por defecto solo aplica a ligas nuevas.** Cambiar el calendario en el código no modifica una liga ya creada; hay que reiniciarla desde la app.
- **Sin tests automatizados.** La verificación es una prueba de humo manual con jsdom en cada despliegue, no una suite.

---

Hecho con 🏓 por [Pablo Crespo](https://www.linkedin.com/in/pablocrespobellido) · [GitHub](https://github.com/pCresp0)
