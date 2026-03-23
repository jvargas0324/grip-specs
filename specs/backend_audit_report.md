# Backend Audit Report — GRIP API

**Spec de referencia:** Technical Spec **v7.19** (v7.18 + UX M1: pestañas checklist manual vs PDF en `ChecklistFormComponent`; sin cambio de contrato API backend)  
**Functional Spec:** Especificación funcional maestra (Fase 1), índice R1–R6 en §6.0  
**Alcance:** Código en `grip-backend/app` (config, core, db/models, schemas, api, services).  
**Última actualización de este informe:** 2026-03-23 (v7.19: alineación documental con pestañas de ingesta M1 en frontend; cambio solo UI.)

---

## Resumen ejecutivo

| Resultado | Cantidad |
|-----------|----------|
| 🟢 PASS   | 46       |
| 🟡 WARNING| 2        |
| 🔴 FAIL   | 0        |

---

## A. Esquema de base de datos y vectores

| Ítem | Resultado | Detalle |
|------|-----------|---------|
| Columnas `ai_summary` y `suggested_action` en modelo Finding | 🟢 PASS | Definidas en `app/db/models/findings.py` como `Text`, nullable. |
| Migración Alembic para ai_summary / suggested_action | 🟢 PASS | `alembic/versions/0004_add_findings_ai_summary_suggested_action.py` añade ambas columnas. |
| Columna `embedding` en findings con 768 dimensiones | 🟢 PASS | `findings.py`: `Vector(768)`. Migración 0003 ya aplica 768. |
| Columna `embedding` en strategic_feedback con 768 dimensiones | 🟢 PASS | `app/db/models/strategic_feedback.py`: `Vector(768)`. |
| Coherencia spec v7.11 ↔ modelos SQLAlchemy | 🟢 PASS | Tabla findings en spec incluye ai_summary, suggested_action, embedding VECTOR(768); modelos alineados. |
| Índices de performance para dashboard findings | 🟢 PASS | Migración `alembic/versions/0006_add_findings_performance_indexes.py` agrega índices para `findings(created_at desc/status/visit_id/category/status+created_at)` y `visits(store_id)` alineados a consultas paginadas del dashboard JZ. |
| Relación territorial users->zones (sin stores.gv_id) | 🟢 PASS | Migración `alembic/versions/0007_users_zone_scope.py` elimina `stores.gv_id` y agrega `users.zone_id` con FK a `zones` e índice de soporte. Modelos `users.py` y `stores.py` alineados. |
| Columna `store_code` en `findings` + índice | 🟢 PASS | Modelo `app/db/models/findings.py`; migración `alembic/versions/0008_add_findings_store_code.py` (columna, backfill vía `visits→stores`, índice `ix_findings_store_code`). Ingesta JSON/PDF persiste `store_code`. |

---

## B. Contratos de API (módulos 1–5)

