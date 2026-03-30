# GRIP — Test Plan: MVP Blockers (B-01 a B-07)

| Metadata | Valor |
|---|---|
| **Fecha** | 2026-03-29 |
| **Versión** | 1.0 |
| **Alcance** | Pruebas funcionales de backend API para los 7 bloqueantes MVP |
| **Prerequisitos** | Backend corriendo en :8000, DB migrada (0017), seed data aplicado |
| **Script automatizado** | `grip-backend/tests/functional/test_mvp_blockers.sh` |

---

## Precondiciones generales

| Recurso | Detalle |
|---|---|
| Backend | `uvicorn app.main:app --reload --port 8000` |
| DB | PostgreSQL con migraciones 0001-0017 aplicadas |
| Tenant | Ritmo (`00000000-0000-0000-0000-000000000001`) seedeado |
| Roles | 10 roles Ritmo seedeados |
| Employees | 6 dev employees con password `grip2026` |
| Stores | Al menos 1 store en la DB |

### Identidades de prueba

| Email | Rol | UUID | Password |
|---|---|---|---|
| `ceo@ritmo.dev` | CEO | `10000000-...01` | grip2026 |
| `coo@ritmo.dev` | COO | `10000000-...02` | grip2026 |
| `regional@ritmo.dev` | GERENTE_REGIONAL | `10000000-...03` | grip2026 |
| `ventas@ritmo.dev` | GTE_VENTAS | `10000000-...04` | grip2026 |
| `jz@ritmo.dev` | JEFE_ZONA | `10000000-...05` | grip2026 |
| `jt@ritmo.dev` | JEFE_TIENDA | `10000000-...06` | grip2026 |

---

## B-01 — Role Entity Configurable

### TC-B01-01: Listar roles seed
| Campo | Detalle |
|---|---|
| **CA** | CA-01 |
| **Precondición** | Seed de roles ejecutado |
| **Endpoint** | `GET /api/v1/roles/` |
| **Esperado** | HTTP 200, array de 10 roles con campos: code, name, hierarchy_level, can_execute_da, can_execute_3point, can_execute_checklist, can_write_weekly_report, da_scope |

### TC-B01-02: Crear rol con código duplicado
| Campo | Detalle |
|---|---|
| **CA** | CA-02, RB-03 |
| **Precondición** | Rol "CEO" ya existe para tenant Ritmo |
| **Endpoint** | `POST /api/v1/roles/` con `{"tenant_id":"...","code":"CEO","name":"CEO Dup","hierarchy_level":1}` |
| **Auth** | X-User-Role: ceo |
| **Esperado** | HTTP 409 Conflict |

### TC-B01-03: Eliminar rol con employees activos
| Campo | Detalle |
|---|---|
| **CA** | CA-03, RB-04 |
| **Precondición** | Rol CEO tiene al menos 1 employee activo asignado |
| **Endpoint** | `DELETE /api/v1/roles/{ceo_role_id}` |
| **Auth** | X-User-Role: ceo |
| **Esperado** | HTTP 400 "Cannot delete role with active employees assigned" |

### TC-B01-04: Crear employee con role_id y reports_to
| Campo | Detalle |
|---|---|
| **CA** | CA-04 |
| **Endpoint** | `POST /api/v1/employees/` con role_id válido + reports_to de un employee existente |
| **Auth** | X-User-Role: ceo |
| **Esperado** | HTTP 201, employee con role nested y reports_to nested |

### TC-B01-05: Reportes directos de un employee
| Campo | Detalle |
|---|---|
| **CA** | CA-05 |
| **Endpoint** | `GET /api/v1/employees/{gte_ventas_id}/reports` |
| **Esperado** | HTTP 200, array con JZ (al menos 1 reporte directo) |

### TC-B01-06: Single root — segundo reports_to=NULL rechazado
| Campo | Detalle |
|---|---|
| **CA** | CA-06, RB-02 |
| **Precondición** | CEO ya existe con reports_to=NULL |
| **Endpoint** | `POST /api/v1/employees/` con `reports_to: null` |
| **Esperado** | HTTP 400 "A root employee (reports_to=NULL) already exists" |

### TC-B01-07: Employee code duplicado
| Campo | Detalle |
|---|---|
| **CA** | CA-08, RB-06 |
| **Precondición** | Employee con employee_code "RIT-CEO-001" ya existe |
| **Endpoint** | `POST /api/v1/employees/` con `employee_code: "RIT-CEO-001"` |
| **Esperado** | HTTP 409 Conflict |

