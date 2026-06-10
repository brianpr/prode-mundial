# Prode Mundial 2026 — Documentación técnica

App de una sola página (HTML + CSS + JS vanilla, sin dependencias) para completar pronósticos de marcador exacto del Mundial FIFA 2026. Soporta prodes ilimitados, asistencia de IA por copiar/pegar, y persistencia local con fallback.

Archivo: `prode-mundial-2026.html` (autocontenido). Fuente de datos: fixture oficial (104 partidos: 72 de fase de grupos en 12 grupos A–L + 32 de eliminatorias). Guia de publicacion: `docs/DEPLOY.md`.

---

## Modelo de datos

### `MATCHES` (const)

Array embebido con los 104 partidos. Cada objeto:

| Campo | Tipo | Descripción |
|---|---|---|
| `id` | number | ID oficial único del partido. Clave primaria en todo el código. |
| `stage` | `"group"` \| `"ko"` | Fase de grupos o eliminatoria. |
| `group` | string | Letra `A`–`L` (grupos) o nombre de fase: `16avos`, `Octavos`, `Cuartos`, `Semifinal`, `Tercer puesto`, `Final` (eliminatorias). |
| `md` | number\|null | Jornada (1–3) en grupos; `null` en eliminatorias. |
| `date`, `time` | string | Fecha `YYYY-MM-DD` y hora. |
| `h`, `a` | string | Nombre del equipo local/visitante en español (display). |
| `hf`, `af` | string | Emoji bandera local/visitante. |
| `hen`, `aen` | string | Nombre en inglés (se usa en los prompts de IA, menos ambiguo). |
| `label` | string | Solo en `ko`: etiqueta del cruce (ej. `16avos #1`). |
| `city`, `stadium` | string | Sede. |

### `store` (estado en memoria)

Objeto raíz que se persiste. Estructura:

```js
{
  active: 0,                  // índice del prode activo
  prodes: [
    {
      id: "p1",               // identificador estable para comparaciones
      name: "Prode 1",
      compareIds: ["p2"],     // otros prodes visibles como referencia
      compareVisible: true,   // muestra/oculta resultados comparativos
      preds: { "104456": { hs:2, as:0 }, ... }
    },
    ...
  ]
}
```

`preds` indexa por `id` de partido. En grupos guarda `{hs, as}` (goles local/visitante). En eliminatorias agrega `{ht, at}` (nombres de los equipos clasificados, que el usuario tipea).

`compareIds` guarda relaciones por `id` de prode, no por índice. Esto permite borrar o cambiar de prode activo sin romper las comparaciones configuradas.

### Constantes de configuración

- `PHASES` — orden de las fases eliminatorias para la navegación.
- `PALETTE` — colores por prode (se cicla con módulo).
- `LS_KEY` — clave de localStorage (`prodeMundial2026_v2`).
- `MARK` — prefijo del espejo en `window.name`.

---

## Capa de persistencia

El sandbox del artifact puede aislar `localStorage` (origen opaco), por lo que el guardado no se asume: se verifica y se replica.

### `restore()`

Intenta recuperar el estado, en orden de prioridad:

1. `localStorage[LS_KEY]` — parsea y valida que tenga `prodes` no vacío.
2. `window.name` (si arranca con `MARK`) — fallback que sobrevive recargas del iframe aunque localStorage esté bloqueado.
3. Migración desde `prodeMundial2026_v1` (formato viejo: array fijo de 2 prodes) → lo convierte al formato nuevo.

Devuelve el `store` o `null` si no hay nada.

### `load()`

Envuelve `restore()`. Si devuelve `null`, retorna el estado por defecto (Prode 1 y Prode 2 vacíos).

### `save()`

Serializa `store` y lo escribe en **dos** destinos:

- `localStorage[LS_KEY]` con **readback de verificación** (lee de vuelta y compara). Si coincide, marca `storageOK = true`.
- `window.name = MARK + json` (siempre, dentro de try/catch).

