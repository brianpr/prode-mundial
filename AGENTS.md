# Repository Guidelines

## Project Structure & Module Organization

This is a static single-page app for World Cup 2026 predictions. The main deliverable is `prode-mundial-2026.html`, which contains all HTML, CSS, and vanilla JavaScript inline. Keep it self-contained: no build step, npm dependencies, or CDN assets.

- `prode-mundial-2026.html` - production app file.
- `index.html` - static redirect entrypoint for hosts that expect an index file.
- `docs/DOCUMENTACION.md` - technical documentation by feature/function; update it when behavior changes.
- `CLAUDE.md` - maintainer notes, sandbox constraints, and known pitfalls.
- `docs/DEPLOY.md` - static hosting and Vercel deployment notes.
- `world-cup_2026.json` - fixture source data.
- `sugerencia-claude.json` - importable prediction example.
- `vercel.json` - root rewrite for static deployment.

## Build, Test, and Development Commands

There is no package manager or build pipeline. Open `prode-mundial-2026.html` directly in a browser for local development.

Use this syntax check before publishing HTML changes:

```bash
node -e "const fs=require('fs');const h=fs.readFileSync('prode-mundial-2026.html','utf8');const m=h.match(/<script>([\s\S]*)<\/script>/);new Function(m[1]);console.log('JS OK')"
```

Also verify the file still ends with `</script></body></html>` and does not introduce native `prompt()` or `confirm()` calls.

## Coding Style & Naming Conventions

Use Spanish for UI text and user-facing messages. Keep existing vanilla JavaScript style: global state is centered around `store`, `prode`, `curView`, and embedded `MATCHES`. Match records by official `id`; do not rely on display names as keys.

All state mutation should end in `save()`. Do not write directly to `localStorage` or `window.name` outside the persistence helpers. Escape dynamic template text with `escapeHtml()`. Use `toast()`, `uiPrompt()`, and `uiConfirm()` instead of browser-native dialogs.

## Testing Guidelines

No automated test framework is configured. For each change, run the Node syntax check above and manually test the affected workflow in the browser. For persistence changes, test both regular browser use and the sandbox fallback behavior described in `CLAUDE.md`. For prediction import changes, validate both array JSON and object/map JSON inputs.

## Commit & Pull Request Guidelines

This checkout does not include Git history, so no repository-specific commit convention can be inferred. Use concise imperative commit subjects, for example `Fix prediction import validation` or `Update group navigation rendering`.

Pull requests should include a short behavior summary, touched files, manual verification steps, and screenshots or screen recordings for visible UI changes. Link related issues when available.

## Agent-Specific Instructions

Keep edits scoped and preserve the single-file app constraint. If you modify `prode-mundial-2026.html`, reread the file from disk before reporting completion, because maintainer notes call out prior editor/mount desynchronization issues. Keep `docs/DOCUMENTACION.md` synchronized with any functional changes.
