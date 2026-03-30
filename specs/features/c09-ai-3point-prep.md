# C-09 — AI-02 3-Point Review Prep Brief

- **Estado:** borrador
- **Prioridad:** Core MVP
- **Depende de:** C-04 (Aspect Library & 3-Point Review, requerido), C-03 (Weekly Report Cascade), C-05 (Auto-Surface Engine)
- **SDD referencia:** §5.8.7 AI-02 (3-Point Review Prep), §5.8.3.1 AI-PREP-02, §5.4.3 Data Sources A-E, BR-AI-01 a BR-AI-05, BR-TP-07
- **Usa IA/Gemini:** Si
- **Ultima actualizacion:** 2026-03-29

---

## 1. Objetivo

- **Aplicacion(es):** grip-backend, grip-frontend
- **Problema:** Al preparar una revision de 3 puntos, el evaluador debe elegir que aspectos evaluar de una libreria extensa. Sin asistencia, la seleccion es ad-hoc, dejando aspectos criticos sin revisar durante meses y perdiendo la oportunidad de focalizar en areas con senales de deterioro.
- **Alcance:**
  - **grip-backend:** Endpoint que recopila historial de aspectos, scores de checklist, follow-ups y senales de reportes semanales, y los envia a Gemini para sugerir aspectos prioritarios con justificacion. Persistencia de sugerencias AI en `ThreePointReview.ai_suggested_aspects` (BR-TP-07).
  - **grip-frontend:** Pantalla de preparacion pre-review que muestra aspectos sugeridos con justificacion, alerta de cobertura y permite al evaluador aceptar o modificar la seleccion.
  - **Fuera de alcance:** Ejecucion de la review (C-04), auto-surface de hallazgos (C-05).

## 2. Referencia a specs base

- Modelo de prompt y output: `GRIP-SDD-Functional-Specs.md` §5.8.7 AI-02
- Preparacion asistida: `GRIP-SDD-Functional-Specs.md` §5.8.3.1 AI-PREP-02
- Data Sources A-E para 3-Point Review: `GRIP-SDD-Functional-Specs.md` §5.4.3
- Reglas de resiliencia IA: `GRIP-SDD-Functional-Specs.md` §5.8 (BR-AI-01 a BR-AI-05)
- Persistencia de sugerencias: `GRIP-SDD-Functional-Specs.md` §5.4 (BR-TP-07)

## 3. Actores y permisos

| Rol | Permiso |
|-----|---------|
| GTE_VENTAS | Solicitar prep brief para reviews de sus JZ |
| GERENTE_REGIONAL | Solicitar prep brief para reviews de empleados de su region |
| COO | Solicitar prep brief para reviews de Gerentes Regionales |
| CEO | Solicitar prep brief para reviews de cualquier empleado |

El actor debe tener relacion jerarquica con el `reviewee` via `reports_to` (§5.1.2).

## 4. Flujos principales

### 4.1 Solicitar brief de preparacion

1. Evaluador navega al perfil del empleado y selecciona "Preparar Review 3 Puntos".
2. Frontend llama `GET /ai/three-point-prep/{reviewee_id}`.
3. Backend verifica que el evaluador tiene relacion jerarquica directa con el reviewee.
4. Backend recopila el contexto (Data Sources §5.4.3):
   - **A.** `aspect_history_last_6_months`: historial de aspectos evaluados en los ultimos 6 meses para este empleado.
   - **B.** `aspects_never_evaluated`: aspectos de la libreria que nunca han sido evaluados para este empleado.
   - **C.** `aspects_with_low_scores`: aspectos con score <= 2 en la ultima evaluacion.
   - **D.** `reviewee_checklist_scores_last_60d`: scores de checklist del reviewee en los ultimos 60 dias.
   - **E.** `reviewee_open_followups`: follow-ups abiertos asignados al reviewee.
   - `reviewee_weekly_report_signals`: senales relevantes de los reportes semanales del reviewee.
5. Backend construye prompt segun §5.8.7 AI-02 (system prompt exacto definido en SDD) y envia a Gemini.
6. Backend parsea respuesta y la retorna al frontend.
7. Backend persiste las sugerencias en `ThreePointReview.ai_suggested_aspects` (BR-TP-07) para trazabilidad.

### 4.2 Fallback por fallo de Gemini

1. Si la llamada a Gemini falla, se activa fallback deterministico (BR-AI-03).
2. El fallback retorna aspectos nunca evaluados ordenados por fecha de ultima evaluacion (mas antiguos primero).
3. Se incluye flag `ai_generated: false` en la respuesta.