Luego llama a `updateSaveBadge()` y `refreshProgress()`. Es el único punto de escritura: toda mutación del estado termina llamando `save()`.

### `updateSaveBadge()`

Pinta el indicador `#savebadge`:

- `storageOK === true` → "💾 Guardado automático" (verde).
- `storageOK === false` → advertencia ámbar recomendando Copia de seguridad.

### Helpers de acceso

- `P()` — devuelve el `preds` del prode activo (`store.prodes[prode].preds`).
- `rec(id)` — devuelve (creando si hace falta) el registro `preds[id]` para mutarlo.
- `ensureProdeMeta()` — migra prodes viejos agregando `id`, `compareIds` y `compareVisible`, y limpia referencias a comparativos inexistentes.

> `prode` (variable global) es el índice activo; `curView` es la vista actual (letra de grupo o nombre de fase).

---

## Gestión de prodes (CRUD)

### `buildProdeBar()`

Renderiza los chips de la barra superior, uno por prode, con su color (`colorOf`) y nombre. Click selecciona (`setProde`), doble click renombra (`renameProde`). Agrega al final el botón `＋ Nuevo` (`newProde`).

### `setProde(i)`

Cambia el prode activo, guarda, reaplica el color de acento (`applyAccent`) y re-renderiza.

### `newProde()`

Pide nombre con `uiPrompt`, crea un prode vacío, lo deja activo.

### `cloneProde()`

Pide nombre y crea un **clon nuevo** con copia profunda (`JSON.parse(JSON.stringify(...))`) de los `preds` del prode activo. No pisa otros prodes.

### `renameProde()`

Cambia el `name` del prode activo vía `uiPrompt`.

### `deleteProde()`

Borra el prode activo (con `uiConfirm`). Bloquea si queda uno solo. Reajusta el índice activo.

También quita el `id` borrado de los `compareIds` de los prodes restantes para que no queden comparaciones colgantes.

### `colorOf(i)` / `applyAccent()`

`colorOf` mapea índice → color de `PALETTE`. `applyAccent` setea las variables CSS `--accent` y `--c` con el color del prode activo (tematiza foco de inputs, badges, barra de progreso, etc.).

---

## Comparaciones entre prodes

Cada prode puede marcar otros prodes como comparativos desde la fila `Comparar:` del header. Los chips activan o desactivan relaciones en `compareIds`; el botón `Ocultar resultados` / `Mostrar resultados` cambia `compareVisible` para el prode activo.

### Render de comparaciones

`activeComparisons()` resuelve los prodes comparativos válidos del prode activo. `comparisonRows(m)` agrega, debajo de cada partido, una cápsula por comparativo con:

- Nombre del prode comparativo.
- Marcador `hs:as`, o `Sin dato` si falta.
- Badge contra el resultado del prode activo:
  - `Exacto` — mismo marcador.
  - `Mismo signo` — mismo ganador o empate.
  - `Diferente` — ambos cargados, pero no coinciden.
  - `Sin base` — el prode activo no tiene marcador cargado.

En eliminatorias también muestra `ht`/`at` del comparativo si existen, pero el match se calcula solo con `hs/as`.

### Acciones rápidas

Debajo de cada partido comparado aparece `Sugerir N:M` cuando hay al menos un marcador en los comparativos. `suggestedScoreFor(m)` agrupa los marcadores completos por `hs:as` y elige el más repetido; si no hay mayoría, usa el primer marcador completo según el orden de prodes. `applySuggestedScore(id)` copia ese marcador al prode activo.

Cada cápsula comparativa incluye `Copiar`, que aplica exactamente el marcador de ese prode al partido actual. En eliminatorias también copia `ht`/`at` si el comparativo los tiene cargados.

---

## Asistencia de IA (copiar / pegar)

Como `window.cowork.askClaude` no está disponible en este runtime, la IA se integra por intercambio manual de texto: la app genera un prompt, el usuario lo lleva a Claude Code (o cualquier IA), y pega de vuelta el JSON.

