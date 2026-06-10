# Prode Mundial 2026

Una app web estatica y autocontenida para armar pronosticos del Mundial FIFA 2026. Permite cargar marcadores, comparar variantes, pedir sugerencias a una IA por copiar/pegar y guardar respaldos sin depender de servidores ni cuentas.

![Static HTML](https://img.shields.io/badge/static-HTML%20%2B%20CSS%20%2B%20JS-0f172a)
![No build](https://img.shields.io/badge/build-none-16a34a)
![Deploy](https://img.shields.io/badge/deploy-Vercel%20ready-000000)

## Que incluye

- Prodes ilimitados con nombre y color propio.
- Fixture completo del Mundial 2026 con fase de grupos y eliminatorias.
- Tabla de posiciones por grupo recalculada en vivo.
- Comparacion entre prodes, con copia rapida de marcadores.
- Flujo de asistencia con IA sin integraciones externas: copiar prompt, pegar JSON.
- Exportacion e importacion de prodes individuales.
- Copia de seguridad y restauracion de todos los prodes.
- Persistencia local con fallback para entornos sandbox.

## Demo local

No hay instalacion ni build.

1. Clona o descarga el repositorio.
2. Abre `prode-mundial-2026.html` en el navegador.
3. Completa marcadores, crea variantes y exporta una copia de seguridad cuando quieras mover tus datos.

Tambien podes servirlo con cualquier servidor estatico:

```bash
python -m http.server 8000
```

Luego abre `http://localhost:8000/prode-mundial-2026.html`.

## Como usarlo

1. Elegi el prode activo desde la barra superior.
2. Carga marcadores en cada grupo; la tabla se actualiza automaticamente.
3. Usa `Nuevo`, `Duplicar`, `Renombrar` o `Borrar` para administrar variantes.
4. Activa comparaciones para ver que pusiste en otros prodes.
5. Para asistencia con IA:
   - toca el boton de IA del torneo, grupo o partido;
   - copia el prompt generado;
   - pegalo en tu asistente preferido;
   - pega de vuelta el JSON con marcadores.
6. Usa `Copia de seguridad` para guardar todos los prodes en un archivo JSON.

## Deploy

El repositorio esta preparado para Vercel como sitio estatico. `vercel.json` sirve `/` desde `prode-mundial-2026.html`, sin framework, dependencias ni build.

Opcion rapida:

```bash
vercel --prod
```

En la configuracion del proyecto:

- Framework Preset: `Other`
- Build Command: vacio
- Output Directory: vacio o `.`
- Install Command: vacio

Para mas detalle, ver [docs/DEPLOY.md](docs/DEPLOY.md).

## Estructura

```text
.
├── prode-mundial-2026.html   # App final, autocontenida
├── index.html                 # Redireccion estatica a la app
├── world-cup_2026.json       # Fixture fuente
├── sugerencia-claude.json    # Ejemplo importable de pronosticos
├── vercel.json               # Rewrite para servir la app desde /
├── README.md                 # Guia publica del proyecto
├── AGENTS.md                 # Instrucciones para agentes de mantenimiento
├── CLAUDE.md                 # Notas de mantenimiento y sandbox
└── docs/
    ├── DOCUMENTACION.md      # Documentacion tecnica por feature
    └── DEPLOY.md             # Guia de publicacion estatica
```

## Desarrollo

La app principal debe seguir siendo un unico archivo HTML con CSS y JavaScript inline. No agregues npm, bundlers, assets CDN ni pasos de build salvo que decidas cambiar explicitamente el modelo del proyecto.

Chequeo minimo antes de publicar cambios en el HTML:

```bash
node -e "const fs=require('fs');const h=fs.readFileSync('prode-mundial-2026.html','utf8');const m=h.match(/<script>([\s\S]*)<\/script>/);new Function(m[1]);console.log('JS OK')"
```

Tambien verifica que:

- el archivo termine con `</script></body></html>`;
- no haya llamadas nativas a `prompt()` o `confirm()`;
- la documentacion en `docs/DOCUMENTACION.md` refleje los cambios funcionales.

## Datos

El fixture esta embebido en `prode-mundial-2026.html` y tambien disponible como fuente en `world-cup_2026.json`. Las predicciones se guardan por `id` oficial de partido, no por nombre visible de equipos.

## Privacidad

La app corre en el navegador. No envia datos a un backend y no requiere cuentas. Los pronosticos quedan en el almacenamiento local del navegador o en los archivos JSON que exportes manualmente.

## Licencia

Licencia pendiente de definir. Si vas a publicar el repositorio para uso externo, agrega una licencia explicita antes de aceptar contribuciones.
