# B-05 — DA Module: COO Scope + Refactor

- **Estado:** en-implementación
- **Prioridad:** 🔴 Bloqueante MVP
- **Depende de:** B-01 (Role.da_scope), B-04 (Auth JWT)
- **SDD referencia:** Sección 5.5 — DA Module (Dienstaufsicht)
- **Última actualización:** 2026-03-29
- **Inicio implementación:** 2026-03-29

---

## 1. Objetivo

- **Aplicación(es):** grip-backend
- **Problema:** `da_service.py` tiene hardcodeado `if current_user.role != "ceo": raise 403`. El COO también puede ejecutar DAs (scope: OPERATIONS_ONLY). Además, el DA actual no genera FollowUpItems ni tiene auto-surface entre ciclos.
- **Alcance:**
  - Reemplazar el guard hardcodeado por el permission flag `can_execute_da` del Role
  - Implementar `da_scope` para limitar el alcance del COO a áreas operacionales
  - Añadir auto-surface: findings de un DA resurgen en el siguiente DA al mismo Gerente Regional
  - **Queda fuera:** FollowUpItem completo (B-07), AI-03 DA Prep Brief (backlog core)

---

## 2. Referencia a specs base

- SDD Master: `specs/GRIP-SDD-Functional-Specs.md` — Sección 5.5.3 (Requirements), 5.5.4 (Business Rules)
- Gap Analysis: Módulo 5.5 en `specs/GRIP-GapAnalysis.md`

---

## 3. Reglas de negocio

1. **RB-01:** Solo employees con `role.can_execute_da = TRUE` pueden crear un DA. Actualmente: CEO y COO.
2. **RB-02:** CEO tiene `da_scope = ALL_AREAS` → puede auditar cualquier Gerente Regional de cualquier área.
3. **RB-03:** COO tiene `da_scope = OPERATIONS_ONLY` → solo puede auditar Gerentes Regionales cuya área funcional sea `Ventas` o `Logistica`.
4. **RB-04:** El target de un DA siempre es un Gerente Regional (`role.code = GTE_REGIONAL`). No se puede crear un DA dirigido a otro rol.
5. **RB-05:** Un DA solo puede tener status `active` si no existe otro DA `active` para el mismo `target_regional_id`. Se pueden abrir nuevos DAs solo cuando el anterior está cerrado.
6. **RB-06:** Los findings abiertos del DA anterior al mismo regional se marcan con `auto_surface_in_next_da = TRUE` en el FollowUpSystem — resurgen automáticamente al abrir el siguiente DA.
7. **RB-07:** El cierre de un DA requiere firma doble: `regional_signature_hash` + firma del ejecutor (CEO o COO).

---

## 4. Cambios en `da_service.py`

### Reemplazar guard hardcodeado

```python
# ANTES (eliminar):
if current_user.role != "ceo":
    raise HTTPException(403, "Only CEO can create DA interventions")

# DESPUÉS:
if not current_employee.role.can_execute_da:
    raise HTTPException(403, "Role does not have DA execution permission")
```

### Validar da_scope para COO

```python
if current_employee.role.da_scope == "OPERATIONS_ONLY":
    regional = db.get(Employee, payload.target_regional_id)
    if regional.functional_area not in ("Ventas", "Logistica"):
        raise HTTPException(403, "COO DA scope limited to Operations areas")
```

### Auto-surface de findings anteriores

Al crear un nuevo DA para un `target_regional_id`, el sistema busca findings abiertos del DA anterior al mismo regional y los añade al contexto del nuevo DA:

```python
previous_open_findings = db.execute(
    select(Finding)
    .join(DAFindingsMap, Finding.id == DAFindingsMap.finding_id)
    .join(DAIntervention, DAFindingsMap.da_id == DAIntervention.id)
    .where(
        DAIntervention.target_regional_id == payload.target_regional_id,
        Finding.status.in_(["open", "in_progress"]),
    )
).all()
# Estos findings se linkean al nuevo DA y se incluyen en el contexto del prompt AI-03
```

---

## 5. Contrato API (cambios)

| Endpoint | Cambio |
|---|---|
| `POST /api/v1/da/create` | Guard: `can_execute_da` en lugar de `role == "ceo"`. Añadir validación `da_scope`. |
| `PATCH /api/v1/da/close` | Firma del ejecutor (CEO o COO) en lugar de `ceo_signature_hash` fijo. Renombrar campo. |
| `GET /api/v1/da/{id}` | Nuevo endpoint: retorna DA con findings vinculados y findings auto-surfaced del ciclo anterior |
| `GET /api/v1/da` | Nuevo endpoint: lista DAs con filtro por `target_regional_id`, `status`, `creator_id` |

**Request `POST /da/create` actualizado:**
```json
{
  "target_regional_id": "uuid",
  "store_id": "uuid",
  "due_date": "2026-04-30",
  "area_scope": "Ventas"
}
```

**Response `GET /da/{id}`:**
```json
{
  "id": "uuid",
  "case_number": 1,
  "creator": { "id": "uuid", "full_name": "Ana CEO", "role_code": "CEO" },
  "target_regional": { "id": "uuid", "full_name": "Carlos Regional" },
  "status": "active",
  "proven_facts": "...",
  "root_cause_analysis": "...",
  "due_date": "2026-04-30",
  "linked_findings": [...],
  "surfaced_from_previous_da": [...]
}
```

---

## 6. Orden de implementación

```
1. services/da_service.py   ← reemplazar guard + añadir da_scope validation + auto-surface logic
2. schemas/da.py            ← actualizar DACreateRequest, añadir DAResponse completo
3. api/da.py                ← añadir GET /da y GET /da/{id}
```

---

## 7. Criterios de aceptación

- [x] **CA-01:** `POST /da/create` con token de COO y target regional de área "Ventas" → 201 Created
- [x] **CA-02:** `POST /da/create` con token de COO y target regional de área "Admin" → 403
- [x] **CA-03:** `POST /da/create` con token de JEFE_ZONA → 403
- [x] **CA-04:** `POST /da/create` cuando ya existe un DA `active` para el mismo regional → 409
- [x] **CA-05:** Al crear DA, findings abiertos del DA anterior al mismo regional aparecen en `surfaced_from_previous_da`
- [x] **CA-06:** `GET /da/{id}` retorna findings vinculados y findings auto-surfaced
- [x] **CA-07:** `PATCH /da/close` acepta firma del ejecutor (CEO o COO) indistintamente
