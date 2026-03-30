# B-04 — Autenticación JWT

- **Estado:** en-implementación
- **Prioridad:** 🔴 Bloqueante MVP
- **Depende de:** B-01 (Employee entity), B-03 (Tenant)
- **SDD referencia:** Sección 5.1 — Employee Database (permisos por rol)
- **Última actualización:** 2026-03-29
- **Inicio implementación:** 2026-03-29

---

## 1. Objetivo

- **Aplicación(es):** grip-backend, grip-frontend
- **Problema:** El sistema actual usa headers mock (`X-User-Role`, `X-User-Zone-Id`) sin autenticación real. Cualquier persona puede hacer llamadas a la API con cualquier rol. En MVP se necesita autenticación real con JWT para que los guards de rol funcionen.
- **Alcance:**
  - Login con email + password → JWT
  - Refresh token
  - Dependency `get_current_employee` que reemplaza el mock actual
  - Guards de rol en endpoints existentes y nuevos
  - Auth service en Angular (login, almacenamiento de token, interceptor)
  - **Queda fuera:** OAuth/SSO, MFA (post-MVP), recuperación de contraseña (post-MVP)

---

## 2. Referencia a specs base

- SDD Master: `specs/GRIP-SDD-Functional-Specs.md` — Sección 5.1.3 (`EMP-08`, `EMP-09`)

---

## 3. Flujo de autenticación

```
[Frontend]                          [Backend]
   |                                    |
   |-- POST /api/v1/auth/login -------->|
   |   { email, password }             |
   |                                   | verifica password (bcrypt)
   |                                   | genera access_token (15min)
   |                                   | genera refresh_token (7d)
   |<-- { access_token, refresh_token, employee, role } --|
   |                                    |
   | [almacena tokens en localStorage] |
   |                                    |
   |-- GET /api/v1/findings ----------->|
   |   Authorization: Bearer <token>   |
   |                                   | valida JWT → extrae employee_id + tenant_id
   |<-- [ findings ] ------------------|
```

---

## 4. Modelo de datos

### Campo `password_hash` en `employees`

```sql
ALTER TABLE employees ADD COLUMN password_hash VARCHAR(255);
```

> En MVP se setea manualmente via script de seed. Login self-service (post-MVP).

---

## 5. JWT Payload

```json
{
  "sub": "<employee_id>",
  "tenant_id": "<tenant_id>",
  "role_code": "JEFE_ZONA",
  "hierarchy_level": 5,
  "can_execute_da": false,
  "can_execute_3point": false,
  "can_execute_checklist": true,
  "da_scope": "NONE",
  "exp": 1234567890
}
```

El token incluye los permission flags del rol para evitar queries adicionales en cada request.

---

## 6. Reglas de negocio

1. **RB-01:** `access_token` expira en 15 minutos. `refresh_token` expira en 7 días.
2. **RB-02:** El endpoint `POST /auth/refresh` acepta el `refresh_token` y retorna un nuevo `access_token`.
3. **RB-03:** Los permission flags en el JWT se sincronizan al hacer login o refresh — no son editables por el cliente.
4. **RB-04:** Un employee con `status = INACTIVE` no puede autenticarse.
5. **RB-05:** Los endpoints públicos son solo: `POST /auth/login`, `POST /auth/refresh`, `GET /health`.

---

## 7. Contrato API

| Método | Path | Descripción | Auth |
|---|---|---|---|
| `POST` | `/api/v1/auth/login` | Login con email + password | Público |
| `POST` | `/api/v1/auth/refresh` | Renovar access_token | Refresh token válido |
| `GET` | `/api/v1/auth/me` | Perfil del employee autenticado | Bearer token |

**Request `POST /auth/login`:**
```json
{ "email": "juan@ritmo.com", "password": "secret" }
```

**Response `POST /auth/login`:**
```json
{
  "access_token": "eyJ...",
  "refresh_token": "eyJ...",
  "token_type": "bearer",
  "employee": {
    "id": "uuid",
    "full_name": "Juan Pérez",
    "role_code": "JEFE_ZONA",
    "hierarchy_level": 5
  }
}
```

---

## 8. Guards de rol en backend

```python
# dependencies.py
def require_role(allowed_roles: list[str]):
    def guard(current_employee: Employee = Depends(get_current_employee)):
        if current_employee.role.code not in allowed_roles:
            raise HTTPException(status_code=403)
    return Depends(guard)

def require_can_execute_da():
    def guard(current_employee = Depends(get_current_employee)):
        if not current_employee.role.can_execute_da:
            raise HTTPException(status_code=403)
    return Depends(guard)
```

---

## 9. Frontend — Angular

```
core/services/auth.service.ts    ← login(), logout(), refreshToken(), getEmployee()
core/interceptors/auth.interceptor.ts ← añade Authorization: Bearer en cada request
core/guards/role.guard.ts        ← protege rutas por role_code o permission flag
```

`auth.service.ts` almacena tokens en `localStorage` y expone el employee actual como `BehaviorSubject`.

---

## 10. Orden de implementación

```
BACKEND:
1. pip install python-jose[cryptography] bcrypt passlib
2. app/core/security.py          ← create_token(), verify_token(), hash_password(), verify_password()
3. db/models/employees.py        ← añadir password_hash
4. alembic/versions/XXXX_b04_auth.py
5. api/auth.py                   ← /login, /refresh, /me
6. api/dependencies.py           ← get_current_employee() reemplaza mock

FRONTEND:
7. auth.service.ts               ← login, logout, refresh
8. auth.interceptor.ts           ← Bearer token en requests
9. login component               ← pantalla de login
```

---

## 11. Criterios de aceptación

- [x] **CA-01:** `POST /auth/login` con credenciales válidas retorna access_token y refresh_token
- [x] **CA-02:** `POST /auth/login` con credenciales inválidas retorna 401
- [x] **CA-03:** `GET /api/v1/findings` sin token retorna 401 (cuando no hay fallback headers)
- [x] **CA-04:** `GET /api/v1/findings` con token de role `JEFE_ZONA` retorna solo findings de su scope
- [x] **CA-05:** `POST /api/v1/da/create` con token de role `JEFE_ZONA` retorna 403
- [x] **CA-06:** `POST /auth/refresh` con refresh_token válido retorna nuevo access_token
- [x] **CA-07:** Employee con `status = INACTIVE` recibe 401 al intentar login
- [ ] **CA-08:** El interceptor Angular añade `Authorization: Bearer` automáticamente en todos los requests — pendiente frontend
