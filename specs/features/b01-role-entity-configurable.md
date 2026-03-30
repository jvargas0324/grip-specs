# B-01 — Role Entity Configurable

- **Estado:** en-implementación
- **Prioridad:** 🔴 Bloqueante MVP — sin esto ningún otro módulo puede implementarse correctamente
- **SDD referencia:** Sección 5.1 — Employee Database (Hub Central)
- **Gap Analysis:** B-01 en GRIP-GapAnalysis.md
- **Última actualización:** 2026-03-29
- **Inicio implementación:** 2026-03-29

---

## 1. Objetivo

- **Aplicación(es):** grip-backend
- **Problema:** El modelo actual tiene los roles hardcodeados como un `Enum` de PostgreSQL (`jz`, `gv`, `regional`, `coo`, `ceo`). Esto impide representar la estructura real de Ritmo (Gerente Logístico, Gerente Regional, Jefe de Zona, etc.) y hace imposible configurar permisos por rol sin cambiar código.
- **Alcance:**
  - Crear tabla `roles` con entidad configurable por tenant
  - Refactorizar tabla `users` → `employees` con `role_id` (FK a `roles`) y campos adicionales del SDD
  - Migración Alembic que preserve los datos existentes
  - Schemas Pydantic actualizados
  - Endpoints CRUD para `roles` y `employees`
  - **Queda fuera:** autenticación JWT (B-04), multi-tenancy completo (B-03), frontend (tarea separada)

---

## 2. Referencia a specs base

- SDD Master: `specs/GRIP-SDD-Functional-Specs.md` — Sección 5.1.2 (Data Model), 5.1.3 (Functional Requirements), 5.1.4 (Business Rules)
- Gap Analysis: `specs/GRIP-GapAnalysis.md` — Módulo 5.1

---

## 3. Actores y permisos

| Rol | Operación permitida |
|---|---|
| CEO / COO | CRUD completo de roles y employees |
| Gerente Regional | Read employees de su región |
| Todos los roles | Read su propio perfil |

> En MVP con auth mock, los endpoints son accesibles sin restricción estricta. Los guards se endurecen en B-04.

---

## 4. Modelo de datos objetivo

### Tabla `roles` (nueva)

```sql
CREATE TABLE roles (
    id          UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id   UUID NOT NULL REFERENCES tenants(id),  -- B-03; por ahora nullable
    code        VARCHAR(50) NOT NULL,    -- e.g. "CEO", "COO", "GTE_REGIONAL", "GTE_VENTAS", "JEFE_ZONA", "JEFE_TIENDA"
    name        VARCHAR(255) NOT NULL,   -- display name
    hierarchy_level INTEGER NOT NULL,    -- 1=CEO, 2=C-suite, 3=Regional, 4=Area, 5=Zona, 6=Tienda
    can_execute_da          BOOLEAN NOT NULL DEFAULT FALSE,
    can_execute_3point      BOOLEAN NOT NULL DEFAULT FALSE,
    can_execute_checklist   BOOLEAN NOT NULL DEFAULT FALSE,
    can_write_weekly_report BOOLEAN NOT NULL DEFAULT FALSE,
    da_scope    VARCHAR(20) NOT NULL DEFAULT 'NONE',  -- NONE | OPERATIONS_ONLY | ALL_AREAS
    created_at  TIMESTAMP DEFAULT NOW(),
    UNIQUE(tenant_id, code)
);
```

### Tabla `employees` (reemplaza `users`)