---

## B-02 — Jerarquía Flexible reports_to

### TC-B02-01: Subárbol completo desde CEO
| Campo | Detalle |
|---|---|
| **CA** | CA-01 |
| **Endpoint** | `GET /api/v1/employees/{ceo_id}/tree` |
| **Esperado** | HTTP 200, estructura anidada: CEO → COO → Regional → Gte Ventas → JZ → JT |

### TC-B02-02: Scope de Gte Ventas
| Campo | Detalle |
|---|---|
| **CA** | CA-02, RB-03 |
| **Endpoint** | `GET /api/v1/employees/{gte_ventas_id}/scope` |
| **Esperado** | HTTP 200, array con JZ y JT (descendientes) |

### TC-B02-03: Ancestors desde JZ
| Campo | Detalle |
|---|---|
| **CA** | — |
| **Endpoint** | `GET /api/v1/employees/{jz_id}/ancestors` |
| **Esperado** | HTTP 200, array ordenado: [Gte Ventas, Regional, COO, CEO] |

### TC-B02-04: Cycle detection en reports_to
| Campo | Detalle |
|---|---|
| **CA** | CA-03, RB-02 |
| **Precondición** | CEO→COO→Regional→GteVentas→JZ |
| **Endpoint** | `PATCH /api/v1/employees/{ceo_id}` con `{"reports_to": "{jz_id}"}` |
| **Esperado** | HTTP 400 "Cycle detected" |

### TC-B02-05: Desactivar employee con reportes directos
| Campo | Detalle |
|---|---|
| **CA** | CA-05, RB-04 |
| **Precondición** | Gte Ventas tiene JZ como reporte directo |
| **Endpoint** | `PATCH /api/v1/employees/{gte_ventas_id}` con `{"status":"INACTIVE"}` |
| **Esperado** | HTTP 400 "Cannot deactivate employee with direct reports" |

---

## B-03 — Tenant Isolation

### TC-B03-01: Tenant info por defecto
| Campo | Detalle |
|---|---|
| **CA** | CA-02, CA-03, RB-03 |
| **Endpoint** | `GET /api/v1/tenant/` (sin header X-Tenant-ID) |
| **Esperado** | HTTP 200, `{ code: "RITMO", name: "Tiendas Ritmo", locale: "es-DO" }` |

### TC-B03-02: Tenant inválido
| Campo | Detalle |
|---|---|
| **CA** | — |
| **Endpoint** | `GET /api/v1/tenant/` con `X-Tenant-ID: 99999999-0000-0000-0000-000000000099` |
| **Esperado** | HTTP 403 "Tenant not found or inactive" |

### TC-B03-03: tenant_id NOT NULL en todas las tablas
| Campo | Detalle |
|---|---|
| **CA** | CA-01 |
| **Tipo** | Verificación de DB (query directa) |
| **Esperado** | 11 tablas core con `tenant_id NOT NULL` |

---

## B-04 — Autenticación JWT

### TC-B04-01: Login válido
| Campo | Detalle |
|---|---|
| **CA** | CA-01 |
| **Endpoint** | `POST /api/v1/auth/login` con `{"email":"ceo@ritmo.dev","password":"grip2026"}` |
| **Esperado** | HTTP 200, `{ access_token, refresh_token, token_type: "bearer", employee: { full_name, role_code: "CEO" } }` |

### TC-B04-02: Login inválido
| Campo | Detalle |
|---|---|
| **CA** | CA-02 |
| **Endpoint** | `POST /api/v1/auth/login` con password incorrecto |
| **Esperado** | HTTP 401 |

### TC-B04-03: Perfil con Bearer token
| Campo | Detalle |
|---|---|
| **CA** | — |
| **Precondición** | access_token del TC-B04-01 |
| **Endpoint** | `GET /api/v1/auth/me` con `Authorization: Bearer <token>` |
| **Esperado** | HTTP 200, perfil del CEO |

### TC-B04-04: Refresh token
| Campo | Detalle |
|---|---|
| **CA** | CA-06, RB-02 |
| **Precondición** | refresh_token del TC-B04-01 |
| **Endpoint** | `POST /api/v1/auth/refresh` con `{"refresh_token":"..."}` |
| **Esperado** | HTTP 200, nuevo access_token |