### `matchLabel(m)`

Texto legible de un partido. En grupos: `local vs visitante` (nombres en inglés). En eliminatorias: usa los equipos ya tipeados (`ht`/`at`).

### `collectForPrompt(scope)`

Resuelve qué partidos entran en el prompt según el `scope`:

- `"all"` → todos los de fase de grupos.
- letra (`"A"`–`"L"`) → ese grupo.
- `"m" + id` → un único partido (grupo o eliminatoria).

### `buildPrompt(scope)`

Arma el texto del prompt: instrucción (predecir marcador exacto, considerar ranking FIFA/plantel/forma), formato de salida exigido (`[{"id":N,"hs":..,"as":..}]`, solo JSON) y el listado `id ... : local vs visitante`.

### `openPrompt(scope)`

Abre el modal de texto (`#modal`) en modo lectura con el prompt generado y un botón **Copiar**. Disparado por los botones 🤖 (torneo, grupo, o partido individual).

### `openPaste()`

Abre el mismo modal en modo edición para pegar la respuesta de la IA. Botón **Aplicar** llama `applyPaste`.

### `copyTA()`

Copia el contenido del textarea al portapapeles (`navigator.clipboard` con fallback a `execCommand('copy')`).

### `parseJSON(s)`

Extrae JSON de una respuesta "sucia": quita fences ```` ```json ````, y recorta entre el primer `{`/`[` y el último `}`/`]`. Tolera texto extra alrededor.

### `applyPaste()`

Parsea lo pegado y aplica los marcadores al prode activo. Acepta dos formas:

- Array: `[{id, hs, as, ht?, at?}]`.
- Objeto/mapa: `{ "104456": {hs, as}, ... }` (se normaliza a array).

Por cada item con `id` que exista en `MATCHES`, escribe `hs`/`as` (y `ht`/`at` si vienen), sanea a entero ≥ 0, cuenta los aplicados, guarda, re-renderiza y cierra el modal con un toast del total.

### `closeModal()`

Oculta el modal de texto.

---

## Copia de seguridad / Restaurar

### `normalizeBackupProdes(data)`

Normaliza archivos de respaldo antes de restaurar. Acepta tanto el formato completo nuevo (`{active, prodes:[...]}`) como un array suelto de prodes o predicciones. Devuelve prodes con `id`, `name`, `preds`, `compareIds` y `compareVisible` saneados.

### `normalizeSingleProde(data)`

Normaliza un archivo para importación individual. Si el JSON trae varios prodes, toma el primero. También acepta un objeto con `preds` o un mapa directo de predicciones.

### `backupJSON()`

Serializa `store` completo (todos los prodes y el índice activo) y dispara la descarga de `copia-seguridad-prode-mundial-2026_FECHA.json` vía un `<a download>` con un `Blob`. Es el respaldo confiable cuando el guardado automático no está disponible.

### `restoreBackup(input)`

Lee el archivo elegido (`FileReader`), parsea y normaliza los prodes. Luego, vía `uiConfirm`, restaura reemplazando todos los prodes actuales por los del archivo. Si el archivo trae `active`, intenta conservar ese prode activo dentro de los límites válidos.

Estas acciones están separadas visualmente como herramientas **Globales** en el header.

### `exportJSON()`

Exporta solo el prode activo, en un JSON compatible con `importJSON(input)` y también restaurable como copia de un único prode.

### `importJSON(input)`

Importa un solo prode desde JSON. Si el archivo tiene más de uno, usa el primero. Luego pregunta si reemplazar el prode activo o agregarlo como un prode nuevo.

---

## Render

### `render()`

Orquestador. Reconstruye la barra de prodes, marca el chip de navegación activo, y muestra grupos o eliminatorias según si `curView` es una letra (grupo) o una fase. Llama `refreshProgress()`.

### `buildNav()` / `chip(v, label)`

Construyen los chips de navegación: 12 grupos + las fases de `PHASES`. Cada chip cambia `curView` y re-renderiza.

### `refreshProgress()`

Cuenta partidos cargados (con `hs` y `as` no vacíos) del prode activo, actualiza la barra `#prog` y el texto, y marca con clase `done` los chips de navegación cuyos partidos están todos completos. `matchesOf(v)` devuelve los partidos de una vista.

