# B-06 — Checklist Template Digital

- **Estado:** en-implementación
- **Prioridad:** 🔴 Bloqueante MVP
- **Depende de:** B-01 (Employee), B-03 (Tenant), B-04 (Auth)
- **SDD referencia:** Sección 5.3 — Store Visit Checklist
- **Última actualización:** 2026-03-29
- **Inicio implementación:** 2026-03-29

---

## 1. Objetivo

- **Aplicación(es):** grip-backend, grip-frontend
- **Problema:** La PoC ingesta checklists via PDF (papel → upload → AI extracción). El SDD especifica un checklist digital mobile-first donde el JZ llena el formulario en la app directamente. No existe modelo de `ChecklistTemplate` ni estructura de ítems fijos por sección.
- **Alcance:**
  - Modelo de datos: `ChecklistTemplate`, `ChecklistTemplateItem`, `ChecklistSession`, `ChecklistSessionItem`, `VariableItem`
  - API para ejecutar una visita digital (crear sesión, registrar ítems, cerrar sesión)
  - API para gestionar el template (HQ configura ítems fijos)
  - UI mobile-first para el JZ: navegar por secciones, marcar cumple/no cumple, foto inline
  - **El canal PDF (PoC) se conserva como backup** — no se elimina, se desprioriza
  - **Queda fuera:** AI-08 (sugerencia de ítems variables), auto-surface flags completos (B-07)

---

## 2. Referencia a specs base

- SDD Master: `specs/GRIP-SDD-Functional-Specs.md` — Sección 5.3 completa
- Checklist real de Ritmo: 9 secciones (Exterior, Limpieza, Exhibición, Precios, Neveras, Puesto de Pago, Bodega, Documentos, Empleados)

---

## 3. Modelo de datos

### `checklist_templates` (nueva)

```sql
CREATE TABLE checklist_templates (
    id          UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id   UUID NOT NULL REFERENCES tenants(id),
    name        VARCHAR(255) NOT NULL,        -- e.g. "Checklist Estándar Ritmo v1"
    is_active   BOOLEAN NOT NULL DEFAULT TRUE,
    created_at  TIMESTAMP DEFAULT NOW()
);
```

### `checklist_template_items` (nueva)

```sql
CREATE TABLE checklist_template_items (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    template_id     UUID NOT NULL REFERENCES checklist_templates(id),
    section         VARCHAR(100) NOT NULL,   -- "Exterior" | "Limpieza" | etc.
    description     TEXT NOT NULL,           -- texto del ítem a verificar
    order_index     INTEGER NOT NULL,        -- orden dentro de la sección
    is_required     BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMP DEFAULT NOW()
);
```

### `checklist_sessions` (evoluciona `visits`)

```sql
CREATE TABLE checklist_sessions (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    template_id     UUID NOT NULL REFERENCES checklist_templates(id),
    store_id        UUID NOT NULL REFERENCES stores(id),
    executor_id     UUID NOT NULL REFERENCES employees(id),  -- JZ
    visit_date      DATE NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'IN_PROGRESS',  -- IN_PROGRESS | COMPLETED
    score           NUMERIC(5,2),            -- % de cumplimiento calculado al cerrar
    created_at      TIMESTAMP DEFAULT NOW(),
    completed_at    TIMESTAMP
);
```

### `checklist_session_items` (nueva)

```sql
CREATE TABLE checklist_session_items (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    session_id      UUID NOT NULL REFERENCES checklist_sessions(id),
    template_item_id UUID REFERENCES checklist_template_items(id),  -- NULL si es ítem variable
    is_variable     BOOLEAN NOT NULL DEFAULT FALSE,
    description     TEXT NOT NULL,           -- copiado del template o escrito por JZ
    section         VARCHAR(100) NOT NULL,
    result          VARCHAR(20),             -- CUMPLE | NO_CUMPLE | NA
    observation     TEXT,
    photo_url       TEXT,
    created_at      TIMESTAMP DEFAULT NOW()
);
```

### `variable_items` (sugeridos por JZ o AI)

```sql
CREATE TABLE variable_items (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    session_id      UUID NOT NULL REFERENCES checklist_sessions(id),
    description     TEXT NOT NULL,
    section         VARCHAR(100),
    source          VARCHAR(20) NOT NULL DEFAULT 'JZ',  -- JZ | AI
    justification   TEXT,                    -- razón del ítem (especialmente si es AI)
    created_at      TIMESTAMP DEFAULT NOW()
);
```

---

## 4. Seed data — Template Ritmo (9 secciones)

Al ejecutar la migración, insertar el template base de Ritmo con sus ítems fijos por sección:

| Sección | Ítems representativos |
|---|---|
| Exterior | Fachada limpia, letrero iluminado, entrada despejada |
| Limpieza | Pisos, estantes, baños, área de caja |
| Exhibición | Productos ordenados, frentes llenos, etiquetas correctas |
| Precios | Todos los productos tienen precio visible, precios correctos |
| Neveras | Temperatura correcta, productos organizados, puertas sin daño |
| Puesto de Pago | Caja organizada, cambio disponible, recibos funcionando |
| Bodega | Organización, control de inventario, acceso restringido |
| Documentos | Permisos vigentes, documentos de empleados al día |
| Empleados | Uniforme completo, actitud, presencia completa del equipo |