### TC-B04-05: Login employee inactivo
| Campo | Detalle |
|---|---|
| **CA** | CA-07, RB-04 |
| **Precondición** | Employee con status=INACTIVE |
| **Endpoint** | `POST /api/v1/auth/login` con credenciales del employee inactivo |
| **Esperado** | HTTP 401 "Account is inactive" |

### TC-B04-06: JZ no puede crear DA (role guard)
| Campo | Detalle |
|---|---|
| **CA** | CA-05 |
| **Endpoint** | `POST /api/v1/da/create` con token de JZ |
| **Esperado** | HTTP 403 |

---

## B-05 — DA Module: COO Scope

### TC-B05-01: JZ no puede crear DA
| Campo | Detalle |
|---|---|
| **CA** | CA-03, RB-01 |
| **Endpoint** | `POST /api/v1/da/create` con X-User-Role: jz |
| **Esperado** | HTTP 403 "Role does not have DA execution permission" |

### TC-B05-02: DA duplicado activo para mismo regional
| Campo | Detalle |
|---|---|
| **CA** | CA-04, RB-05 |
| **Precondición** | DA activo ya existe para target_regional_id |
| **Endpoint** | `POST /api/v1/da/create` con mismo target_regional_id |
| **Esperado** | HTTP 409 |

### TC-B05-03: Target debe ser Gerente Regional
| Campo | Detalle |
|---|---|
| **CA** | —, RB-04 |
| **Endpoint** | `POST /api/v1/da/create` con target_regional_id de un JZ |
| **Esperado** | HTTP 400 "DA target must be a Gerente Regional" |

### TC-B05-04: GET DA detail con findings
| Campo | Detalle |
|---|---|
| **CA** | CA-06 |
| **Precondición** | DA creado con findings vinculados |
| **Endpoint** | `GET /api/v1/da/{id}` |
| **Esperado** | HTTP 200, `{ creator, target_regional, linked_findings[], surfaced_from_previous_da[] }` |

### TC-B05-05: Listar DAs
| Campo | Detalle |
|---|---|
| **CA** | — |
| **Endpoint** | `GET /api/v1/da` |
| **Esperado** | HTTP 200, array de DAs con case_number, status, creator_name, target_regional_name |

---

## B-06 — Checklist Template Digital

### TC-B06-01: Template con 9 secciones
| Campo | Detalle |
|---|---|
| **CA** | CA-01 |
| **Endpoint** | `GET /api/v1/checklist/template` |
| **Esperado** | HTTP 200, 9 secciones (Exterior, Limpieza, Exhibicion, Precios, Neveras, Puesto de Pago, Bodega, Documentos, Empleados), 34 items total |

### TC-B06-02: Crear sesión de checklist
| Campo | Detalle |
|---|---|
| **CA** | CA-02 |
| **Endpoint** | `POST /api/v1/checklist/sessions` con store_id y visit_date |
| **Auth** | JZ |
| **Esperado** | HTTP 201, sesión con status IN_PROGRESS, items pre-poblados del template |

### TC-B06-03: Registrar item NO_CUMPLE con foto
| Campo | Detalle |
|---|---|
| **CA** | CA-03 |
| **Precondición** | Sesión IN_PROGRESS del TC-B06-02 |
| **Endpoint** | `POST /api/v1/checklist/sessions/{id}/items` con `result: NO_CUMPLE`, observation, photo_url |
| **Esperado** | HTTP 200, item actualizado con result, observation, photo_url |

### TC-B06-04: Completar sesión — score + findings
| Campo | Detalle |
|---|---|
| **CA** | CA-04, CA-05 |
| **Precondición** | Sesión con al menos 1 CUMPLE y 1 NO_CUMPLE |
| **Endpoint** | `POST /api/v1/checklist/sessions/{id}/complete` |
| **Esperado** | HTTP 200, `{ status: "COMPLETED", score: 50.0, findings_created: 1 }` |

### TC-B06-05: Agregar item variable
| Campo | Detalle |
|---|---|
| **CA** | CA-06 |
| **Precondición** | Sesión IN_PROGRESS |
| **Endpoint** | `POST /api/v1/checklist/sessions/{id}/variable-items` con `{"description":"...","section":"Exterior","source":"JZ"}` |
| **Esperado** | HTTP 200, variable item creado |

