# C-06 — Escalation Automation

- **Estado:** borrador
- **Prioridad:** Core MVP
- **Depende de:** B-07 (FollowUpItem entity), B-02 (reports_to hierarchy), C-05 (Auto-surface engine)
- **SDD referencia:** GRIP-SDD-Functional-Specs.md §5.6 (FU-07, BR-FU-05), §5.7 (dashboard escalation widgets)
- **Ultima actualizacion:** 2026-03-29

---

## 1. Objetivo

Implementar la escalacion automatica de FollowUpItems que superan su deadline. Segun BR-FU-05, la escalacion automatica ocurre a las 24 horas despues del deadline. El sistema crea un nuevo FollowUpItem asignado al manager (reports_to) del responsable original. El proceso se ejecuta como nightly job idempotente.

## 2. Referencia a specs base

Toda la especificacion normativa vive en **GRIP-SDD-Functional-Specs.md**:

| Seccion | Contenido |
|---------|-----------|
| §5.6 FU-07 | Mecanismo de escalacion automatica |
| §5.6 BR-FU-05 | Regla: escalacion a 24h despues del deadline |
| §5.7 | Widgets de escalacion en dashboards por rol |
| §8.4 | Seguridad: permisos de escalacion |

## 3. Actores y permisos

| Actor | Acciones permitidas |
|-------|---------------------|
| Sistema (nightly job) | Evaluar FollowUpItems vencidos y crear escalaciones |
| JZ / JL | Recibir escalaciones de JTs bajo su mando |
| GV / GL | Recibir escalaciones de JZs/JLs; ver widget de escalaciones en dashboard |
| Regional | Recibir escalaciones de GVs/GLs |
| COO | Recibir escalaciones de Regionales; ver escalaciones nacionales |

## 4. Flujos principales

### 4.1 Nightly job de escalacion

1. El job se ejecuta una vez al dia (configuracion cron).
2. Consulta FollowUpItems donde:
   - `status` IN (`OPEN`, `IN_PROGRESS`)
   - `deadline` < NOW() - 24 horas
   - `escalated` = false
3. Para cada item que cumple la condicion:
   a. Marca el item original con `status = ESCALATED` y `escalated = true`.
   b. Resuelve el manager del `assigned_to` usando la jerarquia reports_to (B-02).
   c. Crea un **nuevo** FollowUpItem asignado al manager con:
      - `source_type` = mismo que el original
      - `priority` = escalada (sube un nivel si no es ya HIGH)
      - `assigned_to` = manager (reports_to del responsable original)
      - Referencia al item original (parent_id o escalated_from_id)
   d. Registra la escalacion en el audit trail.

### 4.2 Idempotencia

1. El flag `escalated = true` en el item original previene re-escalaciones.
2. Si el job se ejecuta multiples veces en el mismo dia, no genera duplicados.
3. Si un item ya fue escalado manualmente, el job lo omite.

### 4.3 Cadena de escalacion

1. Si el nuevo item escalado tambien supera su deadline + 24h sin resolucion, se escalara nuevamente al siguiente nivel en la proxima ejecucion del job.
2. La cadena sube por la jerarquia: JT -> JZ/JL -> GV/GL -> Regional -> COO.
3. Si no hay manager superior (ej: COO sin reports_to), el item se marca como escalacion terminal y se refleja en dashboard.

### 4.4 Notificacion de escalacion

1. Al crear el item escalado, se genera una notificacion para el manager receptor.
2. La notificacion incluye: item original, responsable original, dias de atraso, prioridad.

## 5. Reglas de negocio

| ID | Regla | Origen SDD |
|----|-------|------------|
| BR-FU-05 | La escalacion automatica ocurre a las 24 horas despues del deadline del FollowUpItem | §5.6 BR-FU-05 |

## 6. Contrato API

Base path: `/api/v1/`

