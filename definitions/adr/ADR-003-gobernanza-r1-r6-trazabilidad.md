# ADR-003: Gobernanza R1-R6 con trazabilidad obligatoria

- **Estado**: aceptado (retrospectivo)
- **Fecha**: 2026-03-23
- **Tipo**: retrospectivo

## Titulo

Reglas R1-R6 como marco normativo con trazabilidad entre especificacion e implementacion.

## Decision a registrar

Las reglas R1-R6 se tratan como contrato de negocio-tecnico y deben mantener trazabilidad verificable entre specs, contratos API y backend.

## Evidencia encontrada en el proyecto

- `specs/GRIP-SDD-Functional-Specs.md` (seccion de reglas R1-R6).
- `specs/GRIP-SDD-Functional-Specs.md` (indice funcional de R1-R6).
- `specs/backend_audit_report.md` (matriz de correspondencia R* vs codigo).
- `specs/api-contract-ssot.md` (alineacion technical-spec vs schemas Pydantic).
- `.cursorrules` en `grip-specs` (flujo SDD y sincronia documental obligatoria).

## Problema que resuelve

Reduce deriva entre negocio, API y codigo; facilita auditoria de cumplimiento funcional.

## Riesgos o limitaciones

- Alto costo de disciplina documental si no hay gobernanza activa.
- Riesgo de falsa coherencia si auditoria no se actualiza junto con cambios reales.

## Notas de alcance

Este ADR documenta la gobernanza vigente; no define por si solo automatizacion de cumplimiento.