### TC-B06-06: Historial de sesiones
| Campo | Detalle |
|---|---|
| **CA** | — |
| **Endpoint** | `GET /api/v1/checklist/sessions` |
| **Esperado** | HTTP 200, array de sesiones con store_id, visit_date, status, score |

---

## B-07 — Follow-Up Item Entity

### TC-B07-01: Auto-creación desde checklist NO_CUMPLE
| Campo | Detalle |
|---|---|
| **CA** | CA-01, RB-01 |
| **Precondición** | Sesión de checklist completada con items NO_CUMPLE (TC-B06-04) |
| **Endpoint** | `GET /api/v1/followups` |
| **Esperado** | Al menos 1 FollowUpItem con `source_type: "CHECKLIST"`, `status: "OPEN"`, `auto_surface_in_next_checklist: true` |

### TC-B07-02: Resolver sin evidencia → 400
| Campo | Detalle |
|---|---|
| **CA** | CA-02, RB-02 |
| **Endpoint** | `PATCH /api/v1/followups/{id}/status` con `{"status":"RESOLVED"}` (sin notes ni photo) |
| **Esperado** | HTTP 400 "Evidence required" |

### TC-B07-03: Resolver con evidencia → 200
| Campo | Detalle |
|---|---|
| **CA** | CA-03 |
| **Endpoint** | `PATCH /api/v1/followups/{id}/status` con `{"status":"RESOLVED","resolution_notes":"...","resolution_photo_url":"..."}` |
| **Esperado** | HTTP 200, status actualizado a RESOLVED |

### TC-B07-04: Owner no puede verificar propio
| Campo | Detalle |
|---|---|
| **CA** | CA-04, RB-03 |
| **Precondición** | FollowUpItem resuelto, owner es JZ |
| **Endpoint** | `PATCH /api/v1/followups/{id}/verify` con X-User-Id del JZ (owner) |
| **Esperado** | HTTP 403 "Owner cannot verify their own follow-ups" |

### TC-B07-05: Superior verifica → 200
| Campo | Detalle |
|---|---|
| **CA** | — |
| **Precondición** | FollowUpItem RESOLVED, owner es JZ |
| **Endpoint** | `PATCH /api/v1/followups/{id}/verify` con X-User-Id del Gte Ventas (superior) |
| **Esperado** | HTTP 200, status VERIFIED |

### TC-B07-06: VERIFIED inmutable
| Campo | Detalle |
|---|---|
| **CA** | CA-08, RB-08 |
| **Precondición** | FollowUpItem en status VERIFIED |
| **Endpoint** | `PATCH /api/v1/followups/{id}/status` con `{"status":"IN_PROGRESS"}` |
| **Esperado** | HTTP 409 "VERIFIED follow-ups cannot be updated" |

### TC-B07-07: Surface para checklist
| Campo | Detalle |
|---|---|
| **CA** | CA-05, RB-06 |
| **Endpoint** | `GET /api/v1/followups/surface/checklist/{store_id}` |
| **Esperado** | HTTP 200, array de FollowUpItems con `auto_surface_in_next_checklist: true` para esa tienda |

### TC-B07-08: Filtro overdue
| Campo | Detalle |
|---|---|
| **CA** | CA-07 |
| **Endpoint** | `GET /api/v1/followups?overdue=true` |
| **Esperado** | HTTP 200, solo items con deadline < hoy y status OPEN/IN_PROGRESS |

---

## Resumen de cobertura

| Feature | Test Cases | CAs cubiertos | RBs cubiertos |
|---|---|---|---|
| B-01 | 7 | CA-01 a CA-08 (excepto CA-07 migración) | RB-02, RB-03, RB-04, RB-06 |
| B-02 | 5 | CA-01 a CA-05 | RB-02, RB-03, RB-04 |
| B-03 | 3 | CA-01, CA-02, CA-03 | RB-03 |
| B-04 | 6 | CA-01, CA-02, CA-05, CA-06, CA-07 | RB-02, RB-04 |
| B-05 | 5 | CA-03, CA-04, CA-06 | RB-01, RB-04, RB-05 |
| B-06 | 6 | CA-01 a CA-06 | — |
| B-07 | 8 | CA-01 a CA-08 | RB-01, RB-02, RB-03, RB-06, RB-08 |
| **Total** | **40** | | |

---

*Test plan generado: 2026-03-29*
*Script automatizado: `grip-backend/tests/functional/test_mvp_blockers.sh`*
