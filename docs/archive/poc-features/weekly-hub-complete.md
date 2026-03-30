# Weekly Hub Complete

> Completar la capa de presentacion del Modulo 3 (Hub Semanal) en grip-frontend: grafico de compliance, formulario de 3 preguntas, selector regional CEO y week number.

- **Estado:** en-implementacion
- **Spec base alineada:** v7.20 (version de `technical-spec.md`)
- **Ultima actualizacion:** 2026-03-28

## 1. Objetivo

- **Aplicacion(es):** **grip-backend** (2 endpoints auxiliares) | **grip-frontend** (4 gaps UI)
- **Problema o necesidad:** El backend del Hub Semanal (M3) esta 100% implementado pero el frontend solo muestra ~70% de la data. Falta: grafico de compliance, formulario de feedback estrategico, drill-down regional para CEO y uso de week_number.
- **Alcance:**
  - **grip-backend:** Agregar `GET /api/v1/zones` y `GET /api/v1/users` (read-only, prerequisitos para dropdowns).
  - **grip-frontend:** Renderizar `compliance_chart`, crear formulario 3 preguntas (`POST /weekly/feedback`), dropdown de region para CEO (R3), badge de semana ISO.
  - **Fuera de alcance:** CRUD de focal points (pertenece a Pre-Visita), gestion completa de `weekly_sessions`, sistema de notificaciones.

## 2. Referencia a specs base

- Reglas de negocio: [functional-spec.md](../functional-spec.md) §5 (Journey 2: Middle Management Analysis), §6.0 (R1-R6).
- Contrato y arquitectura: [technical-spec.md](../technical-spec.md) §2 Modulo 3 (endpoints weekly/summary y weekly/feedback).
- IA / Gemini: [technical-spec.md §3](../technical-spec.md) §3.C (Executive Summarizer) — ya implementado en backend.
- Regla R3 (Visibilidad CEO): [technical-spec.md §4](../technical-spec.md) — CEO bypasses regional filters.

## 3. Actores y permisos

| Rol | Acceso a /weekly | Region selector | Feedback form |
|-----|-----------------|-----------------|---------------|
| jz | No (guard) | — | — |
| gv | No (guard) | — | — |
| regional | Si | No | No |
| coo | Si | No | Si |
| ceo | Si | Si (R3) | Si |

- `GET /api/v1/zones`: cualquier usuario autenticado.
- `GET /api/v1/users`: solo `coo` y `ceo` (require_role).

## 4. Flujos principales

### 4.1 Visualizar compliance chart
1. Usuario genera resumen semanal (ya existente).
2. El backend retorna `compliance_chart: Record<string, number>` con porcentajes por categoria.
3. El frontend renderiza barras horizontales con codigo de color: verde (>=80%), ambar (>=60%), rojo (<60%).

### 4.2 Enviar feedback estrategico (3 preguntas)
1. COO o CEO genera un resumen semanal.
2. Debajo de los hallazgos aparece la seccion "Feedback Estrategico".
3. Completa las 3 preguntas: "Que paso?", "Por que paso?", "Que cambiamos?".
4. Selecciona un Gerente Regional destino del dropdown.
5. Envia → `POST /api/v1/weekly/feedback`.
6. Se muestra banner de exito y el formulario se deshabilita.
- **Excepcion:** Si no hay gerentes regionales en DB, el dropdown queda vacio y el submit esta deshabilitado.

### 4.3 CEO drill-down regional (R3)
1. CEO abre /weekly.
2. Ve un dropdown "Region" con todas las zonas (cargadas de `GET /api/v1/zones`).
3. Selecciona una region (o deja "Todas las regiones").
4. Al generar resumen, el `region_id` se pasa al backend para filtrar.

### 4.4 Week number
1. Al recibir respuesta del backend, se muestra el numero de semana ISO.
2. Si el backend no lo retorna, se calcula desde `start_date`.

## 5. Reglas de negocio

1. El grafico de compliance solo se renderiza si `compliance_chart` tiene al menos una entrada.
2. El formulario de feedback requiere exactamente 3 preguntas no vacias y un `target_user_id` seleccionado.
3. El dropdown de region solo es visible para rol `ceo` (R3).
4. El feedback form solo es visible para roles `coo` y `ceo`.
5. Tras enviar feedback exitosamente, el formulario se deshabilita para evitar doble envio.

## 6. Contrato API

### Endpoints nuevos

| Metodo | Path | Request | Response |
|--------|------|---------|----------|
| GET | `/api/v1/zones` | — | `{ items: [{ id: UUID, name: str\|null }] }` |
| GET | `/api/v1/users?role=regional` | query: `role` (optional) | `{ items: [{ id: UUID, name: str\|null, email: str, role: str }] }` |

### Endpoints existentes consumidos

| Metodo | Path | Campos nuevos en params |
|--------|------|------------------------|
| GET | `/api/v1/weekly/summary` | `region_id` (CEO), `week_number` |
| POST | `/api/v1/weekly/feedback` | (sin cambios, ya existente) |

## 7. Cambios en UI (frontend)

| Componente | Tipo | Descripcion |
|-----------|------|-------------|
| `ComplianceChartComponent` | Nuevo | Barras CSS horizontales, input `data: Record<string,number>` |
| `WeeklyFeedbackComponent` | Nuevo | Formulario reactivo 3 textareas + select regional + submit |
| `WeeklySummaryComponent` | Modificado | Importa hijos, agrega dropdown region (CEO), badge semana, secciones chart y feedback |
| `WeeklyService` | Modificado | +3 metodos (`getZones`, `getUsers`, `submitFeedback`), params extendidos |

Rutas: sin cambios (`/weekly` con guard existente).

## 8. Criterios de aceptacion

- [ ] **CA-1:** Dado un resumen con `compliance_chart` no vacio, cuando se renderiza, entonces se muestran barras horizontales con porcentaje y color por categoria.
- [ ] **CA-2:** Dado un usuario con rol `coo` o `ceo`, cuando genera un resumen, entonces ve el formulario de 3 preguntas debajo de los hallazgos.
- [ ] **CA-3:** Dado que completa las 3 preguntas y selecciona un gerente regional, cuando envia, entonces recibe confirmacion y el formulario se deshabilita.
- [ ] **CA-4:** Dado un usuario con rol `ceo`, cuando abre /weekly, entonces ve un dropdown de regiones cargado desde `GET /zones`.
- [ ] **CA-5:** Dado un usuario con rol `regional`, cuando abre /weekly, entonces NO ve ni el dropdown de region ni el formulario de feedback.
- [ ] **CA-6:** Dado un `start_date`, cuando se genera el resumen, entonces se muestra el numero de semana ISO.
- [ ] **CA-7:** `ng build` compila sin errores.

## 9. Notas

- Los endpoints `GET /zones` y `GET /users` son genericos y podran reutilizarse en futuras features (Pre-Visita, DA).
- El `session_id` para feedback se genera client-side con `crypto.randomUUID()`; el backend auto-crea la session si no existe.
- No se instalo ninguna libreria de graficos — el chart es CSS puro con Tailwind.
