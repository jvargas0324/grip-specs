# B-07 — Follow-Up Item Entity

- **Estado:** en-implementación
- **Prioridad:** 🔴 Bloqueante MVP
- **Depende de:** B-01 (Employee), B-03 (Tenant), B-04 (Auth), B-06 (ChecklistSession)
- **SDD referencia:** Sección 5.6 — Follow-Up System (Core)
- **Última actualización:** 2026-03-29
- **Inicio implementación:** 2026-03-29

---

## 1. Objetivo

- **Aplicación(es):** grip-backend, grip-frontend
- **Problema:** La PoC usa `Finding.status` para trackear follow-ups — mezcla la observación (el hallazgo) con la tarea de resolución. El SDD separa ambos conceptos. Además no existen los flags `auto_surface_*` que son el corazón del mecanismo "0 hallazgos olvidados".
- **Alcance:**
  - Crear entidad `FollowUpItem` separada de `Finding`
  - Lifecycle completo: OPEN → IN_PROGRESS → RESOLVED → VERIFIED
  - Auto-surface flags: `auto_surface_in_next_checklist`, `auto_surface_in_next_da`, `auto_surface_in_next_review`
  - Job diario de escalación por deadline vencido
  - API para gestión de follow-ups por rol
  - UI: panel de follow-ups en dashboard JZ + panel de escalados en Gte. Ventas
  - **Queda fuera:** AI-06 (Follow-Up Automation summary), pattern detection

---

## 2. Referencia a specs base

- SDD Master: `specs/GRIP-SDD-Functional-Specs.md` — Sección 5.6 completa
- Gap Analysis: Módulo 5.6 en `specs/GRIP-GapAnalysis.md`

---

## 3. Separación Finding vs FollowUpItem

```
Finding                          FollowUpItem
────────────────────             ────────────────────────────────
La observación registrada        La tarea de resolución asignada
Quién lo vio y cuándo            Quién lo debe resolver y cuándo
Qué encontró el JZ               Si se resolvió y con qué evidencia
Inmutable una vez creado         Mutable — su estado cambia
```

Un `Finding` puede originar 0 o 1 `FollowUpItem`. Los findings que "cumplen" no generan follow-up.

---

## 4. Modelo de datos

### `follow_up_items` (nueva)

```sql
CREATE TABLE follow_up_items (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),

    -- Origen (trazabilidad)
    source_type     VARCHAR(30) NOT NULL,  -- CHECKLIST | DA | THREE_POINT | WEEKLY_REPORT
    source_id       UUID NOT NULL,         -- FK al Finding, DAIntervention, ThreePointReview, etc.

    -- Contexto
    store_id        UUID REFERENCES stores(id),
    description     TEXT NOT NULL,
    severity        VARCHAR(10) NOT NULL DEFAULT 'MEDIUM',  -- LOW | MEDIUM | HIGH

    -- Asignación
    owner_id        UUID NOT NULL REFERENCES employees(id),
    reported_by     UUID NOT NULL REFERENCES employees(id),

    -- Lifecycle
    status          VARCHAR(20) NOT NULL DEFAULT 'OPEN',
    -- OPEN | IN_PROGRESS | RESOLVED | VERIFIED
    deadline        DATE NOT NULL,

    -- Resolución
    resolution_notes    TEXT,
    resolution_photo_url TEXT,
    resolved_at         TIMESTAMP,
    verified_by         UUID REFERENCES employees(id),
    verified_at         TIMESTAMP,

    -- Auto-surface flags (corazón del mecanismo "0 hallazgos olvidados")
    auto_surface_in_next_checklist  BOOLEAN NOT NULL DEFAULT TRUE,
    auto_surface_in_next_da         BOOLEAN NOT NULL DEFAULT FALSE,
    auto_surface_in_next_review     BOOLEAN NOT NULL DEFAULT FALSE,

    -- Escalación
    escalated_to    UUID REFERENCES employees(id),
    escalated_at    TIMESTAMP,

    created_at      TIMESTAMP DEFAULT NOW(),
    updated_at      TIMESTAMP DEFAULT NOW()
);
```

---

## 5. Reglas de negocio

1. **RB-01:** Todo ítem `NO_CUMPLE` de un checklist genera automáticamente un `FollowUpItem` con `auto_surface_in_next_checklist = TRUE`.
2. **RB-02:** La evidencia (`resolution_photo_url` o `resolution_notes`) es **obligatoria** para pasar a status `RESOLVED`. No se puede resolver sin evidencia.
3. **RB-03:** Solo el superior jerárquico del `owner` puede cambiar status a `VERIFIED`. El owner no puede verificar sus propios follow-ups.
4. **RB-04:** Si `deadline` vence y `status` es `OPEN` o `IN_PROGRESS`, el sistema escala automáticamente al superior del `owner` (job diario).
5. **RB-05:** Un `FollowUpItem` escalado mantiene su owner original — el escalado es una notificación y visibilidad adicional, no una reasignación.
6. **RB-06:** Al iniciar una `ChecklistSession` para una tienda, el sistema consulta los `FollowUpItems` con `auto_surface_in_next_checklist = TRUE` para esa tienda y los inyecta al inicio del checklist como contexto.
7. **RB-07:** Al crear un DA para un Gerente Regional, el sistema consulta `FollowUpItems` con `auto_surface_in_next_da = TRUE` vinculados a ese regional.
8. **RB-08:** Un `FollowUpItem` en status `VERIFIED` es inmutable — no puede reabrirse sin crear uno nuevo.

---

## 6. Lifecycle de estados