### `teamCell(side, m)` / `scoreCell(m)`

Generan el HTML de un equipo (bandera + nombre, alineado según local/visitante) y el par de inputs numéricos de marcador (con `onchange → setScore`).

### `renderGroup(g)`

Renderiza la tarjeta de un grupo: header con botón 🤖 (`openPrompt(g)`), las 6 filas de partidos (`matchRow`), la tabla de posiciones (`standingsTable`) y la leyenda.

### `matchRow(m)`

Fila de un partido de grupo: celdas de equipos, marcador, botón 🤖 individual y metadata (jornada, fecha, estadio).

### `renderKO(ph)` / `koRow(m)`

Renderizan una fase eliminatoria. Cada fila tiene inputs de texto para tipear los equipos clasificados (`ht`/`at`), inputs de marcador, y botón 🤖. Incluye un tip de uso.

### `fmtDate(d)`

Formatea `YYYY-MM-DD` → `DD/MM`.

### `setScore(id, k, v)`

Handler de todos los inputs (marcador y nombres de equipos KO). Escribe `rec(id)[k] = v`, guarda, y si es partido de grupo re-renderiza el grupo para actualizar la tabla de posiciones en vivo.

---

## Tabla de posiciones

### `standingsTable(g, ms)`

Calcula y renderiza la tabla del grupo a partir de los marcadores cargados:

- Inicializa cada equipo con PJ/G/E/P/GF/GC/Pts en 0.
- Por cada partido con marcador completo, suma estadísticas: 3 puntos por victoria, 1 por empate, goles a favor/en contra.
- Ordena por: **Pts → diferencia de gol → goles a favor → nombre**.
- Resalta los **2 primeros** (clase `q`, clasificados directos).

> Nota: la regla real FIFA usa más criterios de desempate y los "mejores terceros"; la leyenda lo aclara. La tabla es una guía visual, no el cómputo oficial de clasificación.

---

## Modales de UI propios

Reemplazan `prompt()`/`confirm()` nativos, que el sandbox bloquea.

### `uiPrompt(title, label, def, cb)`

Modal con un input de texto. Construye botones Cancelar/Aceptar dinámicamente. Enter confirma, Esc cancela. Llama `cb(valor)` al aceptar.

### `uiConfirm(msg, onYes, onNo, yesLabel, noLabel)`

Modal de confirmación con dos acciones configurables (ej. Reemplazar/Agregar, Borrar/Cancelar). La ✕ aborta sin ejecutar ninguna.

### `iAbort()`

Cierra el modal de entrada (`#imodal`).

---

## Inicialización

Al final del script:

1. `prode` se fija al `store.active` válido.
2. `applyAccent()` → color del prode activo.
3. `buildNav()` → chips de navegación.
4. `render()` → primer pintado.
5. `save()` + `updateSaveBadge()` → escribe una vez y muestra el estado real de persistencia.
6. Listeners de click-fuera para cerrar ambos modales.

---

## Utilidades varias

- `toast(m)` — notificación efímera (2,6 s) en `#toast`.
- `escapeHtml(s)` — escapa `& < > "` para inyectar texto en plantillas seguras.

---

## Flujo típico de uso

1. Cargar marcadores tocando los inputs (la tabla de cada grupo se recalcula sola).
2. Para sugerencias: 🤖 → **Copiar** → pegar en Claude Code → traer el JSON → **📥 Pegar respuesta IA**.
3. Crear variantes con **＋ Nuevo** o **⧉ Duplicar** (clon independiente).
4. Si el badge está en ámbar, **⬇ Copia de seguridad** para respaldar todo; **⬆ Restaurar todo** para recuperar o mover entre dispositivos.
