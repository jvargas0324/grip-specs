# GRIP — Handoff: Implementación Bloqueantes MVP (B-01 a B-07)

| Metadata | Valor |
|---|---|
| **Fecha** | 2026-03-29 |
| **Fase** | Implementación de Bloqueantes MVP |
| **Estado anterior** | Specs B-01 a B-07 cerradas ✅ |
| **Siguiente acción** | Implementar B-01 en grip-backend |

---

## 1. Contexto del proyecto

GRIP es una plataforma de control operacional para retail hard discount. El sistema tiene:
- Una **PoC existente** (demo a stakeholders) en `/Users/javivargas/Projects/tiendas-vive/weekly-apps/`
- Un **SDD v1.0.0** como fuente de verdad funcional
- Un **Gap Analysis** que documenta qué existe, qué rediseñar y qué construir

La PoC tiene un stack sólido (FastAPI + SQLAlchemy + Angular 21 + Gemini) pero una arquitectura de datos que no escala al sistema real. Los B items son los rediseños fundamentales que habilitan todo lo demás.

---

## 2. Archivos clave — leer antes de implementar

| Archivo | Propósito |
|---|---|
| `grip-specs/specs/GRIP-SDD-Functional-Specs.md` | SDD master v1.0.0 — fuente de verdad funcional |
| `grip-specs/specs/GRIP-GapAnalysis.md` | Qué existe en la PoC vs qué necesita el SDD |
| `grip-specs/specs/features/b01-*.md` … `b07-*.md` | Spec detallada de cada bloqueante |
| `grip-backend/app/core/ai_client.py` | Gemini client — conservar sin cambios |
| `grip-backend/app/db/models/` | Modelos existentes a refactorizar |
| `grip-backend/alembic/versions/` | Migraciones existentes (referencia) |

---

## 3. Restricciones fijas (no negociables)

| Restricción | Detalle |
|---|---|
| **Proveedor AI** | Gemini se conserva. `ai_client.py` no se modifica. |
| **SDD como fuente de verdad** | Si hay conflicto entre código existente y SDD, el SDD tiene razón |
| **PoC es referencia, no base** | No construir encima de la PoC — refactorizar con el SDD como guía |
| **Orden de trabajo** | grip-specs (si hay cambio de contrato) → grip-backend → grip-frontend |

---

## 4. Stack técnico (conservar)

| Capa | Tecnología |
|---|---|
| Backend | Python / FastAPI / SQLAlchemy 2.x / Alembic |
| Base de datos | PostgreSQL + pgvector (768 dims) |
| AI | Gemini gemini-2.5-flash-lite + gemini-embedding-001 |
| Frontend | Angular 21 + Tailwind CSS |

---

## 5. Backlog de bloqueantes — orden de implementación

Implementar **en este orden exacto** — cada uno desbloquea al siguiente:

| # | Spec | Descripción | Depende de |
|---|---|---|---|
| **B-01** | `b01-role-entity-configurable.md` | Entidad Role configurable + Employee (reemplaza User con enum hardcodeado) | — |
| **B-02** | `b02-reports-to-hierarchy.md` | Árbol jerárquico `reports_to` + endpoints de navegación | B-01 |
| **B-03** | `b03-tenant-isolation.md` | Tabla `tenants` + `tenant_id` en todas las tablas | B-01 |
| **B-04** | `b04-authentication-jwt.md` | Login JWT + guards de rol + interceptor Angular | B-01, B-03 |
| **B-05** | `b05-da-coo-scope.md` | DA: habilitar COO + scope OPERATIONS_ONLY + auto-surface | B-01, B-04 |
| **B-06** | `b06-checklist-template.md` | ChecklistTemplate digital + sesión de visita + UI mobile | B-01, B-03, B-04 |
| **B-07** | `b07-followup-item-entity.md` | FollowUpItem separado de Finding + auto-surface flags + job escalación | B-01, B-03, B-04, B-06 |

---

## 6. Protocolo por cada bloqueante

Para cada B item seguir este orden:

```
1. Leer la spec completa: grip-specs/specs/features/b0X-*.md
2. BACKEND:
   a. Crear/modificar modelos en grip-backend/app/db/models/
   b. Crear migración Alembic (con seed data si aplica)
   c. Crear schemas Pydantic en grip-backend/app/schemas/
   d. Crear/modificar service en grip-backend/app/services/
   e. Crear/modificar router en grip-backend/app/api/
   f. Registrar router en grip-backend/app/main.py si es nuevo
3. FRONTEND (si la spec lo requiere):
   a. Service en grip-frontend/src/app/features/<modulo>/services/
   b. Componentes Angular
4. VALIDAR todos los Criterios de Aceptación de la spec
5. Actualizar grip-specs/specs/backend_audit_report.md con los endpoints nuevos
```

---

## 7. Cómo arrancar B-01 en sesión nueva

Prompt de inicio recomendado:

```
Lee estos archivos antes de comenzar:
- @grip-specs/specs/features/b01-role-entity-configurable.md
- @grip-specs/specs/GRIP-SDD-Functional-Specs.md (sección 5.1)
- @grip-backend/app/db/models/users.py (modelo actual a reemplazar)

Tarea: implementar B-01 en grip-backend siguiendo exactamente
el orden de implementación de la spec (8 archivos en secuencia).
Empezar por db/models/roles.py.
```

---

## 8. Lo que NO tocar durante la implementación de B items

| Archivo / Módulo | Razón |
|---|---|
| `grip-backend/app/core/ai_client.py` | Conservar sin cambios — es el componente mejor construido |
| `grip-backend/app/services/ingestion_pdf_service.py` | Canal PDF se conserva como backup — no priorizar |
| `grip-backend/app/modules/rag_memory/` | RAG es Phase 2 — no modificar |
| `grip-specs/docs/archive/` | Specs obsoletas de la PoC — no referenciar |

---

## 9. Después de B-01 a B-07

Una vez implementados y validados los 7 bloqueantes, el siguiente paso es:

1. **Especificar C-01 a C-12** (Core MVP) — con el modelo real como referencia
2. **Implementar C items** en el mismo orden spec → backend → frontend

Los C items son:
- C-01: Checklist digital UI mobile-first completa
- C-02: Ítems variables + AI-08 (sugerencias)
- C-03: WeeklyReport con cascade 4 niveles
- C-04: AspectLibrary + ThreePointReview completo
- C-05: Auto-surface engine (job diario)
- C-06: Escalación automática por deadline
- C-07: Dashboards restantes (Gte Ventas, Regional, COO, CEO, JT)
- C-08 a C-12: AI prompts (AI-01, AI-02, AI-03, AI-05, AI-06)

---

## 10. Estado del repo al cierre de esta sesión

```
weekly-apps/
  grip-specs/
    specs/
      GRIP-SDD-Functional-Specs.md    ✅ SSOT master v1.0.0
      GRIP-GapAnalysis.md             ✅ Gap analysis completo
      api-contract-ssot.md            ⏳ Pendiente rebuild
      backend_audit_report.md         ⏳ Pendiente rebuild
      features/
        b01-role-entity-configurable  ✅ spec-cerrada
        b02-reports-to-hierarchy      ✅ spec-cerrada
        b03-tenant-isolation          ✅ spec-cerrada
        b04-authentication-jwt        ✅ spec-cerrada
        b05-da-coo-scope              ✅ spec-cerrada
        b06-checklist-template        ✅ spec-cerrada
        b07-followup-item-entity      ✅ spec-cerrada
    docs/
      archive/
        functional-spec-poc-v1.md     📦 archivado
        technical-spec-poc-v7.20.md   📦 archivado
        poc-features/                 📦 4 feature specs archivadas
    CLAUDE.md                         ✅ actualizado con nuevo SSOT

  grip-backend/                       ⏳ pendiente implementación B-01
  grip-frontend/                      ⏳ pendiente implementación B-04, B-06, B-07
```

---

*Handoff generado al cierre de Fase B + setup de backlog — 2026-03-29*
*Próxima sesión: implementación B-01 → B-07 en grip-backend*