| Ítem | Resultado | Detalle |
|------|-----------|---------|
| POST /api/v1/ingest — payload (header + raw_content) | 🟢 PASS | `app/schemas/ingestion.py`: IngestRequest con IngestHeader (store_code, jz_email, visit_date) y raw_content (list IngestItem con category, item, status, comment). |
| POST /api/v1/ingest — respuesta (visit_id, findings con summary/suggested_action) | 🟢 PASS | IngestResponse con visit_id y findings (IngestFindingSummary con summary, suggested_action). Router en `app/api/ingestion.py`. |
| POST /api/v1/ingest/pdf — creación de job async | 🟢 PASS | `app/api/ingestion.py` agrega endpoint async y respuesta con `job_id/status/limits`; esquema en `app/schemas/ingestion_pdf.py`. |
| GET /api/v1/ingest/pdf/{job_id} — estado/preview | 🟢 PASS | Implementado en `app/api/ingestion.py`; expone estado y preview serializado desde `ingestion_jobs.extraction_result`. |
| POST /api/v1/ingest/pdf/{job_id}/confirm — persistencia final | 🟢 PASS | Implementado en `app/api/ingestion.py` + `app/services/ingestion_pdf_service.py`; persiste visita/hallazgos desde snapshot de preview (sin recálculo IA) y retorna warnings. |
| GET /api/v1/weekly/summary — query region_id, week_number | 🟢 PASS | `app/api/weekly.py`: parámetros opcionales region_id, week_number. |
| GET /api/v1/weekly/summary — respuesta (compliance_chart, top_high_findings, focal_points, executive_brief) | 🟢 PASS | WeeklySummaryResponse en `app/schemas/weekly.py` con esos campos. |
| POST /api/v1/weekly/feedback — payload (session_id, questions, target_id) | 🟢 PASS | WeeklyFeedbackRequest con session_id, questions (min 3, max 3), target_id. Spec usa target_id (alias target_user_id). |
| POST /api/v1/chat — payload (message, store_code?) | 🟢 PASS | `ChatRequest` en `app/schemas/rag.py`; router `app/api/chat.py` con `Depends(get_current_user)`. Misma lógica que ask-grip vía `rag_service.ask_grip`. **Canónico para el SPA.** |
| POST /api/v1/query/ask-grip — payload (query) | 🟢 PASS | AskGripRequest con query. Router en `app/api/rag.py` bajo prefix /api/v1/query. |
| POST /api/v1/query/ask-grip — autenticación vs `/api/v1/chat` | 🟡 WARNING | El handler en `rag.py` no aplica `get_current_user` en la firma; `/api/v1/chat` sí. Deuda documentada en technical-spec §2 Módulo 4; alinear antes de exponer el alias en entornos sin otra capa de auth. |
| POST /api/v1/da/create — payload (target_regional_id, store_id, due_date) | 🟢 PASS | DACreateRequest en `app/schemas/da.py`. Protegido con require_role(['ceo']). |
| PATCH /api/v1/da/close — payload (da_id, firmas, resolution_notes) | 🟢 PASS | DACloseRequest con da_id, regional_signature_hash, ceo_signature_hash, resolution_notes opcional. |
| PATCH /api/v1/findings/{id}/status — body (status) | 🟢 PASS | FindingStatusUpdateModel con status (open, in_progress, verified, reopened). Router en `app/api/findings.py`. |
| GET /api/v1/findings — paginación + filtros server-side | 🟢 PASS | `app/api/findings.py` implementa `page/page_size`, filtros (`search`, `category`, `date_start`, `date_end`, `status`, `store_id`), `COUNT(*)`, `LIMIT/OFFSET`, orden `created_at desc` y respuesta paginada (`items/total/page/page_size/total_pages`). |
| GET /api/v1/findings — filtro por `store_code` | 🟢 PASS | El contrato mantiene filtro principal por `store_code` con paginación y scope por zona para no-CEO. |
| GET /api/v1/stores/options — dropdown zonal desacoplado | 🟢 PASS | Endpoint dedicado retorna opciones `{store_code, store_name}` para poblar dropdown por zona sin depender de `/findings`. |
| Scope por zona de usuario en findings/weekly | 🟢 PASS | `CurrentUser` incorpora `zone_id` (`X-User-Zone-Id`) en `app/api/dependencies.py`; `findings.py` y `weekly_service.py` aplican alcance por `current_user.zone_id` en roles no CEO, preservando contratos HTTP. |
| Registro de routers en main.py | 🟢 PASS | `app/main.py`: ingestion, weekly, **chat**, rag, da, findings, **stores** incluidos con `include_router`. |

---

## C. Hard constraints (R1–R6)

Definición canónica: [technical-spec.md §4](technical-spec.md). El **cierre DA con ambos hashes** es obligación del contrato API (§2 Módulo 5), no el identificador R6. **R6** = memoria post-DA / RAG.

| Regla | Resultado | Detalle |
|-------|-----------|---------|
| R1 (Bloqueo de cierre sin evidencia) | 🟢 PASS | `app/api/findings.py`: si `new_status == 'verified'`, cuenta `Evidence` por `finding_id`; si 0 → HTTP 400. Comentario `Implementa R1`. |
| R2 (Persistencia: reopened ⇒ severity_rank = 'high') | 🟢 PASS | `app/api/findings.py`: si `new_status == 'reopened'` se asigna `finding.severity_rank = 'high'`. Comentario `Implementa R2`. |
| R3 (Visibilidad directa / drill-down CEO) | 🟢 PASS | `app/services/weekly_service.py`: si `current_user.role != 'ceo'` se aplica filtro por región; si es `ceo` no se aplica. Spec: bypass documentado en `GET /api/v1/weekly/summary`; `GET /findings` puede alinearse en el futuro. |
| R4 (Bloqueo por DA activo en tienda) | 🟢 PASS | `app/api/findings.py`: si existe `DAIntervention` con `store_id` y `status == 'active'`, HTTP 403 al pasar a `verified` o `in_progress`. Comentario `Implementa R4`. |
| R5 (Escalamiento de firma — MFA para ceo_signature_hash) | 🟡 WARNING | `close_da` exige presencia de `regional_signature_hash` y `ceo_signature_hash`. No hay integración MFA real para el hash del CEO. |
| R6 (Memoria post-DA — RAG al cerrar) | 🟢 PASS | `app/services/da_service.py` `close_da`: con texto en `resolution_notes` o `structural_action_plan`, llama `generate_embedding` y persiste `StrategicFeedback` con `embedding` (consultable vía RAG junto a otros embeddings). Comentario `Implementa R6`. |

