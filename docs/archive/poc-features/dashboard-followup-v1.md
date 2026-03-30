# Dashboard Follow-up (M2) v1

> Spec retroactiva — documenta la funcionalidad ya implementada del Módulo 2 (Dashboard JZ, Follow-up de hallazgos y filtro de tiendas). Creada a partir de análisis del código en `grip-backend` y `grip-frontend`.

- **Estado:** completada
- **Spec base alineada:** v7.20
- **Última actualización:** 2026-03-28

## 1. Objetivo

- **Aplicación(es):** `grip-backend` y `grip-frontend`.
- **Problema o necesidad:** El Jefe de Zona y roles superiores necesitan visualizar, filtrar, paginar y resolver hallazgos de visitas. El flujo de follow-up requiere validar reglas de negocio (R1, R2, R4) al cambiar el estado de un hallazgo.
- **Alcance:**
  - **Backend:** Endpoint paginado de hallazgos con filtros server-side, endpoint de actualización de estado con reglas R1/R2/R4, endpoint de opciones de tienda desacoplado con scope zonal.
  - **Frontend:** Dashboard con tabla paginada, panel de filtros (búsqueda, estado, tienda, categoría, fechas), modal de resolución con validación R1 client-side.
  - **Permisos:** Todos los roles autenticados pueden consultar hallazgos (scope filtrado por zona para no-CEO). Todos los roles autenticados pueden actualizar estado.

## 2. Referencia a specs base

- Reglas de negocio: [functional-spec.md](../functional-spec.md) §2.2 (Módulo 2: Follow-up).
- Contrato y arquitectura: [technical-spec.md](../technical-spec.md) §2 (Módulo 2), §4 (R1-R6).
- SSOT Pydantic: [api-contract-ssot.md](../api-contract-ssot.md) → Módulo 2 (findings) y Módulo 2b (stores).
- Trazabilidad: [backend_audit_report.md](../backend_audit_report.md) §B y §C.

## 3. Actores y permisos

- **Todos los roles autenticados** (`jz`, `gv`, `regional`, `coo`, `ceo`) pueden consultar hallazgos y actualizar estado.
- **Scope territorial:** roles no-CEO ven solo hallazgos/tiendas de su zona (`current_user.zone_id`). CEO ve todo.
- **Autenticación actual:** headers mock (`X-User-Id`, `X-User-Role`, `X-User-Zone-Id`) hasta implementación de JWT real.

## 4. Flujos principales

### 4.1 Consultar hallazgos (Dashboard)

1. El usuario abre `/dashboard`.
2. El componente carga opciones de tienda (`GET /api/v1/stores/options`) y hallazgos paginados (`GET /api/v1/findings`).
3. El usuario puede filtrar por: búsqueda libre, estado, tienda, categoría, rango de fechas.
4. Cada cambio de filtro reinicia la paginación a página 1 y recarga.
5. El usuario navega entre páginas con botones Anterior/Siguiente.

### 4.2 Resolver un hallazgo

1. El usuario pulsa "Resolver" en una fila de la tabla.
2. Se abre un modal con el resumen IA del hallazgo (solo lectura).
3. El usuario selecciona nuevo estado (`in_progress` o `verified`) y opcionalmente adjunta evidencia.
4. **R1 (client-side):** Si el estado es `verified`, el campo de evidencia se vuelve obligatorio; el botón de envío se deshabilita hasta completarlo.
5. Al confirmar, el cliente llama `PATCH /api/v1/findings/{id}/status`.
6. **R1 (server-side):** Si `status == verified` y no hay evidencias asociadas al hallazgo, retorna HTTP 400.
7. **R4 (server-side):** Si la tienda tiene un DA activo y el estado destino es `verified` o `in_progress`, retorna HTTP 403.
8. **R2 (server-side):** Si el estado es `reopened`, la severidad se eleva automáticamente a `high`.
9. En éxito, el modal se cierra y la lista se recarga.
10. En error, se muestra el mensaje devuelto por el API.

## 5. Reglas de negocio

