# CLAUDE.md — Prode Mundial 2026

Guía para desarrollar/mantener esta app. Leé esto antes de tocar el código.

## Qué es

App de una sola página para completar pronósticos de **marcador exacto** del Mundial FIFA 2026. Prodes ilimitados, tabla de posiciones en vivo, asistencia de IA por copiar/pegar, persistencia local.

- **Entregable:** `prode-mundial-2026.html` — un único archivo autocontenido (HTML + CSS + JS vanilla). **Sin dependencias, sin build, sin CDN.** Todo inline.
- **Documentación funcional por función:** `docs/DOCUMENTACION.md` (mantenela sincronizada al cambiar funciones).
- **Datos:** fixture oficial, 104 partidos (72 grupos en A–L + 32 eliminatorias). Origen: `world-cup_2026.json`. Letras de grupo verificadas contra el fixture público de ProdeLibre.
- **Archivos de datos auxiliares:** `sugerencia-claude.json` (un prode base importable).

## Dónde corre y por qué importa

La app vive como **artifact de Cowork**, embebido en un **iframe sandboxeado**. Ese sandbox impone restricciones que ya nos rompieron features. **Tenelas siempre presentes:**

| Bloqueado en el sandbox | No uses | Usá en su lugar |
|---|---|---|
| `prompt()` / `confirm()` nativos | diálogos del navegador | `uiPrompt()` / `uiConfirm()` (modal propio `#imodal`) |
| `localStorage` puede estar aislado (origen opaco) | asumir que persiste | capa `save()` con readback + espejo en `window.name` |
| `window.cowork.askClaude` no inyectado | sugerencias en vivo | flujo IA copiar/pegar (`buildPrompt`/`applyPaste`) |

Regla general: **no asumas APIs del navegador**. Verificá o degradá con fallback visible al usuario (ej. el badge `#savebadge`).

El mismo HTML corre **mejor** fuera del sandbox (doble-click al `.html`): ahí `localStorage` persiste normal. No rompas esa compatibilidad.

## Arquitectura (mapa mental)

- Estado global: `store = { active, prodes:[{name, preds}] }`. `preds` indexa por **`id` oficial del partido** → `{hs, as}` (grupos) y además `{ht, at}` (nombres en eliminatorias).
- `MATCHES` (const embebida) = los 104 partidos. **`id` es la clave primaria en todo el código.**
- Variables globales: `prode` (índice activo), `curView` (grupo o fase visible).
- **Único punto de escritura: `save()`.** Toda mutación de estado debe terminar llamando `save()`. No escribas `localStorage`/`window.name` desde otro lado.
- Persistencia: `load()` → `restore()` (localStorage → window.name → migración v1 → default). `save()` escribe en ambos destinos y actualiza `storageOK` + badge.
- Render: `render()` orquesta → `buildProdeBar`, `buildNav`, y `renderGroup`/`renderKO`. La tabla de posiciones (`standingsTable`) se recalcula desde `preds` en cada render del grupo.
- IA: `openPrompt(scope)` arma texto (`buildPrompt`) para pegar afuera; `openPaste()`+`applyPaste()` ingieren el JSON devuelto (`parseJSON` tolera markdown/ruido).

Para el detalle función por función, ver `docs/DOCUMENTACION.md`.

## Convenciones

- Español en UI y comentarios. Nombres en inglés (`hen`/`aen`) solo para los prompts de IA (menos ambiguos para el modelo).
- Acotá la salida de IA al formato `[{"id":N,"hs":..,"as":..}]`. Siempre matchear por `id` contra `MATCHES`; ignorar ids desconocidos.
- Sanear inputs de marcador a entero ≥ 0.
- Tematización por color: usar variables CSS `--accent`/`--c` (las setea `applyAccent`). No hardcodear colores de prode.
- Texto dinámico en plantillas: pasar por `escapeHtml()`.
- Notificaciones: `toast()`. Nada de `alert()`.

## Cómo editar y verificar

Workflow probado (evita un bug de desincronización entre el editor de archivos y el mount de bash):

1. **Generá/regenerá el archivo completo con un script Python** (heredoc en bash) en vez de editar por línea. Editar el HTML in-place con la tool de edición desincronizó el archivo del mount de bash al menos una vez. Si editás puntual, **releé por bash** antes de publicar.
2. **Chequeo de sintaxis JS obligatorio** antes de publicar. Extraé el `<script>` y corré `new Function(js)` en Node con stubs de `window/document/localStorage/...`. Verificá también: presencia de funciones clave, cierre `</script></body></html>`, y ausencia de `prompt(`/`confirm(`/`askClaude` nativos si corresponde.
3. **Copiá** el resultado a la carpeta del usuario (`prode-mundial-2026.html`).
4. **Publicá** con `update_artifact` (id `prode-mundial-2026`), no `create_artifact` (ya existe). Poné `update_summary` claro.

Mini-check de sintaxis (Node):

```bash
node -e '
const fs=require("fs"); const h=fs.readFileSync("prode.html","utf8");
const js=h.match(/<script>([\s\S]*)<\/script>/)[1];
const stub="var navigator={clipboard:{}},window={name:\"\"},document={addEventListener(){},getElementById:()=>({classList:{add(){},remove(){},toggle(){}},style:{},appendChild(){},select(){},focus(){}}),querySelectorAll:()=>[],createElement:()=>({classList:{},style:{setProperty(){}},appendChild(){}}),documentElement:{style:{setProperty(){}}},body:{appendChild(){}}},localStorage={getItem:()=>null,setItem(){}},setTimeout(){},clearTimeout(){};";
try{ new Function(stub+js); console.log("JS OK"); }catch(e){ console.log("ERR:",e.message); }'
```

## Gotchas (aprendidos en el camino)

- **No edites el `.html` con la tool de edición y después lo regeneres por bash sin releer**: se desincronizan y podés truncar el archivo. Elegí una vía (preferí regenerar completo por Python) y verificá por bash.
- El badge de guardado dice la **verdad** sobre persistencia. Si está en ámbar, el guardado automático no está disponible en ese runtime; la vía sólida es Exportar/Importar. No prometas auto-guardado sin chequear.
- `window.name` sobrevive el botón Recargar del artifact, **no** sobrevive cerrar/reabrir la app. Es mitigación, no solución.
- Al sumar features que pidan texto/confirmación, **siempre** `uiPrompt`/`uiConfirm`. Nunca el diálogo nativo.

## Ideas / backlog

- Sistema de puntaje (ej. 3 pts acierto exacto, 1 pt acertar ganador) para comparar prodes cuando haya resultados reales.
- Prompt de IA para eliminatorias una vez definidos los cruces.
- Desempates oficiales FIFA completos + cómputo de "mejores terceros" en la clasificación.
- Persistencia host-side real si Cowork expone una API de storage (hoy no confirmada).
