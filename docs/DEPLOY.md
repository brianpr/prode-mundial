# Deploy

Esta app no necesita build. El archivo productivo es `prode-mundial-2026.html` y puede publicarse en cualquier hosting estatico.

## Vercel

El repositorio incluye `vercel.json` para que la ruta `/` sirva el HTML principal. Tambien incluye un `index.html` minimo que redirige a `prode-mundial-2026.html` para hostings que esperan un archivo index.

Configuracion recomendada:

| Campo | Valor |
|---|---|
| Framework Preset | `Other` |
| Build Command | vacio |
| Install Command | vacio |
| Output Directory | vacio o `.` |

Deploy por CLI:

```bash
vercel --prod
```

Deploy por dashboard:

1. Importar el repositorio.
2. Elegir `Other` como framework.
3. Dejar comandos de instalacion y build vacios.
4. Deploy.

## Otros hostings

Funciona en cualquier servidor de archivos estaticos:

- GitHub Pages
- Netlify
- Cloudflare Pages
- Azure Static Web Apps
- un bucket S3 o equivalente

Si el proveedor no respeta rewrites, el `index.html` incluido redirige a la app sin duplicar el archivo fuente.

## Verificacion previa

Antes de publicar cambios en la app:

```bash
node -e "const fs=require('fs');const h=fs.readFileSync('prode-mundial-2026.html','utf8');const m=h.match(/<script>([\s\S]*)<\/script>/);new Function(m[1]);console.log('JS OK')"
```

Y confirmar:

- `prode-mundial-2026.html` termina en `</script></body></html>`;
- no hay `prompt()` ni `confirm()` nativos;
- el flujo de exportar/importar JSON sigue funcionando en navegador.
