# C-05 — Auto-Surface Engine

- **Estado:** borrador
- **Prioridad:** Core MVP
- **Depende de:** B-07 (FollowUpItem entity), C-01 (Checklist digital), C-03 (Weekly Report cascade), C-04 (ThreePointReview)
- **SDD referencia:** GRIP-SDD-Functional-Specs.md §5.6 (FU-09, FU-09b, FU-09c, FU-09d, BR-FU-07 a BR-FU-09), §5.4.5 Efecto 2, §5.5.5 Efecto 2
- **Ultima actualizacion:** 2026-03-29

---

## 1. Objetivo

Implementar el motor de auto-resurface que garantiza que ningun hallazgo (FollowUpItem) sea olvidado. El motor marca automaticamente items de follow-up para que resurjan en el siguiente checklist, la siguiente revision de 3 puntos, o la siguiente auditoria DA, segun corresponda. Incluye audit trail de resurgimientos y alimenta el KPI de "Forgotten Findings" (BR-DB-03).

## 2. Referencia a specs base

Toda la especificacion normativa vive en **GRIP-SDD-Functional-Specs.md**:

| Seccion | Contenido |
|---------|-----------|
| §5.6 FU-09 | Auto-surface en siguiente checklist |
| §5.6 FU-09b | Auto-surface en siguiente 3-Point Review |
| §5.6 FU-09c | Auto-surface en siguiente DA |
| §5.6 FU-09d | Audit trail de resurgimientos (surfaced_in[]) |
| §5.4.5 Efecto 2 | Feedback loop de 3-Point Review a Follow-Up |
| §5.5.5 Efecto 2 | Triple resurface de DA findings |
| §5.7 BR-DB-03 | KPI Forgotten Findings |

## 3. Actores y permisos

| Actor | Acciones permitidas |
|-------|---------------------|
| Sistema (job automatico) | Evaluar y marcar FollowUpItems para resurface |
| JZ / JL | Ver items resurgidos en sus checklists y reviews |
| GV / GL | Ver Forgotten Findings KPI en dashboard |
| Regional / COO | Ver Forgotten Findings agregado en dashboard |

## 4. Flujos principales

### 4.1 Tres mecanismos de resurface

#### 4.1.1 Resurface en siguiente checklist (FU-09)

1. Al crear una ChecklistSession (C-01), el sistema consulta FollowUpItems con `auto_surface_in_next_checklist = true` para la tienda.
2. Los items que cumplen la condicion se inyectan como variable items con source = `OPEN_FOLLOWUP` (conecta con C-02).
3. Se registra el resurface en `surfaced_in[]` del FollowUpItem (FU-09d).

#### 4.1.2 Resurface en siguiente 3-Point Review (FU-09b)

1. Al iniciar una ThreePointReview (C-04), el sistema consulta FollowUpItems con `auto_surface_in_next_review = true` para el colaborador revisado.
2. Los aspectos relacionados se priorizan en la sugerencia de seleccion (fuente B de §5.4.3).
3. Se registra el resurface en `surfaced_in[]` (FU-09d).

#### 4.1.3 Resurface en siguiente DA (FU-09c)

1. Al preparar una DAAudit (§5.5), el sistema consulta FollowUpItems con `auto_surface_in_next_da = true` para la tienda.
2. Los items se incluyen en el brief de preparacion de DA.
3. Se registra el resurface en `surfaced_in[]` (FU-09d).
4. Triple resurface para DA findings (§5.5.5 Efecto 2): un hallazgo de DA resurge en checklist, 3-Point Review Y siguiente DA simultaneamente.

### 4.2 Evaluacion de flags de resurface

El motor evalua que flags activar segun:

1. **source_type del FollowUpItem:**
   - `CHECKLIST` -> `auto_surface_in_next_checklist = true` (BR-FU-07).
   - `THREE_POINT_REVIEW` -> `auto_surface_in_next_review = true` (BR-FU-08).
   - `DA_AUDIT` -> los tres flags en true (triple resurface, BR-FU-09).
   - `WEEKLY_REPORT` -> `auto_surface_in_next_checklist = true`.
2. **Status del item:** Solo items en status OPEN o IN_PROGRESS se marcan para resurface.
3. Los flags se activan al crear el FollowUpItem y se desactivan cuando el item es RESOLVED o VERIFIED.

### 4.3 Audit trail (FU-09d)

1. Cada vez que un FollowUpItem resurge en un contexto, se agrega un registro a `surfaced_in[]`.
2. El registro incluye: tipo de contexto (CHECKLIST, THREE_POINT_REVIEW, DA), ID del contexto, fecha.
3. Esto permite rastrear cuantas veces ha resurgido un hallazgo sin ser resuelto.

### 4.4 Forgotten Findings KPI (BR-DB-03)

