# Ejemplo — Spec de feature (referencia SDD)

> Archivo de ejemplo rellenado a propósito pedagógico. Las features reales deben vivir en `specs/features/<nombre>.md` sin prefijo `_example-`.

## 1. Objetivo

- **Aplicación(es):** grip-frontend (principal), grip-backend (solo si se añade query opcional).
- **Problema o necesidad:** El Gerente Regional quiere filtrar el resumen semanal por rango de fechas sin recargar toda la página.
- **Alcance:** UI: parámetros `start_date` / `end_date` ya contemplados en API. Backend: sin cambios si el contrato actual basta. Fuera de alcance: export PDF.

## 2. Referencia a specs base

- Reglas de negocio: [functional-spec.md](../functional-spec.md) §2.3 (Módulo 3 Weekly Hub).
- Contrato API: [technical-spec.md](../technical-spec.md) §2 Módulo 3 — `GET /api/v1/weekly/summary` (`start_date`, `end_date`).
- SSOT Pydantic: [api-contract-ssot.md](../api-contract-ssot.md) → en repo `grip-backend`, `app/schemas/weekly.py`.

## 3. Actores y permisos

- Gerente Regional, COO, CEO (mismos que hoy para Weekly).
- Sin nuevos roles.

## 4. Flujos principales

1. Usuario abre `/weekly`.
2. Introduce `store_code` y opcionalmente fechas inicio/fin.
3. Pulsa generar; el cliente llama `GET /api/v1/weekly/summary` con query params alineados al spec.
4. Si 4xx/5xx, mostrar mensaje y no vaciar el último resultado válido.

## 5. Reglas de negocio

1. Fechas fin ≥ fechas inicio cuando ambas están presentes.
2. Respetar R3: si el usuario es `ceo`, el backend ya consolida sin filtro regional en el servicio weekly (no reimplementar en el cliente).

## 6. Contrato API

- **Modificado:** `GET /api/v1/weekly/summary` — añadir en la spec de feature solo si se cambian params o respuesta; hoy: `start_date`, `end_date` opcionales (ver technical-spec §2).

## 7. Cambios en UI (frontend)

- `WeeklySummaryComponent`: controles de fecha + pasar params a `WeeklyService.getWeeklySummary`.
- `WeeklyService`: extender firma del método con `{ start_date?, end_date? }`.

## 8. Criterios de aceptación

- Dado un usuario autenticado con acceso weekly, cuando selecciona un rango válido y genera, entonces la petición incluye `start_date` y `end_date` en ISO8601 y la UI muestra el nuevo resumen.
- Dado un rango inválido (fecha fin anterior a fecha inicio), cuando intenta generar, entonces el formulario muestra error de validación y no llama al API.

## 9. Notas

- Tras implementar, actualizar criterios si el comportamiento real difiere; revisar [backend_audit_report.md](../backend_audit_report.md) solo si cambia contrato o reglas R*.
