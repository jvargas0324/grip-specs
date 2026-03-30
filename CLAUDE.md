# grip-specs — SSOT documental (Claude Code)

Este repo contiene las especificaciones, auditoria de contrato y decisiones arquitectonicas del proyecto GRIP. **No contiene codigo de aplicacion.**

## Mapa de archivos

| Ruta | Proposito |
|------|-----------|
| `specs/GRIP-SDD-Functional-Specs.md` | **SSOT MASTER** — Especificacion funcional completa v1.0.0 (SDD) |
| `specs/GRIP-GapAnalysis.md` | Gap Analysis PoC vs SDD — backlog priorizado |
| `specs/api-contract-ssot.md` | Mapa spec <-> Pydantic en grip-backend (`app/schemas/`) |
| `specs/backend_audit_report.md` | Trazabilidad implementacion vs SDD (en construccion) |
| `specs/features/_template.md` | Plantilla para nuevas feature specs |
| `specs/features/*.md` | Specs individuales por feature |
| `docs/SDD_PROMPT_TEMPLATES.md` | Plantillas de prompt SDD (spec-only, implementacion, cierre) |
| `docs/PR_CHECKLIST.md` | Checklist de PR y orden spec -> backend -> frontend |
| `docs/archive/` | Specs obsoletos de la PoC (SUPERSEDED) |
| `definitions/` | ADRs, RFCs, diagnosticos arquitectonicos |

## Flujo de modificacion de specs

1. Antes de tocar codigo, crear o actualizar `specs/features/<nombre>.md` usando `_template.md`.
2. Toda feature spec debe referenciar la seccion del SDD master que implementa:
   - `specs/GRIP-SDD-Functional-Specs.md` (seccion 5.X correspondiente)
3. Si cambia el contrato API:
   - Actualizar `api-contract-ssot.md` (mapa modulo -> codigo).
   - Actualizar `backend_audit_report.md` (trazabilidad).
4. Para nuevas features: copiar `specs/features/_template.md` a `specs/features/<nombre>.md`.
5. Los specs obsoletos de la PoC estan en `docs/archive/` — no referenciarlos.

## Git

- **Este repo no usa ramas `feature/*` ni `dev`.** Todos los cambios a specs van directamente a `main`. Es un repo de documentacion pura — no hay builds, CI ni releases que proteger, y las specs deben estar disponibles de inmediato como SSOT para backend y frontend.

## Convenciones

- El **SSOT master** es `specs/GRIP-SDD-Functional-Specs.md` v1.0.0. Toda decision funcional se contrasta contra este documento.
- Los schemas Pydantic viven en `grip-backend/app/schemas/`, no aqui. Este repo documenta el contrato humano.
- La carpeta `definitions/` contiene decisiones arquitectonicas (ADRs) y diagnosticos; consultar antes de cambios estructurales al backend.
- La carpeta `docs/archive/` contiene specs de la PoC (SUPERSEDED). No usar como referencia de implementacion.