1. Un FollowUpItem se considera "forgotten" si: status = OPEN y no ha tenido actualizacion en mas de N dias (configurable).
2. Este KPI es visible desde nivel GV+ en dashboards (DB-08, C-07).
3. El conteo de forgotten findings se calcula como query sobre FollowUpItems.

## 5. Reglas de negocio

| ID | Regla | Origen SDD |
|----|-------|------------|
| BR-FU-07 | FollowUpItem de checklist resurge en siguiente checklist de la misma tienda | §5.6 BR-FU-07 |
| BR-FU-08 | FollowUpItem de 3-Point Review resurge en siguiente review del mismo colaborador | §5.6 BR-FU-08 |
| BR-FU-09 | FollowUpItem de DA resurge en los tres contextos simultaneamente (triple resurface) | §5.6 BR-FU-09 |

## 6. Contrato API

Base path: `/api/v1/`

| Metodo | Endpoint | Descripcion |
|--------|----------|-------------|
| GET | `/follow-ups/surface/checklist/{store_id}` | Obtener items a resurgir en siguiente checklist de la tienda |
| GET | `/follow-ups/surface/review/{employee_id}` | Obtener items a resurgir en siguiente 3-Point Review del colaborador |
| GET | `/follow-ups/surface/da/{store_id}` | Obtener items a resurgir en siguiente DA de la tienda |
| GET | `/follow-ups/{item_id}/surface-history` | Obtener audit trail de resurgimientos |
| GET | `/follow-ups/forgotten` | Listar Forgotten Findings (OPEN + sin actualizacion > N dias) |
| PATCH | `/follow-ups/{item_id}/surface-flags` | Actualizar flags de resurface manualmente |

### Schemas principales

**SurfaceResult:**
```json
{
  "store_id": "uuid",
  "context_type": "CHECKLIST | THREE_POINT_REVIEW | DA",
  "items": [
    {
      "follow_up_id": "uuid",
      "title": "string",
      "source_type": "CHECKLIST | THREE_POINT_REVIEW | DA_AUDIT | WEEKLY_REPORT",
      "status": "OPEN | IN_PROGRESS",
      "priority": "HIGH | MEDIUM | LOW",
      "created_at": "datetime",
      "times_surfaced": 3
    }
  ]
}
```

**SurfaceHistoryEntry:**
```json
{
  "context_type": "CHECKLIST | THREE_POINT_REVIEW | DA",
  "context_id": "uuid",
  "surfaced_at": "datetime"
}
```

**ForgottenFindingsResponse:**
```json
{
  "total_count": 15,
  "items": [
    {
      "follow_up_id": "uuid",
      "title": "string",
      "days_without_update": 12,
      "assigned_to": "uuid",
      "store_id": "uuid"
    }
  ]
}
```

## 7. Cambios en UI (frontend)

| Pantalla | Descripcion |
|----------|-------------|
| Follow-Up List | Badge indicando cuantas veces ha resurgido un item (times_surfaced) |
| Checklist Execution (C-01) | Seccion de variable items resurgidos con badge "Resurface" |
| 3-Point Review (C-04) | Indicador de aspectos sugeridos por follow-ups abiertos |
| Forgotten Findings Widget | Widget de dashboard (C-07) mostrando conteo y lista de findings olvidados |
| Surface History | Timeline de resurgimientos por FollowUpItem |

## 8. Criterios de aceptacion

- [ ] Al crear ChecklistSession, se obtienen FollowUpItems con auto_surface_in_next_checklist = true para la tienda (FU-09).
- [ ] Al iniciar ThreePointReview, se priorizan aspectos con follow-ups abiertos (FU-09b).
- [ ] Al preparar DA, se incluyen FollowUpItems con auto_surface_in_next_da = true (FU-09c).
- [ ] Items de DA activan los tres flags simultaneamente (BR-FU-09, triple resurface).
- [ ] Cada resurface se registra en surfaced_in[] con tipo, ID y fecha (FU-09d).
- [ ] Forgotten Findings KPI calcula correctamente items OPEN sin actualizacion > N dias (BR-DB-03).
- [ ] Los flags se desactivan automaticamente cuando el item pasa a RESOLVED o VERIFIED.
- [ ] Endpoints protegidos con JWT (B-04) y tenant isolation (B-03).

## 9. Notas

- B-07 ya creo la tabla FollowUpItem (migracion 0017) con campos status, priority, deadline, source_type, assigned_to y escalated. C-05 agrega los campos auto_surface_in_next_checklist, auto_surface_in_next_review, auto_surface_in_next_da y surfaced_in[].
- La tabla surfaced_in puede implementarse como JSONB array o como tabla separada FollowUpSurfaceLog; decision de implementacion a definir en el backend.
- El parametro N (dias sin actualizacion para Forgotten Findings) debe ser configurable por tenant.
- C-05 es consumido por C-01 (variable items OPEN_FOLLOWUP), C-02 (inyeccion), C-04 (fuente B de seleccion de aspectos), y C-07 (Forgotten Findings widget).