```sql
CREATE TABLE employees (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id       UUID,               -- nullable hasta B-03
    employee_code   VARCHAR(50),        -- único por tenant
    first_name      VARCHAR(255) NOT NULL,
    last_name       VARCHAR(255),
    email           VARCHAR(255) UNIQUE NOT NULL,
    role_id         UUID NOT NULL REFERENCES roles(id),
    reports_to      UUID REFERENCES employees(id),  -- nullable para CEO
    functional_area VARCHAR(100),       -- "Ventas" | "Logistica" | "Admin" | etc.
    status          VARCHAR(20) NOT NULL DEFAULT 'ACTIVE',  -- ACTIVE | INACTIVE
    hire_date       DATE,
    locale          VARCHAR(20) DEFAULT 'es-DO',
    timezone        VARCHAR(50) DEFAULT 'America/Santo_Domingo',
    created_at      TIMESTAMP DEFAULT NOW(),
    updated_at      TIMESTAMP DEFAULT NOW()
);
```

> **Nota sobre `zone_id`:** La tabla `users` actual tiene `zone_id`. En el nuevo modelo, la pertenencia territorial se deriva del árbol `reports_to` — el JZ tiene asignadas tiendas directamente. `zone_id` se migra a `employees` como campo temporal durante la transición y se depreca en B-06.

---

## 5. Reglas de negocio

1. **RB-01:** Un `Role` con `da_scope = ALL_AREAS` solo puede asignarse al CEO. `OPERATIONS_ONLY` solo al COO.
2. **RB-02:** Un `Employee` con `reports_to = NULL` es el nodo raíz (CEO). Solo puede existir uno por tenant.
3. **RB-03:** `Role.code` es único por tenant. No se pueden crear dos roles con el mismo código.
4. **RB-04:** No se puede eliminar un `Role` si tiene `Employees` activos asignados.
5. **RB-05:** `hierarchy_level` determina quién puede ejecutar 3-Point Reviews — solo roles con `can_execute_3point = TRUE` (nivel 4 = Area Manager hacia arriba).
6. **RB-06:** El campo `employee_code` debe ser único dentro del mismo tenant.

---

## 6. Seed data (Ritmo — tenant inicial)

Al ejecutar la migración, insertar los roles base de Ritmo:

| code | name | hierarchy_level | can_da | can_3pt | can_checklist | can_weekly | da_scope |
|---|---|---|---|---|---|---|---|
| `CEO` | CEO | 1 | TRUE | TRUE | FALSE | TRUE | ALL_AREAS |
| `COO` | COO | 2 | TRUE | TRUE | FALSE | TRUE | OPERATIONS_ONLY |
| `CFO` | CFO | 2 | FALSE | FALSE | FALSE | FALSE | NONE |
| `CTO` | CTO | 2 | FALSE | FALSE | FALSE | FALSE | NONE |
| `CPO` | CPO | 2 | FALSE | FALSE | FALSE | FALSE | NONE |
| `GTE_REGIONAL` | Gerente Regional | 3 | FALSE | TRUE | FALSE | TRUE | NONE |
| `GTE_VENTAS` | Gerente de Ventas | 4 | FALSE | TRUE | FALSE | TRUE | NONE |
| `GTE_LOGISTICO` | Gerente Logístico | 4 | FALSE | TRUE | FALSE | TRUE | NONE |
| `JEFE_ZONA` | Jefe de Zona | 5 | FALSE | FALSE | TRUE | TRUE | NONE |
| `JEFE_TIENDA` | Jefe de Tienda | 6 | FALSE | FALSE | FALSE | FALSE | NONE |

---

## 7. Contrato API

### Roles

| Método | Path | Descripción | Auth mínimo |
|---|---|---|---|
| `GET` | `/api/v1/roles` | Listar roles del tenant | Cualquier rol autenticado |
| `POST` | `/api/v1/roles` | Crear nuevo rol | CEO / COO |
| `GET` | `/api/v1/roles/{id}` | Obtener rol por ID | Cualquier rol autenticado |
| `PATCH` | `/api/v1/roles/{id}` | Actualizar rol | CEO / COO |
| `DELETE` | `/api/v1/roles/{id}` | Eliminar rol (si no tiene employees) | CEO |

