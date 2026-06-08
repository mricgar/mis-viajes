# Prompt de contexto — Proyecto "Mis Viajes"

## Qué es
Aplicación de planificación de viajes en dos archivos HTML standalone (sin backend, sin framework). Se alojan en GitHub Pages en `https://mricgar.github.io/mis-viajes/`.

## Los dos archivos

### `index.html` — versión de María (la usuaria principal)
- Se loguea con su propio Google y guarda en **su Drive** (busca `viajes_backup.json` con `'me' in owners`)
- Scope: `drive.file`
- Sin `SHARED_FILE_ID` hardcodeado
- Tiene funcionalidad de **subir imagen** por item (se guarda en carpeta `imagenes-viajes-investigados` en Drive, referenciada por `imagenId`)

### `viajes-compartido.html` — versión del novio
- Se loguea con su propio Google pero escribe **siempre en el archivo de María**
- `SHARED_FILE_ID = '1zSlW8dw_5-8A7siAdrFgwR0Y-zsV56VW'` hardcodeado
- Scope: `drive` (completo, necesario para escribir en archivo ajeno)
- También tiene funcionalidad de imagen

Ambos comparten el mismo `CLIENT_ID`: `940633332310-8jvdkccdfeedcbna2sl928bpk3lfe2k1.apps.googleusercontent.com`

## Problema pendiente y crítico en `index.html`
`subirADrive()` usa **`fetch()` con PATCH**, que falla por CORS desde GitHub Pages:
```
Access to fetch blocked by CORS policy: Method PATCH is not allowed
```
La solución es reemplazarlo por `gapi.client.request()` con path relativo:
```javascript
async function subirADrive() {
  if (!driveFileId) return;
  await gapi.client.request({
    path: '/upload/drive/v3/files/' + driveFileId,
    method: 'PATCH',
    params: { uploadType: 'media' },
    headers: { 'Content-Type': 'application/json' },
    body: generarJSON()
  });
  localStorage.setItem('vj_drive_sync', Date.now().toString());
}
```
`viajes-compartido.html` tiene el mismo `fetch` PATCH — aplicar el mismo fix.

## Estructura de datos (localStorage + Drive JSON)
```json
{
  "version": 3,
  "temporadas": [...],
  "destinos": [...],
  "items": [
    {
      "id": 1234567890,
      "destinoId": 111,
      "tipo": "vuelo|hotel|actividad|restaurante|general",
      "estado": "investigando|reservado|confirmado|descartado",
      "titulo": "...",
      "precio": "...",
      "fechas": "...",
      "rating": "1-5",
      "url": "...",
      "notas": "...",
      "favorito": false,
      "imagenId": "drive_file_id_opcional",
      "fecha": "01 ene 2025"
    }
  ]
}
```

## Claves localStorage
- `vj_temporadas_v1`, `vj_destinos_v1`, `vj_items_v1`, `vj_tema_v1`, `vj_drive_sync`

## Lo que funciona bien (no tocar)
- Auth overlay con stub inline antes del `<script>` principal (soluciona "handleAuthClick is not defined" que ocurre cuando el script principal falla al parsear)
- `window.onerror` al inicio del script principal para mostrar errores en `#auth-status`
- `esperarGapi()` con timeout de 150 intentos × 100ms (15s)
- `startDriveApp()` filtra con `'me' in owners` para no coger archivos compartidos
- Imagen en items: upload → Drive folder → `imagenId` en el item → botón "🖼 Ver imagen" en tarjeta
- Toggle tema claro/oscuro, filtros por tipo, favoritos, clonar item, descartar/reactivar

## Bugs conocidos / pendientes
1. **CORS en `subirADrive`** — fix descrito arriba, aplicar en ambos archivos
2. `viajes-compartido.html` no tiene el filtro `'me' in owners` (no lo necesita, usa ID fijo)
3. El `fetch` de `subirImagenADrive` (subida de imágenes) también usa `fetch` con multipart — si da CORS habrá que migrarlo también a `gapi.client.request` con base64
