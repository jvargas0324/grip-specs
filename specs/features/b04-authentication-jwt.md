# B-04 — Autenticacion JWT

- **Estado:** en-implementacion
- **Prioridad:** Bloqueante MVP
- **Depende de:** B-01 (Employee entity), B-03 (Tenant)
- **SDD referencia:** Seccion 5.1.3 (EMP-01 a EMP-06, permisos por rol), §8.4 (Security — JWT, RBAC, session timeout), Appendix B (Auth endpoints)
- **Ultima actualizacion:** 2026-03-29
- **Inicio implementacion:** 2026-03-29

---

## 1. Objetivo

- **Aplicacion(es):** grip-backend, grip-frontend
- **Problema:** El sistema usaba headers mock (`X-User-Role`, `X-User-Zone-Id`) sin autenticacion real. Cualquier persona podia llamar la API con cualquier rol. MVP necesita autenticacion real con JWT para que los guards de rol funcionen.
- **Alcance:**
  - Login con email + password -> JWT (access + refresh)
  - Refresh token
  - Dependency `get_current_user` con fallback legacy para transicion
  - Guards de rol en endpoints existentes y nuevos
  - Auth service en Angular (login, almacenamiento de token, interceptor, auth guard)
  - Login component con redirect a dashboard
  - **Queda fuera:** OAuth/SSO, MFA (post-MVP P-05), recuperacion de contrasena (post-MVP), logout endpoint backend (MVP usa client-side token clearing)

## 2. Referencia a specs base

- Employee y roles: `GRIP-SDD-Functional-Specs.md` §5.1.3 (EMP-01 a EMP-06) — CRUD, roles jerarquicos, cascada visibilidad
- Business rules: `GRIP-SDD-Functional-Specs.md` §5.1.4 (BR-EMP-01 a BR-EMP-05) — permisos derivados de Role, no de nivel
- Security NFRs: `GRIP-SDD-Functional-Specs.md` §8.4 — JWT, RBAC, password policy, session timeout
- Auth endpoints: `GRIP-SDD-Functional-Specs.md` Appendix B — `/api/auth/login`, `/api/auth/refresh`, `/api/auth/logout`

## 3. Actores y permisos

| Rol | Permiso |
|-----|---------|
| Cualquier employee ACTIVE | Login con email + password |
| Cualquier employee autenticado | Refresh token, ver /me |
| Employee INACTIVE | Login rechazado (401) |

No se crean permisos nuevos — los permission flags (`can_execute_da`, `can_execute_3point`, `can_execute_checklist`) ya existen en Role (B-01) y se propagan en el JWT.

## 4. Flujos principales

### 4.1 Login

1. Employee abre la app -> auth guard redirige a `/login`.
2. Ingresa email + password.
3. Backend verifica password (bcrypt), valida status = ACTIVE.
4. Genera access_token (15min) y refresh_token (7d).
5. Frontend almacena tokens en localStorage.
6. Redirige a `/dashboard`.

### 4.2 Request autenticado

1. Interceptor Angular agrega `Authorization: Bearer <access_token>` a cada request.
2. Endpoints publicos (`/auth/login`, `/auth/refresh`, `/health`) no requieren token.
3. Backend valida JWT -> extrae employee_id + tenant_id + permission flags.

### 4.3 Token refresh

1. Si un request retorna 401, el interceptor intenta refresh automatico.
2. Envia refresh_token a `POST /auth/refresh`.
3. Backend valida refresh token, re-lee permisos de DB (sincroniza si rol cambio).
4. Retorna nuevo access_token.
5. Interceptor reintenta el request original con el nuevo token.

### 4.4 Logout

1. Frontend limpia tokens de localStorage.
2. Redirige a `/login`.
3. No hay endpoint backend de logout en MVP (stateless JWT — el token expira solo).

### 4.5 Session restore

1. Al abrir la app, auth service lee access_token de localStorage.
2. Decodifica JWT para restaurar user state sin network call.
3. Si expirado, intenta refresh. Si refresh falla, redirige a login.
4. En background, llama `/auth/me` para sincronizar datos frescos.

### Excepciones

- Employee INACTIVE: 401 en login con mensaje claro.
- Token expirado sin refresh_token: redirige a login.
- Credenciales incorrectas: 401 con mensaje generico (no revelar si email existe).

## 5. Reglas de negocio

1. **RB-01:** `access_token` expira en 15 minutos. `refresh_token` expira en 7 dias.
2. **RB-02:** `POST /auth/refresh` acepta refresh_token y retorna nuevo access_token. No rota el refresh_token.
3. **RB-03:** Permission flags en el JWT se sincronizan desde DB en cada login y refresh — no son editables por el cliente.
4. **RB-04:** Employee con `status = INACTIVE` no puede autenticarse.
5. **RB-05:** Endpoints publicos: `POST /auth/login`, `POST /auth/refresh`, `GET /health`.
6. **RB-06:** Backend soporta fallback a headers `X-User-*` para transicion (legacy dev/test). JWT tiene prioridad.

> **Nota sobre session timeout vs SDD §8.4:** El SDD define "default: 8 hours mobile, 2 hours web". En MVP esto se logra con la combinacion access_token (15min) + refresh_token (7d): la sesion efectiva dura mientras el refresh sea valido. Los timeouts mas cortos del SDD se implementaran post-MVP con idle timeout en frontend.

## 6. Contrato API

| Metodo | Path | Descripcion | Auth |
|--------|------|-------------|------|
| `POST` | `/api/v1/auth/login` | Login con email + password | Publico |
| `POST` | `/api/v1/auth/refresh` | Renovar access_token | Refresh token valido |
| `GET` | `/api/v1/auth/me` | Perfil del employee autenticado | Bearer token |