---

## D. Integración IA

Normativa documentada en [technical-spec.md §3.F](technical-spec.md) (resiliencia y degradación).

| Ítem | Resultado | Detalle |
|------|-----------|---------|
| Modelo de generación de texto: gemini-2.5-flash-lite | 🟢 PASS | `app/core/ai_client.py`: TEXT_MODEL = "gemini-2.5-flash-lite". |
| Modelo de embeddings: gemini-embedding-001 (768 dims) | 🟢 PASS | `app/core/ai_client.py`: EMBEDDING_MODEL, EMBEDDING_DIMENSIONS = 768. |
| Excepciones tipadas (GeminiAPIError, GeminiQuotaError, GeminiTimeoutError) | 🟢 PASS | Definidas y mapeadas desde `google.api_core` en `app/core/ai_client.py`. |
| Weekly (M3): frase literal anti-sesgo en system prompt | 🟢 PASS | `app/services/weekly_service.py`: WEEKLY_SYSTEM_PROMPT incluye detección de contradicción con reincidencias. |
| RAG (M4): frase literal riesgo sistémico en system prompt | 🟢 PASS | `app/services/rag_service.py`: RAG_SYSTEM_PROMPT. |
| Ingestion: Gemini para extracción + embedding real (sin mocks) | 🟢 PASS | `process_ingestion` usa `generate_structured_text` y `generate_embedding`; fallos propagados al router. |
| Ingestion PDF async: extracción IA de header/checklist + trazabilidad (`confidence`, `warnings`) | 🟢 PASS | `app/services/ingestion_pdf_service.py` construye preview y persiste `confidence_score`, `warnings`, `missing_fields` en `ingestion_jobs`. |
| Ingestion PDF async: optimización de costo (preview=confirm) | 🟢 PASS | `confirm_pdf_job` reutiliza snapshot `preview_ready` para persistir, evitando una segunda ronda de llamadas Gemini durante confirmación. |
| Ingestion PDF async: almacenamiento de original en object storage (retención 1 año) | 🟢 PASS | `app/core/object_storage.py` + `ingestion_pdf_service.py` almacenan PDF fuente y guardan `retention_until` (365 días). |
| Ingestion PDF async: control de rol `jefe_zona` en upload/confirm | 🟢 PASS | Endpoints M1-PDF usan `Depends(require_role(['jz']))` en `app/api/ingestion.py`. |
| Weekly: Gemini para focal_points y executive_brief | 🟢 PASS | `get_weekly_summary` con try/except: fallo Gemini degrada campos generados sin tumbar el agregado numérico. |
| RAG: vectorizar pregunta, búsqueda coseno, contexto a Gemini | 🟢 PASS | `ask_grip`: try/except `GeminiAPIError`/`GeminiQuotaError`; respuesta degradada con mensaje al usuario. |
| DA: Gemini para proven_facts y root_cause_analysis | 🟢 PASS | `create_da` con try/except: fallo Gemini deja expediente sin texto generado pero persiste DA. |
| DA cierre (R6): embedding no bloquea commit | 🟢 PASS | `da_service.close_da`: si `generate_embedding` lanza `GeminiAPIError`, se loguea warning y el `commit` del cierre sigue. |
| Manejo HTTP ingestión (503/429) ante Gemini | 🟢 PASS | `app/api/ingestion.py`: `GeminiQuotaError` → 429; `GeminiAPIError` → 503. |

---

## Correspondencia R* ↔ código (canónica)

| ID | Ubicación principal |
|----|---------------------|
| **R1** | `app/api/findings.py` — verificación sin evidencia |
| **R2** | `app/api/findings.py` — reopened → severity high |
| **R3** | `app/services/weekly_service.py`, `app/api/weekly.py` — bypass CEO en resumen semanal |
| **R4** | `app/api/findings.py` — DA activo bloquea verified/in_progress |
| **R5** | Pendiente MFA; `close_da` solo valida hashes |
| **R6** | `app/services/da_service.py` `close_da` — embedding + StrategicFeedback para RAG |

**Mantenimiento:** Tras cambiar contratos o reglas R*, actualizar [technical-spec.md](technical-spec.md), [functional-spec.md §6.0](functional-spec.md), [api-contract-ssot.md](api-contract-ssot.md) y esta sección en el mismo cambio; **reconciliar la versión** del encabezado de este informe con la versión declarada en `technical-spec.md`.

---

*Fin del reporte. Alineado con Technical Spec v7.17 y revisión SDD 2026-03-23 (M1-PDF + performance M2 + store_code + stores/options + users->zones + reconciliación contrato SSOT).*
