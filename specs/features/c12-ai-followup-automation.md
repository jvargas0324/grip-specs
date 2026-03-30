# C-12 — AI-06 Follow-Up Automation

- **Estado:** borrador
- **Prioridad:** Core MVP
- **Depende de:** C-05 (Auto-Surface Engine), C-06 (Escalation Automation)
- **SDD referencia:** §5.8.7 AI-06 (Follow-Up Automation), §5.8.3.4 AI-FUA-01 a AI-FUA-03, BR-AI-01 a BR-AI-05
- **Usa IA/Gemini:** Si
- **Ultima actualizacion:** 2026-03-29

---

## 1. Objetivo

- **Aplicacion(es):** grip-backend, grip-frontend
- **Problema:** La gestion de follow-ups requiere revision manual constante para identificar vencidos, proximos a vencer y candidatos a escalacion. Sin automatizacion, los items se pierden en el volumen y los supervisores carecen de un resumen ejecutivo para actuar rapidamente.
- **Alcance:**
  - **grip-backend:** 3 capacidades del SDD: (AI-FUA-01) resumen diario de overdue, (AI-FUA-02) redaccion de recordatorios para items proximos a vencer, (AI-FUA-03) sugerencias de escalacion para items estancados. Job diario + endpoint on-demand. Fallback deterministico si Gemini falla.
  - **grip-frontend:** Panel de automatizacion de follow-ups con resumen, items overdue, proximos a vencer y borradores de escalacion.
  - **Fuera de alcance:** Envio automatico de notificaciones (requiere feature de notificaciones post-MVP), ejecucion automatica de escalaciones (requiere aprobacion humana).

## 2. Referencia a specs base

- Modelo de prompt y output: `GRIP-SDD-Functional-Specs.md` §5.8.7 AI-06
- 3 capacidades de automatizacion: `GRIP-SDD-Functional-Specs.md` §5.8.3.4 (AI-FUA-01 a AI-FUA-03)
- Reglas de resiliencia IA: `GRIP-SDD-Functional-Specs.md` §5.8 (BR-AI-01 a BR-AI-05)
- Follow-Up lifecycle: `GRIP-SDD-Functional-Specs.md` §5.6 (BR-FU-01 a BR-FU-09)

## 3. Actores y permisos

| Rol | Permiso |
|-----|---------|
| JEFE_ZONA | Ver resumen de follow-ups de sus tiendas |
| GTE_VENTAS | Ver resumen de follow-ups de sus JZ y tiendas |
| GERENTE_REGIONAL | Ver resumen de follow-ups de su region |
| COO | Ver resumen de follow-ups operativos |
| CEO | Ver resumen de todos los follow-ups |

Los datos se filtran por la visibilidad jerarquica del usuario (cascada `GRIP-SDD-Functional-Specs.md` §5.1.3 EMP-05).

## 4. Flujos principales

### 4.1 Job diario de automatizacion

El job se ejecuta diariamente y procesa las 3 capacidades. Context inputs del SDD §5.8.7 AI-06:
- `overdue_items`: FollowUpItems con deadline pasada, status OPEN o IN_PROGRESS.
- `due_soon_items`: FollowUpItems cuya deadline esta dentro de los proximos 7 dias.
- `stalled_items`: FollowUpItems con status IN_PROGRESS y sin actividad en >7 dias.
- `escalation_candidates`: FollowUpItems abiertos >2 ciclos sin resolucion.

**AI-FUA-01 — Overdue Summary (resumen diario):**
1. Recopilar `overdue_items`, agrupar por owner y tienda.
2. Enviar a Gemini para generar `summary_headline` y lista priorizada de `overdue` con `action_required` por item.

**AI-FUA-02 — Reminder Drafting (items proximos a vencer):**
1. Recopilar `due_soon_items` (deadline dentro de 7 dias).
2. Enviar a Gemini para generar mensajes de recordatorio contextualizados.
3. Retornar como `due_soon` con `days_remaining`.

**AI-FUA-03 — Escalation Suggestion (items estancados):**
1. Identificar `stalled_items` (IN_PROGRESS sin actividad >7 dias) y `escalation_candidates` (abiertos >2 ciclos).
2. Enviar a Gemini para generar borradores de mensaje de escalacion con destinatario sugerido.
3. Retornar como `escalation_drafts` con `draft_message` y `escalate_to`.

### 4.2 Consulta on-demand (API)

1. Usuario accede al panel de automatizacion de follow-ups.
2. Frontend llama `GET /ai/followup-automation`.
3. Backend retorna el ultimo resultado del job diario, filtrado por visibilidad del usuario.
4. Si no hay resultado reciente, ejecuta generacion on-demand.

### 4.3 Fallback por fallo de Gemini

1. Si Gemini falla, se activa fallback deterministico (BR-AI-03).
2. El fallback retorna:
   - `summary_headline`: template mecanico ("X items vencidos, Y proximos a vencer").
   - `overdue`: lista ordenada por deadline ascendente (mas antiguos primero).
   - `due_soon`: lista ordenada por `days_remaining` ascendente.
   - `escalation_drafts`: items candidatos sin borrador de mensaje (solo datos).
