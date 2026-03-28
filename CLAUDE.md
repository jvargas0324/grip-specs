# grip-specs — SSOT documental (Claude Code)

Este repo contiene las especificaciones, auditoria de contrato y decisiones arquitectonicas del proyecto GRIP. **No contiene codigo de aplicacion.**

## Mapa de archivos

| Ruta | Proposito |
|------|-----------|
| `specs/technical-spec.md` | API, esquema de datos, prompts, reglas R1-R6 tecnicas |
| `specs/functional-spec.md` | Journeys, gobernanza, indice R1-R6 funcional |
| `specs/api-contract-ssot.md` | Mapa spec <-> Pydantic en grip-backend (`app/schemas/`) |
| `specs/backend_audit_report.md` | Trazabilidad implementacion vs spec |
| `specs/features/_template.md` | Plantilla para nuevas feature specs |
| `specs/features/*.md` | Specs individuales por feature |
| `docs/SDD_PROMPT_TEMPLATES.md` | Plantillas de prompt SDD (spec-only, implementacion, cierre) |
| `docs/PR_CHECKLIST.md` | Checklist de PR y orden spec -> backend -> frontend |
| `definitions/` | ADRs, RFCs, diagnosticos arquitectonicos |

## Flujo de modificacion de specs

1. Antes de tocar codigo, editar los `.md` en `specs/`.
2. Si cambia el contrato API o reglas R1-R6:
   - Actualizar `technical-spec.md` (seccion correspondiente).
   - Actualizar `api-contract-ssot.md` (mapa modulo -> codigo).
   - Actualizar `functional-spec.md` §6.0 si el indice R1-R6 funcional se ve afectado.
   - Actualizar `backend_audit_report.md` (trazabilidad).
   - Reconciliar la version del encabezado del audit con la version en `technical-spec.md`.
3. Para nuevas features: copiar `specs/features/_template.md` a `specs/features/<nombre>.md`.

## Convenciones

- Las reglas R1-R6 se definen **solo** en `technical-spec.md` §4 (normativa) y `functional-spec.md` §6.0 (resumen funcional). No crear definiciones alternativas.
- Los schemas Pydantic viven en `grip-backend/app/schemas/`, no aqui. Este repo documenta el contrato humano.
- La carpeta `definitions/` contiene decisiones arquitectonicas (ADRs) y diagnosticos; consultar antes de cambios estructurales al backend.
