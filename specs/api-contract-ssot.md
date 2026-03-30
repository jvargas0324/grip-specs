# Contrato API — SSOT operativo (GRIP)

Este archivo vive en el repo **grip-specs**. El código Pydantic está en el repo hermano **grip-backend** (misma carpeta padre en disco, p. ej. `weekly-apps/grip-backend`).

No se mantiene un `openapi.yaml` generado en el repo de specs. El contrato se define así:

1. **Humano / producto:** [GRIP-SDD-Functional-Specs.md §2](GRIP-SDD-Functional-Specs.md) (endpoints, payloads, lógica).
2. **Máquina / validación HTTP:** esquemas Pydantic v2 en el repo `grip-backend`: `app/schemas/` y modelos locales en `app/api/` cuando aplica.

Cualquier cambio de API debe actualizar **ambos**: la sección correspondiente de `GRIP-SDD-Functional-Specs.md` y los schemas (o modelos) del backend.

## Mapa módulo → código

| Módulo | Router (repo `grip-backend`, `app/api/`) | Schemas / cuerpo |
|--------|-------------------------------------------|------------------|
| 1 Ingestión | `ingestion.py` | `app/schemas/ingestion.py` (canal JSON) + `app/schemas/ingestion_pdf.py` (canal PDF async) |
| 2 Findings | `findings.py` | `FindingStatusUpdateModel`, `FindingReadModel`, `FindingsPageResponse` en el mismo router |
| 2 Tiendas (filtros) | `stores.py` | `StoreOptionReadModel`, `StoreOptionsResponse` en el mismo router |
| 3 Weekly | `weekly.py` | `app/schemas/weekly.py` |
| 4 RAG / Ask GRIP | **`chat.py`** (canónico UI: `POST /api/v1/chat`; auth vía `get_current_user` — headers mock hasta JWT real) y **`rag.py`** (alias: `POST /api/v1/query/ask-grip`; ver technical-spec §2 Módulo 4) | `app/schemas/rag.py` (`AskGripRequest`, `AskGripResponse`, `ChatRequest`) |
| 5 DA | `da.py` | `app/schemas/da.py` |

## Módulo 1 — contrato extendido (canal PDF async)

Coexisten dos canales:

1. **Canal JSON actual:** `POST /api/v1/ingest`.
2. **Canal PDF async nuevo:**
   - `POST /api/v1/ingest/pdf` (crear job de extracción),
   - `GET /api/v1/ingest/pdf/{job_id}` (estado + preview),
   - `POST /api/v1/ingest/pdf/{job_id}/confirm` (persistencia final).

Notas de contrato para el canal PDF:

- Entrada: `multipart/form-data` con PDF de texto digital (máx 1MB, máx 2 páginas).
- Estados de job: `queued | processing | preview_ready | confirmed | failed`.
- Resiliencia: extracción incompleta => persistencia parcial + `warnings` explícitos.
- **`GET /api/v1/ingest/pdf/{job_id}` — modelo `IngestionPdfJobStatusResponse` en `app/schemas/ingestion_pdf.py`:** incluye `error_code` (`ai_quota` \| `ai_unavailable` \| `processing_failed` \| `null`) y `error_detail` (mensaje UI en español cuando `status === failed`; nunca texto crudo de proveedor). Columna persistida: `ingestion_jobs.error_code` (Alembic); `error_detail` en BD almacena el mismo mensaje seguro.

## Reglas de negocio (R1–R6)

Definición canónica: [GRIP-SDD-Functional-Specs.md §4](GRIP-SDD-Functional-Specs.md). Trazabilidad implementación: [backend_audit_report.md](backend_audit_report.md) §C.

## Scope territorial (users -> zones)

- El modelo territorial vigente usa `users.zone_id` (usuario asignado a una sola zona) y elimina la asignación `stores.gv_id`.
- Los endpoints de consulta (`findings`, `weekly`, `stores/options`) aplican scope por zona cuando el rol no es `ceo` **y** `current_user.zone_id` no es nulo (mock: header `X-User-Zone-Id`). Sin `zone_id`, el alcance zonal no se aplica.

## Módulo 2 — contrato paginado de findings

`GET /api/v1/findings` usa paginación y filtros server-side:

- Query params: `page`, `page_size`, `store_code` (preferido), `store_id` (compatibilidad), `status`, `category`, `search`, `date_start`, `date_end`.
- Respuesta:
  - `items: FindingReadModel[]` (cada ítem incluye `store_code` cuando está persistido)
  - `total: int`
  - `page: int`
  - `page_size: int`
  - `total_pages: int`
- Opciones de tienda para el dashboard **no** forman parte de esta respuesta; ver **Módulo 2b** (`GET /api/v1/stores/options`).
- Scope territorial:
  - Para roles no-CEO, `items` se filtran por `current_user.zone_id` **solo si** `zone_id` está presente (p. ej. header `X-User-Zone-Id` en el mock actual). Si no hay `zone_id`, no se aplica filtro zonal (útil solo en pruebas; en producto el usuario territorial debe tener zona asignada).

## Módulo 2b — contrato de tiendas para filtros

`GET /api/v1/stores/options` provee opciones de tienda para poblar dropdown del dashboard:

- Query params: `search` (opcional).
- Respuesta:
  - `items: StoreOptionReadModel[]` con:
    - `store_code: string`
    - `store_name: string | null`
- Scope territorial:
  - Para roles no-CEO, retorna solo tiendas de `current_user.zone_id`.
  - Para CEO, retorna todas las tiendas.
