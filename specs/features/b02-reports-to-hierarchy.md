# B-02 — Jerarquía Flexible reports_to

- **Estado:** en-implementación
- **Prioridad:** 🔴 Bloqueante MVP
- **Depende de:** B-01 (Employee entity)
- **SDD referencia:** Sección 5.1.2 — Employee.reports_to
- **Última actualización:** 2026-03-29
- **Inicio implementación:** 2026-03-29

---

## 1. Objetivo

- **Aplicación(es):** grip-backend
- **Problema:** El modelo actual vincula usuarios a zonas (`zone_id`) para inferir jerarquía. Esto no soporta la estructura real de Ritmo (7+ niveles, múltiples áreas funcionales). La jerarquía debe ser un árbol explícito mediante `reports_to` (FK self-referencial en `employees`).
- **Alcance:**
  - El campo `reports_to` ya existe en la spec de B-01. Esta feature especifica la lógica de negocio, los endpoints de árbol jerárquico y las queries de visibilidad basadas en el árbol.
  - Queda fuera: UI de organigrama (frontend), asignación de tiendas a JZ (B-06).

---

## 2. Referencia a specs base

- SDD Master: `specs/GRIP-SDD-Functional-Specs.md` — Sección 4.4 (Flujo Bottom-to-Top), 5.1.2 (Employee.reports_to)

---

## 3. Modelo del árbol

```
CEO  (reports_to = NULL)
 └── COO
      └── Gerente Regional Norte
           ├── Gerente de Ventas Norte
           │    ├── Jefe de Zona 1
           │    │    └── Jefe de Tienda A
           │    └── Jefe de Zona 2
           └── Gerente Logístico Norte
                └── Jefe de Recibo
```

- Cada nodo conoce a su padre (`reports_to`)
- El árbol completo se reconstruye navegando recursivamente

---

## 4. Reglas de negocio

1. **RB-01:** `reports_to = NULL` solo para el CEO (nodo raíz). Un segundo empleado con `reports_to = NULL` es inválido.
2. **RB-02:** No se permiten ciclos en el árbol. Antes de asignar `reports_to`, el sistema debe verificar que el target no sea descendiente del employee a modificar.
3. **RB-03:** La visibilidad de datos de cada rol se deriva del árbol: un manager ve todo lo que está por debajo de su nodo.
4. **RB-04:** Al desactivar (`status = INACTIVE`) un employee con reportes directos, el sistema debe requerir reasignación de sus subordinados antes de proceder.
5. **RB-05:** `hierarchy_level` del Role es la fuente de verdad para permisos — no la profundidad del árbol. El árbol puede tener profundidad variable sin romper la lógica de permisos.

---

## 5. Contrato API

| Método | Path | Descripción |
|---|---|---|
| `GET` | `/api/v1/employees/{id}/tree` | Subárbol completo desde el employee hacia abajo |
| `GET` | `/api/v1/employees/{id}/ancestors` | Cadena de reportes hacia arriba hasta CEO |
| `GET` | `/api/v1/employees/{id}/reports` | Solo reportes directos (1 nivel) |
| `GET` | `/api/v1/employees/{id}/scope` | Todos los employees visibles para este nodo (descendientes) |

**Response `GET /api/v1/employees/{id}/tree`:**
```json
{
  "id": "uuid",
  "full_name": "María López",
  "role_code": "GTE_VENTAS",
  "reports": [
    {
      "id": "uuid",
      "full_name": "Juan Pérez",
      "role_code": "JEFE_ZONA",
      "reports": [
        {
          "id": "uuid",
          "full_name": "Ana Torres",
          "role_code": "JEFE_TIENDA",
          "reports": []
        }
      ]
    }
  ]
}
```

---

## 6. Implementación en grip-backend

```
1. services/hierarchy_service.py   ← lógica de árbol (build_tree, get_ancestors, get_scope, cycle_check)
2. api/employees.py                ← añadir endpoints /tree, /ancestors, /reports, /scope
```

La query del árbol usa CTE recursivo de PostgreSQL:

```sql
WITH RECURSIVE subordinates AS (
  SELECT id, full_name, role_id, reports_to, 0 AS depth
  FROM employees WHERE id = :root_id
  UNION ALL
  SELECT e.id, e.full_name, e.role_id, e.reports_to, s.depth + 1
  FROM employees e
  INNER JOIN subordinates s ON e.reports_to = s.id
)
SELECT * FROM subordinates;
```

---

## 7. Criterios de aceptación

- [x] **CA-01:** `GET /tree` retorna el subárbol completo con anidamiento correcto
- [x] **CA-02:** `GET /scope` retorna todos los descendientes de un nodo (usado para filtrar visibilidad de datos)
- [x] **CA-03:** Crear un employee con `reports_to` apuntando a un descendiente suyo falla con 400 (ciclo detectado)
- [x] **CA-04:** Solo existe un employee con `reports_to = NULL` por tenant
- [x] **CA-05:** Desactivar un employee con reportes directos sin reasignarlos falla con 400
