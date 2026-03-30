# ADR-002: Resiliencia IA por degradacion no bloqueante

- **Estado**: aceptado (retrospectivo)
- **Fecha**: 2026-03-23
- **Tipo**: retrospectivo

## Titulo

Fallas de IA degradan salida, pero no bloquean transacciones criticas.

## Decision a registrar

Cuando un proveedor IA falle (timeout, cuota, error API), el sistema debe preservar la operacion principal y degradar campos IA no criticos, evitando perdida de transaccion de negocio.

## Evidencia encontrada en el proyecto

- `specs/GRIP-SDD-Functional-Specs.md` (seccion de resiliencia): establece no bloqueo por fallas IA.
- `specs/backend_audit_report.md` (seccion de resiliencia/trazabilidad): confirma implementacion en rutas criticas.
- `grip-backend/app/core/ai_client.py`: taxonomia de errores (`GeminiAPIError`, `GeminiQuotaError`, `GeminiTimeoutError`).
- `grip-backend/app/services/weekly_service.py`: fallback cuando falla generacion IA.
- `grip-backend/app/services/da_service.py`: cierre DA persiste aunque falle embedding post-cierre.
- `grip-backend/app/services/ingestion_service.py`: fallback ante salida IA invalida.

## Problema que resuelve

Evita que una dependencia externa IA comprometa continuidad operativa en procesos de negocio.

## Riesgos o limitaciones

- Degradacion puede reducir calidad informativa temporal.
- Sin reintentos/colas formalizados, el enriquecimiento puede quedar incompleto sin recuperacion diferida.
- Requiere observabilidad para detectar degradaciones silenciosas.

## Notas de alcance

La estrategia de retries/backoff/circuit breaker queda como decision prospectiva adicional.