> **Nota sobre path prefix:** SDD Appendix B usa `/api/auth/*`. La implementacion usa `/api/v1/auth/*` para consistencia con el resto de la API versionada. Es una decision de implementacion, no contradiccion.

**Request `POST /auth/login`:**
```json
{ "email": "jz@ritmo.dev", "password": "grip2026" }
```

**Response `POST /auth/login`:**
```json
{
  "access_token": "eyJ...",
  "refresh_token": "eyJ...",
  "token_type": "bearer",
  "employee": {
    "id": "uuid",
    "full_name": "Juan Perez",
    "role_code": "JEFE_ZONA",
    "hierarchy_level": 5
  }
}
```

**Request `POST /auth/refresh`:**
```json
{ "refresh_token": "eyJ..." }
```

**Response `POST /auth/refresh`:**
```json
{
  "access_token": "eyJ...",
  "token_type": "bearer"
}
```

**Response `GET /auth/me`:**
```json
{
  "id": "uuid",
  "employee_code": "EMP-001",
  "full_name": "Juan Perez",
  "email": "jz@ritmo.dev",
  "role_code": "JEFE_ZONA",
  "hierarchy_level": 5,
  "functional_area": "VENTAS",
  "status": "ACTIVE"
}
```

## 7. JWT Payload

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
  "type": "access",
  "exp": 1234567890
}
```

Refresh token solo contiene: `sub`, `tenant_id`, `type: "refresh"`, `exp`.

## 8. Modelo de datos

### Campo `password_hash` en `employees`

```sql
ALTER TABLE employees ADD COLUMN password_hash VARCHAR(255);
```

Migrado en Alembic 0015. En MVP se setea via seed script (`grip2026` para dev employees). Self-service password reset es post-MVP.

## 9. Guards de rol en backend

```python
# dependencies.py — get_current_user()
# Prioridad: 1. JWT Bearer, 2. X-User-Id + DB lookup, 3. X-User-Role header (legacy)

# Uso en endpoints:
def require_role(allowed_roles: list[str]):
    def guard(current_user = Depends(get_current_user)):
        if current_user.role_code not in allowed_roles:
            raise HTTPException(status_code=403)
    return Depends(guard)
```

## 10. Frontend — Angular

### Archivos

| Archivo | Descripcion |
|---------|-------------|
| `core/services/auth.service.ts` | login(), logout(), refreshToken(), session restore, BehaviorSubject<CurrentUser> |
| `core/interceptors/auth.interceptor.ts` | Authorization: Bearer en requests, auto-refresh en 401, skip en publicos |
| `core/guards/auth.guard.ts` | Redirige a /login si no hay token |
| `core/guards/role.guard.ts` | Protege rutas por UserRole (actualizado para CurrentUser nullable) |
| `core/components/login/login.component.ts` | Formulario email + password, error handling, redirect |

### Mapeo role_code -> UserRole (frontend)

```
JEFE_TIENDA, JEFE_ZONA -> 'jz'
GTE_VENTAS, GTE_LOGISTICO -> 'gv'
GERENTE_REGIONAL -> 'regional'
COO -> 'coo'
CEO, CFO, CTO, CPO -> 'ceo'
```

### Cambios en main-layout

- Eliminado: mock role selector, headers X-User-*.
- Agregado: user info en sidebar (nombre, email, roleCode), boton "Cerrar sesion".
- Sidebar navigation filtrada por rol real del JWT.

## 11. Decisiones de implementacion

| Decision | Contexto |
|----------|----------|
| **bcrypt directo (no passlib)** | passlib tiene incompatibilidad con bcrypt 5.x. Se usa `bcrypt` directamente. |
| **JWT + legacy headers coexisten** | `get_current_user()` intenta Bearer primero, luego fallback a `X-User-*` para transicion gradual. |
| **No logout endpoint backend** | JWT es stateless. MVP usa client-side token clearing. Server-side blacklist es post-MVP. |
| **`/api/v1/` prefix** | SDD Appendix B usa `/api/`. Implementacion usa `/api/v1/` para versionado consistente. |
| **access_token 15min** | SDD §8.4 dice 2h web / 8h mobile. MVP usa 15min + 7d refresh. Idle timeout en frontend es post-MVP. |

## 12. Criterios de aceptacion

### Backend (todos completados)

- [x] **CA-01:** `POST /auth/login` con credenciales validas retorna access_token y refresh_token
- [x] **CA-02:** `POST /auth/login` con credenciales invalidas retorna 401
- [x] **CA-03:** `GET /api/v1/findings` sin token retorna 401 (cuando no hay fallback headers)
- [x] **CA-04:** `GET /api/v1/findings` con token de role `JEFE_ZONA` retorna solo findings de su scope
- [x] **CA-05:** `POST /api/v1/da/create` con token de role `JEFE_ZONA` retorna 403
- [x] **CA-06:** `POST /auth/refresh` con refresh_token valido retorna nuevo access_token
- [x] **CA-07:** Employee con `status = INACTIVE` recibe 401 al intentar login

### Frontend (todos completados)

- [x] **CA-08:** Interceptor Angular agrega `Authorization: Bearer` automaticamente en todos los requests
- [x] **CA-09:** Auth guard redirige a `/login` si no hay token
- [x] **CA-10:** Login component envia credenciales y almacena tokens en localStorage
- [x] **CA-11:** Interceptor reintenta request tras refresh automatico en 401
- [x] **CA-12:** Logout limpia tokens y redirige a `/login`
- [x] **CA-13:** Session restore al recargar la app (decodifica JWT sin network call)
- [x] **CA-14:** Main layout muestra user real del JWT (nombre, email, roleCode) sin mock selector
