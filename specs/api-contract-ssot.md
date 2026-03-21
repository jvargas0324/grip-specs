# Contrato API — SSOT operativo (GRIP)

Este archivo vive en el repo **grip-specs**. El código Pydantic está en el repo hermano **grip-backend** (misma carpeta padre en disco, p. ej. `weekly-app/grip-backend`).

No se mantiene un `openapi.yaml` generado en el repo de specs. El contrato se define así:

1. **Humano / producto:** [technical-spec.md §2](technical-spec.md) (endpoints, payloads, lógica).
2. **Máquina / validación HTTP:** esquemas Pydantic v2 en el repo `grip-backend`: `app/schemas/` y modelos locales en `app/api/` cuando aplica.

Cualquier cambio de API debe actualizar **ambos**: la sección correspondiente de `technical-spec.md` y los schemas (o modelos) del backend.

## Mapa módulo → código

| Módulo | Router (repo `grip-backend`, `app/api/`) | Schemas / cuerpo |
|--------|-------------------------------------------|------------------|
| 1 Ingestión | `ingestion.py` | `app/schemas/ingestion.py` |
| 2 Findings | `findings.py` | `FindingStatusUpdateModel`, `FindingReadModel` en el mismo router |
| 3 Weekly | `weekly.py` | `app/schemas/weekly.py` |
| 4 RAG / Ask GRIP | `rag.py` | `app/schemas/rag.py` |
| 5 DA | `da.py` | `app/schemas/da.py` |

## Reglas de negocio (R1–R6)

Definición canónica: [technical-spec.md §4](technical-spec.md). Trazabilidad implementación: [backend_audit_report.md](backend_audit_report.md) §C.