3. Flag `ai_generated: false` en la respuesta.

## 5. Reglas de negocio

| ID | Regla |
|----|-------|
| AI-FUA-01 | Resumen diario de items overdue, agrupados por owner y tienda. |
| AI-FUA-02 | Recordatorios para items cuya deadline esta dentro de 7 dias. |
| AI-FUA-03 | Sugerencia de escalacion para items IN_PROGRESS sin actividad >7 dias o abiertos >2 ciclos sin resolucion. |
| BR-AI-01 | Contexto estructurado para Gemini; nunca datos crudos. |
| BR-AI-02 | Los borradores de escalacion son sugerencias; el usuario decide si enviar. |
| BR-AI-03 | Fallo de IA no bloquea la generacion del resumen. Degradar a fallback deterministico. |
| BR-AI-05 | Tenant isolation en datos enviados a Gemini. |

## 6. Contrato API

### `GET /ai/followup-automation`

**Query params:**
- `scope` (opcional): `STORE | ZONE | REGION | ALL` (default inferido del rol).

**Response 200 (schema from §5.8.7 AI-06):**

```json
{
  "generated_at": "datetime",
  "ai_generated": true,
  "summary_headline": "string",
  "overdue": [
    {
      "item_id": "uuid",
      "description": "string",
      "owner": "string",
      "aging_days": 0,
      "action_required": "string"
    }
  ],
  "due_soon": [
    {
      "item_id": "uuid",
      "description": "string",
      "days_remaining": 0
    }
  ],
  "escalation_drafts": [
    {
      "item_id": "uuid",
      "draft_message": "string",
      "escalate_to": "string"
    }
  ],
  "data_sources": ["string"]
}
```

**Response 403:** Usuario sin permisos.

### `POST /ai/followup-automation/run` (admin/internal)

Trigger manual del job diario. Solo CEO o admin.

**Response 202:** Job enqueued.

## 7. Cambios en UI (frontend)

- **Panel "Automatizacion Follow-Ups"** (`/followups/automation`):
  - Header con `summary_headline` (texto resumen generado por AI o template mecanico).
  - Seccion "Vencidos" con tabla: descripcion, owner, dias de atraso (`aging_days`), accion requerida. Ordenados por aging descendente.
  - Seccion "Proximos a Vencer" con tabla: descripcion, dias restantes (`days_remaining`). Ordenados por urgencia.
  - Seccion "Sugerencias de Escalacion" con:
    - Item description.
    - Borrador de mensaje de escalacion (`draft_message`, editable por el usuario).
    - Destinatario sugerido (`escalate_to`).
    - Boton "Enviar Escalacion" (placeholder hasta feature de notificaciones).
  - Banner de modo degradado cuando `ai_generated: false`.
- **Widget en dashboard principal (C-07):**
  - Contador: "X vencidos, Y proximos a vencer, Z para escalar".
  - Click navega al panel completo.
- **Guard:** Visibilidad filtrada por jerarquia.

## 8. Criterios de aceptacion

- [ ] **Dado** que existen follow-ups vencidos, **cuando** se ejecuta el job o se consulta la API, **entonces** la respuesta incluye `overdue` con `aging_days` y `action_required` para cada item (AI-FUA-01).
- [ ] **Dado** que existen follow-ups con deadline en los proximos 7 dias, **cuando** se genera el resumen, **entonces** la respuesta incluye `due_soon` con `days_remaining` (AI-FUA-02).
- [ ] **Dado** un follow-up IN_PROGRESS sin actividad en >7 dias, **cuando** se ejecuta el job, **entonces** aparece en `escalation_drafts` con borrador de mensaje (AI-FUA-03).
- [ ] **Dado** un follow-up abierto >2 ciclos sin resolucion, **cuando** se ejecuta el job, **entonces** aparece en `escalation_drafts` como candidato a escalacion (AI-FUA-03).
- [ ] **Dado** que Gemini falla, **cuando** se genera el resumen, **entonces** se retorna fallback con lista ordenada por deadline ascendente y `ai_generated: false`.
- [ ] **Dado** un GTE_VENTAS, **cuando** consulta el resumen, **entonces** solo ve follow-ups de entidades dentro de su scope jerarquico (EMP-05).
- [ ] Los borradores de escalacion son editables por el usuario antes de cualquier accion.
- [ ] Toda llamada a Gemini esta envuelta en `try/except` capturando `GeminiAPIError`, `GeminiQuotaError`, `GeminiTimeoutError`.

## 9. Notas

- El envio efectivo de notificaciones/escalaciones no esta en MVP. Los borradores se generan pero la accion de "enviar" es placeholder hasta que exista el feature de notificaciones.
- El system prompt exacto esta definido en `GRIP-SDD-Functional-Specs.md` §5.8.7 AI-06. No duplicar aqui.
- C-06 (Escalation Automation) define las reglas de escalacion por jerarquia. Este spec usa esas reglas para identificar candidatos y sugerir `escalate_to`.
- La frecuencia del job (diaria por defecto) es configurable via variable de entorno.