**Response `GET /api/v1/roles`:**
```json
[
  {
    "id": "uuid",
    "code": "GTE_VENTAS",
    "name": "Gerente de Ventas",
    "hierarchy_level": 4,
    "can_execute_da": false,
    "can_execute_3point": true,
    "can_execute_checklist": false,
    "can_write_weekly_report": true,
    "da_scope": "NONE"
  }
]
```

### Employees

| Método | Path | Descripción | Auth mínimo |
|---|---|---|---|
| `GET` | `/api/v1/employees` | Listar employees (con filtro por role_id, status) | Gerente Regional+ |
| `POST` | `/api/v1/employees` | Crear employee | CEO / COO |
| `GET` | `/api/v1/employees/{id}` | Obtener employee por ID | Propio perfil o superior |
| `PATCH` | `/api/v1/employees/{id}` | Actualizar employee | CEO / COO |
| `GET` | `/api/v1/employees/{id}/reports` | Obtener reportes directos | Superior jerárquico |
| `GET` | `/api/v1/employees/me` | Obtener perfil propio | Cualquier rol |

**Response `GET /api/v1/employees/{id}`:**
```json
{
  "id": "uuid",
  "employee_code": "RIT-001",
  "first_name": "Juan",
  "last_name": "Pérez",
  "email": "juan@ritmo.com",
  "role": {
    "id": "uuid",
    "code": "JEFE_ZONA",
    "name": "Jefe de Zona",
    "hierarchy_level": 5
  },
  "reports_to": {
    "id": "uuid",
    "full_name": "María López",
    "role_code": "GTE_VENTAS"
  },
  "functional_area": "Ventas",
  "status": "ACTIVE"
}
```

---

## 8. Orden de implementación en grip-backend

```
1. db/models/roles.py          ← nuevo modelo Role
2. db/models/employees.py      ← reemplaza users.py (conservar users.py como alias temporal)
3. alembic/versions/XXXX_b01_role_employee.py  ← migración con seed data Ritmo
4. schemas/roles.py            ← Pydantic: RoleCreate, RoleUpdate, RoleResponse
5. schemas/employees.py        ← Pydantic: EmployeeCreate, EmployeeUpdate, EmployeeResponse
6. api/roles.py                ← router CRUD roles
7. api/employees.py            ← router CRUD employees (reemplaza users.py)
8. main.py                     ← registrar nuevos routers, marcar users como deprecated
```

---

## 9. Criterios de aceptación

- [x] **CA-01:** `GET /api/v1/roles` retorna los 10 roles seed de Ritmo con todos sus campos
- [x] **CA-02:** `POST /api/v1/roles` crea un rol nuevo con `code` único — falla con 409 si el código ya existe
- [x] **CA-03:** `DELETE /api/v1/roles/{id}` falla con 400 si el rol tiene employees activos asignados
- [x] **CA-04:** `POST /api/v1/employees` crea un employee con `role_id` válido y `reports_to` opcional
- [x] **CA-05:** `GET /api/v1/employees/{id}/reports` retorna la lista de reportes directos del employee
- [x] **CA-06:** Un employee con `reports_to = NULL` es el nodo raíz — solo puede existir uno
- [x] **CA-07:** La migración Alembic preserva los usuarios existentes de `users` mapeando roles por nombre
- [x] **CA-08:** `employee_code` es único — falla con 409 si se intenta duplicar

---

## 10. Notas

- Los FKs que actualmente apuntan a `users.id` (visits.jz_id, da_interventions.creator_id, etc.) deben actualizarse para apuntar a `employees.id`. Esto se hace en esta misma migración.
- `tenant_id` se deja como nullable hasta B-03 (multi-tenancy). En B-03 se vuelve NOT NULL.
- El endpoint `GET /api/v1/users` existente se conserva como alias deprecated durante la transición.
- Dependency: ninguna. Este es el primer bloqueante — no depende de B-02, B-03 ni B-04.