| Metodo | Endpoint | Descripcion |
|--------|----------|-------------|
| GET | `/follow-ups/escalated` | Listar items escalados recibidos por el usuario autenticado |
| GET | `/follow-ups/{item_id}/escalation-chain` | Ver cadena completa de escalacion de un item |
| POST | `/follow-ups/{item_id}/escalate` | Escalacion manual (para supervisores que quieran escalar antes de las 24h) |
| POST | `/admin/jobs/escalation/run` | Trigger manual del nightly job (solo admin, para testing) |

### Schemas principales

**EscalatedItemResponse:**
```json
{
  "id": "uuid",
  "original_item_id": "uuid",
  "title": "string",
  "source_type": "CHECKLIST | THREE_POINT_REVIEW | DA_AUDIT | WEEKLY_REPORT",
  "priority": "HIGH | MEDIUM | LOW",
  "status": "OPEN",
  "assigned_to": "uuid",
  "original_assigned_to": "uuid",
  "original_deadline": "datetime",
  "days_overdue": 5,
  "escalated_at": "datetime"
}
```

**EscalationChain:**
```json
{
  "root_item_id": "uuid",
  "chain": [
    {
      "item_id": "uuid",
      "assigned_to": "uuid",
      "assigned_to_name": "string",
      "role": "JT",
      "status": "ESCALATED",
      "escalated_at": "datetime"
    },
    {
      "item_id": "uuid",
      "assigned_to": "uuid",
      "assigned_to_name": "string",
      "role": "JZ",
      "status": "OPEN",
      "escalated_at": "datetime"
    }
  ]
}
```

**JobRunResult:**
```json
{
  "job_name": "escalation",
  "run_at": "datetime",
  "items_evaluated": 42,
  "items_escalated": 7,
  "errors": 0
}
```

## 7. Cambios en UI (frontend)

| Pantalla | Descripcion |
|----------|-------------|
| Escalation Inbox | Lista de items escalados recibidos, ordenados por prioridad y dias de atraso |
| Escalation Badge | Badge en nav principal con conteo de escalaciones pendientes |
| Escalation Chain View | Timeline visual de la cadena de escalacion (quien -> quien -> quien) |
| Dashboard Widget (C-07) | Widget de escalaciones activas por nivel en dashboards de GV+ |

### Requisitos UX

- Las escalaciones se destacan visualmente con color rojo/naranja segun prioridad.
- La cadena de escalacion se muestra como timeline vertical.
- Notificacion push o badge al recibir nueva escalacion.

## 8. Criterios de aceptacion

- [ ] El nightly job identifica correctamente FollowUpItems vencidos > 24h (BR-FU-05).
- [ ] Se crea un nuevo FollowUpItem asignado al manager via reports_to (FU-07, B-02).
- [ ] El item original se marca como ESCALATED y escalated = true.
- [ ] El job es idempotente: ejecutar dos veces no genera duplicados.
- [ ] La cadena de escalacion sube correctamente por la jerarquia organizacional.
- [ ] La escalacion manual funciona para supervisores que quieran escalar antes de las 24h.
- [ ] Las escalaciones se reflejan en los dashboards de nivel GV+ (C-07).
- [ ] Endpoints protegidos con JWT (B-04) y tenant isolation (B-03).

## 9. Notas

- B-07 ya creo la tabla FollowUpItem con campo `escalated` (migracion 0017). C-06 agrega la logica del job y el campo `escalated_from_id` para la cadena.
- B-02 proporciona los endpoints /tree, /ancestors y /scope para resolver la jerarquia reports_to.
- El SDD especifica 24 horas despues del deadline, no un periodo generico. Esto es un requisito exacto de BR-FU-05.
- El nightly job debe implementarse como tarea programada (ej: Celery beat, APScheduler, o cron del sistema). La eleccion es decision de implementacion del backend.
- Los widgets de escalacion en dashboards se especifican en §5.7 y se implementan en C-07.