```
          [Finding NO_CUMPLE creado]
                    │
                    ▼
                  OPEN  ──── deadline vence ──→ ESCALADO (owner sigue, superior notificado)
                    │
            owner actualiza
                    │
                    ▼
               IN_PROGRESS
                    │
         owner sube evidencia
                    │
                    ▼
               RESOLVED  (evidencia obligatoria)
                    │
         superior verifica
                    │
                    ▼
               VERIFIED  ✅ (inmutable)
```

---

## 7. Job de escalación diario

```python
# Ejecutar cada día a las 06:00 AM (hora del tenant)
async def escalation_job(db: Session, tenant_id: UUID):
    today = date.today()
    overdue_items = db.execute(
        select(FollowUpItem).where(
            FollowUpItem.tenant_id == tenant_id,
            FollowUpItem.deadline < today,
            FollowUpItem.status.in_(["OPEN", "IN_PROGRESS"]),
            FollowUpItem.escalated_at.is_(None),  # no escalar dos veces
        )
    ).all()

    for item in overdue_items:
        superior = get_direct_superior(item.owner_id, db)
        if superior:
            item.escalated_to = superior.id
            item.escalated_at = datetime.utcnow()
            # Notificar al superior
            notify(superior, item)
    db.commit()
```

---

## 8. Contrato API

| Método | Path | Descripción | Auth |
|---|---|---|---|
| `GET` | `/api/v1/followups` | Lista follow-ups en scope del usuario (filtros: status, store_id, severity, overdue) | Todos |
| `GET` | `/api/v1/followups/{id}` | Detalle de un follow-up | Owner o superior |
| `PATCH` | `/api/v1/followups/{id}/status` | Actualizar status + evidencia | Owner |
| `PATCH` | `/api/v1/followups/{id}/verify` | Verificar cierre | Superior del owner |
| `GET` | `/api/v1/followups/surface/checklist/{store_id}` | Follow-ups a surfacear al iniciar checklist en tienda X | JZ |
| `GET` | `/api/v1/followups/surface/da/{regional_id}` | Follow-ups a surfacear al iniciar DA sobre regional X | CEO/COO |

**Request `PATCH /followups/{id}/status`:**
```json
{
  "status": "RESOLVED",
  "resolution_notes": "Se limpió el piso y se tomó foto de evidencia",
  "resolution_photo_url": "https://storage/evidencia.jpg"
}
```

**Response `GET /followups?status=OPEN&overdue=true`:**
```json
{
  "items": [
    {
      "id": "uuid",
      "description": "Piso con manchas en sección neveras",
      "severity": "MEDIUM",
      "source_type": "CHECKLIST",
      "store": { "code": "CACIQUE T002", "name": "Cacique T002" },
      "owner": { "id": "uuid", "full_name": "Juan JT" },
      "deadline": "2026-03-22",
      "days_overdue": 7,
      "status": "OPEN",
      "auto_surface_in_next_checklist": true
    }
  ],
  "total": 1,
  "overdue_count": 1
}
```

---

## 9. UI Frontend

### Dashboard JZ — widget Follow-Ups

```
┌─────────────────────────────────────┐
│ FOLLOW-UPS PENDIENTES (3)           │
│                                     │
│ 🔴 Piso manchado — 7 días vencido   │
│    Tienda: CACIQUE T002             │
│    [Ver] [Resolver]                 │
│                                     │
│ 🟡 Precio faltante — vence mañana   │
│    Tienda: CACIQUE T003             │
│    [Ver] [Resolver]                 │
└─────────────────────────────────────┘
```

### Modal "Resolver"

```
┌─────────────────────────────────────┐
│ RESOLVER HALLAZGO                   │
│                                     │
│ 📝 Notas de resolución (requerido)  │
│ [___________________________]       │
│                                     │
│ 📷 Evidencia fotográfica (requerida)│
│ [Tomar foto] [Subir imagen]         │
│                                     │
│ [Marcar como Resuelto]              │
└─────────────────────────────────────┘
```

---

## 10. Orden de implementación

```
BACKEND:
1. db/models/follow_up_items.py
2. alembic/versions/XXXX_b07_followup_items.py
3. schemas/followup.py
4. services/followup_service.py        (create, update_status, verify, surface_for_checklist, surface_for_da)
5. services/escalation_job.py          (job diario)
6. api/followups.py
7. Actualizar checklist_service.py     (auto-crear FollowUpItem al registrar NO_CUMPLE)

FRONTEND:
8. features/followup/followup-widget/  (widget en dashboard JZ)
9. features/followup/resolve-modal/    (modal de resolución con evidencia)
10. features/followup/followup-list/   (vista completa de follow-ups)
```

---

## 11. Criterios de aceptación

- [x] **CA-01:** Al registrar ítem `NO_CUMPLE` en checklist → se crea `FollowUpItem` con `status: OPEN` y `auto_surface_in_next_checklist: TRUE`
- [x] **CA-02:** `PATCH /followups/{id}/status` con `status: RESOLVED` sin evidencia → 400
- [x] **CA-03:** `PATCH /followups/{id}/status` con `status: RESOLVED` con foto → 200, status actualizado
- [x] **CA-04:** `PATCH /followups/{id}/verify` ejecutado por el owner mismo → 403
- [x] **CA-05:** `GET /followups/surface/checklist/{store_id}` retorna solo items con `auto_surface_in_next_checklist: TRUE` para esa tienda
- [x] **CA-06:** Job de escalación marca como `escalated_to` el superior del owner cuando `deadline < today`
- [x] **CA-07:** `GET /followups?overdue=true` retorna items con deadline vencido en scope del usuario
- [x] **CA-08:** Un `FollowUpItem` en status `VERIFIED` no puede actualizarse (retorna 409)
- [x] **CA-09:** Link "Follow-Ups" visible en sidebar para todos los roles. Cada rol ve su scope.
