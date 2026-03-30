# B-03 — Tenant Isolation (Multi-tenancy)

- **Estado:** en-implementación
- **Prioridad:** 🔴 Bloqueante MVP
- **Depende de:** B-01 (Employee entity)
- **SDD referencia:** Sección 4.3 — Tenant Isolation Strategy, Sección 9 — Configuration & Multi-Tenancy
- **Última actualización:** 2026-03-29
- **Inicio implementación:** 2026-03-29

---

## 1. Objetivo

- **Aplicación(es):** grip-backend
- **Problema:** Ninguna tabla tiene `tenant_id`. En un sistema multi-tenant, todos los datos de un tenant son visibles para otro. El MVP opera con un solo tenant (Ritmo) pero la arquitectura debe ser multi-tenant desde el inicio para no tener que reescribir después.
- **Alcance:**
  - Crear tabla `tenants`
  - Añadir `tenant_id` (NOT NULL) a todas las tablas core
  - Middleware de resolución de tenant por request
  - Seed del tenant inicial: Ritmo
  - **MVP:** un solo tenant activo. La lógica de creación de tenants adicionales queda para post-MVP.
  - **Queda fuera:** schema-per-tenant (PostgreSQL schemas separados). En MVP se usa row-level isolation con `tenant_id`. Schema-per-tenant es una migración futura si el volumen lo justifica.

---

## 2. Referencia a specs base

- SDD Master: `specs/GRIP-SDD-Functional-Specs.md` — Sección 4.3, Sección 9
- Gap Analysis: B-03 en `specs/GRIP-GapAnalysis.md`

---

## 3. Modelo de datos

### Tabla `tenants` (nueva)

```sql
CREATE TABLE tenants (
    id          UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    code        VARCHAR(50) UNIQUE NOT NULL,   -- e.g. "RITMO"
    name        VARCHAR(255) NOT NULL,          -- e.g. "Tiendas Ritmo"
    locale      VARCHAR(20) DEFAULT 'es-DO',
    timezone    VARCHAR(50) DEFAULT 'America/Santo_Domingo',
    status      VARCHAR(20) NOT NULL DEFAULT 'ACTIVE',
    created_at  TIMESTAMP DEFAULT NOW()
);
```

### Tablas que reciben `tenant_id NOT NULL`

| Tabla | Acción |
|---|---|
| `roles` | Añadir `tenant_id NOT NULL REFERENCES tenants(id)` |
| `employees` | Cambiar `tenant_id` de nullable a NOT NULL |
| `stores` | Añadir `tenant_id NOT NULL` |
| `zones` | Añadir `tenant_id NOT NULL` |
| `visits` | Añadir `tenant_id NOT NULL` |
| `findings` | Añadir `tenant_id NOT NULL` |
| `da_interventions` | Añadir `tenant_id NOT NULL` |
| `weekly_sessions` | Añadir `tenant_id NOT NULL` |
| `focal_points` | Añadir `tenant_id NOT NULL` |
| `strategic_feedback` | Añadir `tenant_id NOT NULL` |
| `evidences` | Añadir `tenant_id NOT NULL` |

---

## 4. Resolución de tenant por request

En MVP, el tenant se resuelve por header HTTP:

```
X-Tenant-ID: <tenant_uuid>
```

En producción futura se puede resolver por subdominio (`ritmo.grip.app`). El mecanismo de resolución vive en `app/api/dependencies.py` como una dependency de FastAPI.

```python
# dependencies.py
async def get_current_tenant(
    x_tenant_id: str = Header(...),
    db: Session = Depends(get_db),
) -> Tenant:
    tenant = db.get(Tenant, x_tenant_id)
    if not tenant or tenant.status != "ACTIVE":
        raise HTTPException(status_code=403, detail="Tenant not found or inactive")
    return tenant
```

Todas las queries core incluyen `WHERE tenant_id = :tenant_id` usando la tenant resolvida.

---

## 5. Seed data

```sql
INSERT INTO tenants (id, code, name, locale, timezone, status)
VALUES (
  'fixed-uuid-ritmo',  -- UUID fijo para desarrollo local
  'RITMO',
  'Tiendas Ritmo',
  'es-DO',
  'America/Santo_Domingo',
  'ACTIVE'
);
```

---

## 6. Reglas de negocio

1. **RB-01:** Toda query que acceda a datos operativos DEBE incluir `tenant_id` como filtro. Ningún endpoint retorna datos cross-tenant.
2. **RB-02:** El tenant se resuelve en cada request — nunca se almacena en sesión del servidor.
3. **RB-03:** En MVP, si no se envía `X-Tenant-ID`, el sistema usa el tenant RITMO por defecto (facilita desarrollo local).
4. **RB-04:** Un `Role` pertenece a un tenant — no son compartidos entre tenants.

---

## 7. Contrato API

| Método | Path | Descripción |
|---|---|---|
| `GET` | `/api/v1/tenant` | Retorna info del tenant activo (resuelto por header) |

> No se expone CRUD de tenants en MVP. El tenant Ritmo se crea via seed en migración.

---

## 8. Orden de implementación en grip-backend

```
1. db/models/tenants.py                       ← nuevo modelo Tenant
2. alembic/versions/XXXX_b03_tenants.py       ← crea tabla + seed Ritmo + añade tenant_id a todas las tablas
3. api/dependencies.py                         ← añadir get_current_tenant()
4. Actualizar todas las queries en services/  ← añadir tenant_id filter
5. api/tenant.py                              ← endpoint GET /tenant
```

---

## 9. Criterios de aceptación

- [x] **CA-01:** Todas las tablas core tienen `tenant_id NOT NULL` después de la migración
- [x] **CA-02:** `GET /api/v1/findings` sin `X-Tenant-ID` usa tenant RITMO por defecto en desarrollo
- [x] **CA-03:** `GET /api/v1/tenant` retorna `{ code: "RITMO", name: "Tiendas Ritmo", locale: "es-DO" }`
- [x] **CA-04:** Una query directa a DB sin filtro `tenant_id` no es posible desde los servicios (verificado en code review)
- [x] **CA-05:** La migración preserva todos los registros existentes asignándolos al tenant RITMO