1. **R1 — Evidencia obligatoria para verificación:** No se puede marcar un hallazgo como `verified` sin al menos una evidencia asociada. Validado en cliente (formulario inválido) y servidor (HTTP 400).
2. **R2 — Reapertura eleva severidad:** Al cambiar estado a `reopened`, `severity_rank` se establece automáticamente a `high`.
3. **R4 — Bloqueo por DA activo:** Si la tienda del hallazgo tiene una `DAIntervention` con `status == active`, no se permite transición a `verified` o `in_progress` (HTTP 403).
4. **Scope zonal:** Roles no-CEO solo ven hallazgos y tiendas de su zona. CEO sin restricción.
5. **Paginación server-side:** `page_size` se limita al rango [1, 100]. Si `page > total_pages`, se clampea a la última página.

## 6. Contrato API

### 6.1 Listar hallazgos (paginado)

- **GET** `/api/v1/findings`
- **Query params:**

| Param | Tipo | Default | Descripción |
|-------|------|---------|-------------|
| `page` | int | 1 | Página actual (1-indexed) |
| `page_size` | int | 20 | Items por página (1-100) |
| `store_code` | string | — | Filtro por código de tienda exacto |
| `store_id` | UUID | — | Filtro por ID de tienda (compatibilidad) |
| `status` | string | — | `open` \| `in_progress` \| `verified` \| `reopened` |
| `category` | string | — | Filtro por categoría exacta |
| `search` | string | — | Búsqueda ILIKE en category, control_point, ai_summary, suggested_action, severity_rank, observation, status |
| `date_start` | date | — | Hallazgos con `created_at >= date_start` |
| `date_end` | date | — | Hallazgos con `created_at <= date_end` |

- **Response 200:**

```json
{
  "items": [
    {
      "id": "UUID",
      "visit_id": "UUID | null",
      "store_code": "string | null",
      "category": "string | null",
      "control_point": "string | null",
      "compliance": true,
      "observation": "string | null",
      "ai_summary": "string | null",
      "suggested_action": "string | null",
      "severity_rank": "low | medium | high | null",
      "parent_finding_id": "UUID | null",
      "status": "open | in_progress | verified | reopened",
      "created_at": "ISO8601"
    }
  ],
  "total": 42,
  "page": 1,
  "page_size": 20,
  "total_pages": 3
}
```

- **Ordenamiento:** `created_at DESC`.
- **Scope:** Si `current_user.role != ceo` y `zone_id` presente, filtra via `Finding → Visit → Store.zone_id`.

### 6.2 Actualizar estado de hallazgo

- **PATCH** `/api/v1/findings/{finding_id}/status`
- **Request:**

```json
{
  "status": "open | in_progress | verified | reopened",
  "evidence_url": "string | null",
  "resolution_notes": "string | null"
}
```

- `evidence_url`: si se provee y no está vacío, se crea un registro en `evidences` antes de validar R1.
- `resolution_notes`: capturado en el payload pero **no persistido** en la implementación actual (reservado para uso futuro).

- **Response 200:**

```json
{
  "id": "UUID",
  "status": "string",
  "severity_rank": "string | null"
}
```

- **Errores:**
  - `400` — R1: intento de `verified` sin evidencias.
  - `403` — R4: DA activo bloquea `verified` o `in_progress`.
  - `404` — Hallazgo no encontrado.

- **Secuencia interna:**
  1. Buscar hallazgo; 404 si no existe.
  2. Crear evidencia si `evidence_url` provisto.
  3. Validar R1 (conteo de evidencias).
  4. Resolver `store_id` via Visit FK.
  5. Validar R4 (DA activo).
  6. Aplicar R2 (severidad `high` si `reopened`).
  7. Actualizar estado, commit, retornar.
  8. Si R1/R4 falla, toda la transacción (incluida la evidencia) hace rollback.

### 6.3 Opciones de tienda (dropdown)

- **GET** `/api/v1/stores/options`
- **Query params:**

| Param | Tipo | Default | Descripción |
|-------|------|---------|-------------|
| `search` | string | — | Búsqueda ILIKE en `code` y `name` |

- **Response 200:**

```json
{
  "items": [
    {
      "store_code": "string",
      "store_name": "string | null"
    }
  ]
}
```

- **Ordenamiento:** `Store.code ASC`.
- **Scope:** Misma lógica zonal que findings.

## 7. Cambios en UI (frontend)

### 7.1 Dashboard (`DashboardJzComponent`)