### 4.3 Aceptar o modificar sugerencias

1. Frontend muestra los aspectos sugeridos con `priority_rank`, `reason` y `supporting_evidence`.
2. El evaluador puede:
   - Aceptar la sugerencia tal cual y proceder a crear la review.
   - Reemplazar uno o mas aspectos de la sugerencia por otros de la libreria.
3. La seleccion final (sea AI o manual) se registra al crear la ThreePointReview.

## 5. Reglas de negocio

| ID | Regla |
|----|-------|
| BR-AI-01 | Toda llamada a Gemini debe incluir contexto estructurado; nunca enviar datos crudos sin preprocesar. |
| BR-AI-02 | Las sugerencias de IA son orientativas; el evaluador decide que aspectos evaluar. |
| BR-AI-03 | Un fallo de IA nunca bloquea la creacion de una review. Degradar a fallback deterministico. |
| BR-AI-05 | Los datos enviados a Gemini deben respetar el tenant isolation. |
| BR-TP-07 | Las sugerencias AI se persisten en `ThreePointReview.ai_suggested_aspects` para trazabilidad y analisis posterior. |

## 6. Contrato API

### `GET /ai/three-point-prep/{reviewee_id}`

**Request:** Path param `reviewee_id` (UUID).

**Response 200 (schema from §5.8.7 AI-02):**

```json
{
  "reviewee_id": "uuid",
  "generated_at": "datetime",
  "ai_generated": true,
  "suggested_aspects": [
    {
      "aspect_id": "uuid",
      "aspect_name": "string",
      "reason": "string",
      "supporting_evidence": ["string"],
      "priority_rank": 1
    }
  ],
  "coverage_alert": {
    "months_since_full_cycle": 0,
    "uncovered_categories": ["string"]
  },
  "data_sources": ["string"]
}
```

**Response 403:** Evaluador sin relacion jerarquica con el reviewee.

**Response 404:** Empleado no encontrado.

## 7. Cambios en UI (frontend)

- **Pantalla "Preparar Review 3 Puntos"** (`/employees/{reviewee_id}/review-prep`):
  - Lista de aspectos sugeridos, cada uno con:
    - Nombre del aspecto y ranking de prioridad (`priority_rank`).
    - Razon de la sugerencia (`reason`, texto AI).
    - Evidencia de soporte (`supporting_evidence`, lista colapsable).
  - Alerta de cobertura: si `months_since_full_cycle > 6` o hay `uncovered_categories`, mostrar banner de advertencia.
  - Selector para reemplazar aspectos sugeridos por otros de la libreria.
  - Banner de modo degradado cuando `ai_generated: false`.
  - Boton "Iniciar Review" que navega a C-04 con los aspectos seleccionados.
- **Guard:** Verificar relacion jerarquica evaluador-reviewee.

## 8. Criterios de aceptacion

- [ ] **Dado** un GTE_VENTAS con JZ asignados, **cuando** solicita prep brief para un JZ, **entonces** recibe `suggested_aspects` con aspectos priorizados y justificados (cada uno con `reason` y `supporting_evidence`).
- [ ] **Dado** un empleado con aspectos nunca evaluados, **cuando** se genera el brief, **entonces** el `coverage_alert` incluye las categorias no cubiertas (`uncovered_categories`).
- [ ] **Dado** que Gemini falla, **cuando** se solicita el brief, **entonces** se retorna fallback con aspectos nunca evaluados ordenados por antiguedad y `ai_generated: false`.
- [ ] **Dado** que se genera un brief exitosamente, **cuando** se crea una ThreePointReview para ese reviewee, **entonces** las sugerencias AI quedan persistidas en `ai_suggested_aspects` (BR-TP-07).
- [ ] **Dado** un evaluador sin relacion jerarquica con el reviewee, **cuando** solicita prep brief, **entonces** recibe 403.
- [ ] Toda llamada a Gemini esta envuelta en `try/except` capturando `GeminiAPIError`, `GeminiQuotaError`, `GeminiTimeoutError`.

## 9. Notas

- C-04 (Aspect Library & 3-Point Review) es prerequisito obligatorio; sin la libreria de aspectos no hay datos de entrada para el AI.
- El system prompt exacto esta definido en `GRIP-SDD-Functional-Specs.md` §5.8.7 AI-02. No duplicar aqui.
- `reviewee_weekly_report_signals` depende de C-03 (Weekly Report Cascade). Si C-03 no esta implementado, este campo se omite del contexto sin afectar el flujo.
