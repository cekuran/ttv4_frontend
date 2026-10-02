# TaskTime frontend

Aplicación de una sola página en un único archivo, [index.html](index.html). CSS, HTML y JS viven juntos; sin build, bundler ni transpiler: lo que se commitea es lo que se sirve. Dependencias externas por CDN: [D3 v7](https://d3js.org/) (gráficas), Google Fonts y Material Icons.

```
index.html → API_URL (proxy Cloudflare) → backend (Apps Script) → Google Sheets
```

## Pantallas

Tres bloques hermanos dentro de `<body>`:

- **Setup** (`#setup-screen`): primera vez, cuando `ping` responde `ok: false`. Pide el ID de la hoja maestra y las credenciales del admin (`configurarSpreadsheetMaestro`).
- **Login** (`#login-screen`): se oculta cuando hay un token válido en `localStorage`.
- **App**: navegación lateral con las vistas `Dashboard`, `Tasks`, `Logs`, `Stats`, `Recurrentes` (rutinas) y `Admin` (solo visible si `rol === 'admin'`). Además existen `Mi cuenta` y `no-hojas` (usuario sin hoja vinculada, a la espera del admin).

## Configuración

- **`API_URL`** (`const API_URL = '...'`, sección `==== Backend API ====`): URL del **proxy** de Cloudflare, no del backend directo. [../tools/Set-ApiDeployment.ps1](../tools/Set-ApiDeployment.ps1) la reescribe (opciones 1, 5 u 8). Para desarrollo local usar `http://127.0.0.1:8787` (proxy con `npm run dev`).
- **Sesión** en `localStorage`:
  - `tasktime_auth_token`: token emitido por `loginUsuario`; se envía en el cuerpo de cada petición.
  - `tasktime_session`: datos de la sesión (usuario, rol, hoja activa).
- Un error `No autenticado` limpia ambas claves automáticamente.

## Llamadas a la API

```js
const tasks = await call('getTasks');
```

- `call(fn, ...args)` → `rawCall` hace `POST` a `API_URL` con `{ action, args, token }` y `Content-Type: text/plain;charset=utf-8` (Apps Script rechaza `application/json`).
- Respuesta `{ ok, data }` o `{ ok: false, error }`; los errores se lanzan como `Error` con `serverError = true`.
- `bootstrap`, `getRoutineStatus`, `getDailyRoutineWeekStatus` y `adminOverview` deduplican peticiones idénticas en vuelo (`DEDUPE_ACTIONS`).
- Cold-start de Apps Script: ante un `504` del proxy se reintenta hasta 2 veces (esperas de 30 s y 15 s).

## Ejecutar en local

```bash
python -m http.server 8000 -d frontend
# abrir http://localhost:8000
```

No abrir `index.html` con `file://`: el `fetch` desde origen nulo es bloqueado. Alternativa: opción 4 de `Set-ApiDeployment.ps1`, que elige un puerto libre entre 8000 y 8099.

## Despliegue

Se publica en Netlify. Recomendado, desde la raíz del repo:

```powershell
pwsh ./tools/Set-ApiDeployment.ps1   # opción 6: deploy del frontend en Netlify
```

Manual, desde la raíz del repo:

```bash
netlify deploy --dir frontend --no-build --prod
```

`.netlifyignore` excluye `.git`, `.gitignore`, `*.bak` y `*.md` del deploy. La URL publicada se registra en [../release_info.txt](../release_info.txt) (`[netlify]`).

## Convenciones

- Estado en un único objeto `state`; las mutaciones pasan por pequeñas funciones de render, sin framework. Los arrays `state.tasks` / `state.labels` se reemplazan (no se mutan) para que los índices `taskById` / `labelById` se reconstruyan.
- No añadir paso de build ni dependencias de npm.
- Las acciones del backend se consumen con `call('accion', ...)`; ver [../backend/README.md](../backend/README.md).

## Estructura

```
frontend/
├── index.html        # la app completa
├── .netlifyignore    # archivos excluidos del deploy
└── .gitignore
```