- **Ruta:** `/dashboard` (ruta por defecto del SPA, dentro de `MainLayoutComponent`).
- **Header:** Título "Dashboard Jefe de Zona" + botón "Cargar Checklist" que navega a `/checklist`.
- **Panel de filtros:** Grid de 6 columnas con:
  - Búsqueda libre (ILIKE server-side).
  - Select de estado (Todos, Abierto, En progreso, Verificado, Reabierto).
  - Select de tienda (cargado desde `stores/options`, muestra nombre + código).
  - Select de categoría (derivado dinámicamente de los hallazgos cargados).
  - Date pickers inicio/fin.
- **Tabla de hallazgos:** 8 columnas (Fecha, Estado, Código tienda, Categoría, Punto de control, Severidad, Resumen IA, Acciones).
  - Badges de color por estado: zinc (open), amber (in_progress), emerald (verified), orange (reopened).
  - Badges de severidad: red (high), amber (medium), gray (otro).
- **Paginación:** Botones Anterior/Siguiente con texto "Mostrando página X de Y - Total: Z".
- **Protección contra race conditions:** Contador secuencial (`findingsLoadSeq`) descarta respuestas obsoletas.

### 7.2 Modal de resolución

- **Trigger:** Botón "Resolver" por fila.
- **Overlay:** `fixed inset-0 bg-black/50 z-50`, cierre por backdrop click o ESC.
- **Contenido:**
  - Resumen IA del hallazgo (solo lectura).
  - Select de estado: `in_progress` (default) o `verified`.
  - Input de evidencia (enlace o texto).
  - Validación R1 dinámica: si `status === verified`, evidencia se vuelve `required`. Mensaje de error: "La evidencia es obligatoria al marcar como Verificado (R1)".
  - Botón Guardar deshabilitado si formulario inválido.

### 7.3 Servicios

- **`FindingsService`:** `getFindings(params)`, `updateFindingStatus(id, payload)`. Mapea `evidence_link` (frontend) → `evidence_url` (backend). Maneja formato de respuesta paginado y legacy (array).
- **`StoresService`:** `getStoreOptions(search?)`. Retorna `StoreOptionsResponse`.

## 8. Criterios de aceptación

- [x] Dado un usuario autenticado, cuando abre `/dashboard`, entonces ve la tabla de hallazgos paginada con 20 items por página ordenados por fecha descendente.
- [x] Dado un usuario no-CEO con `zone_id`, cuando consulta hallazgos, entonces solo ve hallazgos de tiendas de su zona.
- [x] Dado un CEO, cuando consulta hallazgos, entonces ve hallazgos de todas las zonas.
- [x] Dado cualquier filtro activo (búsqueda, estado, tienda, categoría, fechas), cuando el usuario cambia el filtro, entonces la paginación se reinicia a página 1 y la tabla se recarga con resultados filtrados.
- [x] Dado un hallazgo con estado `open`, cuando el usuario pulsa "Resolver" y selecciona `in_progress` sin evidencia, entonces la actualización se ejecuta correctamente.
- [x] Dado un hallazgo, cuando el usuario selecciona `verified` sin completar evidencia, entonces el botón Guardar permanece deshabilitado (R1 client-side).
- [x] Dado un hallazgo sin evidencias previas, cuando el servidor recibe `status: verified`, entonces retorna HTTP 400 con mensaje de R1.
- [x] Dado un hallazgo de una tienda con DA activo, cuando se intenta `verified` o `in_progress`, entonces el servidor retorna HTTP 403 (R4).
- [x] Dado un hallazgo, cuando se cambia a `reopened`, entonces `severity_rank` se actualiza a `high` automáticamente (R2).
- [x] Dado el dropdown de tiendas, cuando el usuario abre el dashboard, entonces las opciones se cargan desde `GET /stores/options` filtradas por zona.

## 9. Notas

- `resolution_notes` se captura en el payload del PATCH pero **no se persiste** en la implementación actual. Reservado para uso futuro.
- El filtro de categoría se deriva dinámicamente de los hallazgos cargados en la página actual, no de un catálogo server-side.
- El componente usa `NgZone.run()` y `ChangeDetectorRef.detectChanges()` para forzar re-render agresivo y evitar race conditions en navegación rápida.
- Esta spec fue creada retroactivamente (2026-03-28) a partir del código implementado. Absorbe el contenido de `docs/PLAN_MODULO2_FOLLOWUP.md` que sirvió como plan de ejecución original.