> El seed completo se define en la migración Alembic con todos los ítems del checklist real de Ritmo.

---

## 5. Flujo de una visita digital

```
1. JZ abre app → selecciona tienda → crea ChecklistSession (status: IN_PROGRESS)
2. Sistema carga los ítems del template activo para esa tienda
3. JZ navega sección por sección:
   a. Marca cada ítem: CUMPLE / NO_CUMPLE / NA
   b. Si NO_CUMPLE → panel inline para foto + observación (≤ 4 taps)
   c. Puede añadir ítems variables en cualquier sección
4. Al completar todas las secciones → JZ cierra la sesión
5. Sistema calcula score = (CUMPLE / total_evaluados) * 100
6. Por cada ítem NO_CUMPLE → se crea un Finding + FollowUpItem (B-07)
```

---

## 6. Contrato API

| Método | Path | Descripción | Auth |
|---|---|---|---|
| `GET` | `/api/v1/checklist/template` | Template activo del tenant con todos sus ítems | JZ+ |
| `POST` | `/api/v1/checklist/sessions` | Crear nueva sesión de visita | JZ |
| `GET` | `/api/v1/checklist/sessions/{id}` | Estado actual de la sesión | JZ |
| `POST` | `/api/v1/checklist/sessions/{id}/items` | Registrar resultado de un ítem | JZ |
| `POST` | `/api/v1/checklist/sessions/{id}/variable-items` | Añadir ítem variable | JZ |
| `POST` | `/api/v1/checklist/sessions/{id}/complete` | Cerrar sesión y calcular score | JZ |
| `GET` | `/api/v1/checklist/sessions` | Historial de sesiones (filtro: store_id, executor_id) | JZ+ |

**Response `GET /checklist/template`:**
```json
{
  "id": "uuid",
  "name": "Checklist Estándar Ritmo v1",
  "sections": [
    {
      "name": "Exterior",
      "items": [
        { "id": "uuid", "description": "Fachada limpia", "order_index": 1, "is_required": true },
        { "id": "uuid", "description": "Letrero iluminado", "order_index": 2, "is_required": true }
      ]
    }
  ]
}
```

**Request `POST /checklist/sessions/{id}/items`:**
```json
{
  "template_item_id": "uuid",
  "result": "NO_CUMPLE",
  "observation": "Piso con manchas en sección de neveras",
  "photo_url": "https://storage/foto.jpg"
}
```

---

## 7. UI Frontend (grip-frontend)

### Pantallas nuevas

```
/checklist/new          ← seleccionar tienda + iniciar sesión
/checklist/:id          ← ejecutar checklist (tab por sección)
/checklist/:id/summary  ← resumen al completar (score + findings creados)
/checklist/history      ← historial de visitas del JZ
```

### Principios de diseño (SDD 1.4)

- **TAP, not type:** cada ítem se marca con 1 tap (CUMPLE / NO_CUMPLE / NA)
- **Foto inline:** al marcar NO_CUMPLE, el panel de foto aparece en el mismo ítem — no navega a otra pantalla
- **Progreso visible:** barra de progreso por sección + score parcial en tiempo real
- **Secciones como tabs:** navegación horizontal entre las 9 secciones

---

## 8. Orden de implementación

```
BACKEND:
1. db/models/checklist_template.py
2. db/models/checklist_session.py
3. alembic/versions/XXXX_b06_checklist.py  (+ seed template Ritmo)
4. schemas/checklist.py
5. services/checklist_service.py           (create_session, record_item, complete_session, calc_score)
6. api/checklist.py

FRONTEND:
7. features/checklist/services/checklist.service.ts
8. features/checklist/checklist-new/           (selección de tienda)
9. features/checklist/checklist-execute/       (formulario por sección)
10. features/checklist/checklist-summary/      (resultado)
```

---

## 9. Criterios de aceptación

- [x] **CA-01:** `GET /checklist/template` retorna las 9 secciones con sus ítems en orden correcto
- [x] **CA-02:** `POST /checklist/sessions` crea una sesión `IN_PROGRESS` para la tienda seleccionada
- [x] **CA-03:** `POST /sessions/{id}/items` con `result: NO_CUMPLE` registra el ítem y acepta foto + observación
- [x] **CA-04:** `POST /sessions/{id}/complete` calcula el score correctamente y cambia status a `COMPLETED`
- [x] **CA-05:** Al completar la sesión, por cada ítem `NO_CUMPLE` se crea un `Finding` vinculado
- [x] **CA-06:** El JZ puede añadir ítems variables con `source: JZ` en cualquier sección
- [x] **CA-07:** La UI completa una visita de 10 ítems en ≤ 3 minutos — implementado: 1-tap per item, section tabs, progress bar
- [x] **CA-08:** La foto se captura inline sin salir del flujo del checklist — implementado: panel inline en NO_CUMPLE con observation + photo_url
- [x] **CA-09:** Link "Checklist" visible en sidebar para roles JZ, GV, Regional. Guard en ruta. No visible para COO/CEO.
