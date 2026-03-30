# GRIP — Gap Analysis
## Auditoría SDD v1.0.0 vs Implementación PoC

| Metadata | Valor |
|---|---|
| **Fecha** | 2026-03-29 |
| **SDD auditado** | v1.0.0 — Approved |
| **Repo auditado** | `/Users/javivargas/Projects/tiendas-vive/weekly-apps/` |
| **Autor** | Javier Vargas — AI-Assisted (Claude) |

### Notación

| Símbolo | Significado |
|---|---|
| ✅ | Existe y es reutilizable con ajustes menores |
| ⚙️ | Existe pero necesita rediseño para cumplir el SDD |
| 🔴 | No existe — debe construirse desde cero |
| ⚠️ | Existe pero con arquitectura cuestionable — requiere decisión |

---

## Resumen Ejecutivo

| Categoría | Módulos / Componentes |
|---|---|
| ✅ Reutilizable | Stack técnico, ai_client.py (Gemini), Finding model (base), Visit model (base), Alembic setup |
| ⚙️ Rediseñar | DA Module, Weekly Reports, Follow-Up (status tracking), User model |
| 🔴 Construir | 3-Point Review, 36-Point Aspect Library, Checklist Template digital, Role entity configurable, Tenant isolation, `reports_to` hierarchy, Auto-surface flags, Dashboards role-based, AI prep briefs, Pattern detection, Early warning |
| ⚠️ Challenge | PDF ingestion (vs checklist digital), RAG/Ask GRIP (fuera de SDD scope), DA CEO-only (vs CEO+COO), Zona plana (vs jerarquía flexible), Semantic matching pgvector |

**Conclusión de alto nivel:** La PoC estableció un stack técnico sólido y validó la integración con Gemini. Sin embargo, la arquitectura de datos y la mayoría de los módulos funcionales deben rediseñarse o construirse desde cero para cumplir el SDD. No es una refactoring incremental — es una construcción con base en el stack existente.

---

## Stack Tecnológico

### Encontrado en el repo

| Capa | Tecnología | Versión |
|---|---|---|
| **Backend** | Python / FastAPI | FastAPI ≥ 0.109 |
| **ORM** | SQLAlchemy | ≥ 2.0.25 |
| **Migraciones** | Alembic | ≥ 1.13 |
| **DB** | PostgreSQL + pgvector | pgvector ≥ 0.2.4 |
| **Validación** | Pydantic v2 | (implícito en FastAPI moderno) |
| **AI** | Google Gemini | google-generativeai ≥ 0.8.0 |
| **Modelo texto** | gemini-2.5-flash-lite | — |
| **Modelo embedding** | gemini-embedding-001 | 768 dims |
| **Frontend** | Angular | v21.2 |
| **CSS** | Tailwind CSS | v3.4 |
| **PDF parsing** | pypdf | ≥ 5.0.1 |

### Evaluación

| Componente | Evaluación | Nota |
|---|---|---|
| FastAPI + SQLAlchemy + Alembic | ✅ | Stack sólido para producción |
| PostgreSQL + pgvector | ✅ | Correcto para datos relacionales + embeddings |
| Gemini gemini-2.5-flash-lite | ✅ | Modelo adecuado. Conservar según restricción |
| gemini-embedding-001 (768 dims) | ✅ | Coherente con el modelo de embeddings |
| Angular 21 | ✅ | Framework moderno, mobile-capable |
| Tailwind CSS | ✅ | Alineado con diseño "tap-not-type" del SDD |
| pypdf (PDF ingestion) | ⚠️ | Ver challenge C-01 |
| **Sin autenticación real** | 🔴 | No existe JWT, OAuth ni sesión real. Todo el auth es mock. |
| **Sin multi-tenancy** | 🔴 | No existe tenant_id en ninguna tabla |

---

## Auditoría por Módulo

---

### 5.1 Employee Database (Hub Central)

**Categoría: ⚙️ Rediseñar**

#### Lo que existe

```python
# users.py
USER_ROLE = Enum("jz", "gv", "regional", "coo", "ceo")

class User(Base):
    id, email, name, role, zone_id
```

```python
# zones.py
class Zone(Base):
    id, name
```

#### Gaps vs SDD

| Requisito SDD | Estado | Detalle |
|---|---|---|
| `Role` como entidad configurable (no enum) | 🔴 | Roles hardcodeados como Enum de DB. No es configurable por tenant. Rompe el modelo de Ritmo que tiene Gte. Logístico, Gte. Regional, etc. |
| `Employee.reports_to` (árbol jerárquico flexible) | 🔴 | El User tiene `zone_id` (FK a zona), no `reports_to` (FK a Employee). La jerarquía es implícita por zona, no explícita por árbol. |
| `Employee.functional_area` | 🔴 | No existe. Necesario para diferenciar Ventas vs Logística vs Admin |
| `Employee.employee_code` | 🔴 | No existe. Solo email como identificador |
| `Employee.assigned_stores[]` | 🔴 | No existe relación directa Employee → Stores |
| `Employee.hire_date`, `locale`, `timezone` | 🔴 | No existen |
| `Employee.status` (ACTIVE/INACTIVE) | 🔴 | No existe |
| `Role.can_execute_da`, `can_execute_3point_review`, etc. | 🔴 | No existe. Todo está hardcodeado en if/else por rol |
| `Role.da_scope` (NONE / OPERATIONS_ONLY / ALL_AREAS) | 🔴 | No existe. DA solo permite CEO (hardcodeado en da_service.py) |
| `tenant_id` en User | 🔴 | No existe. Sin multi-tenancy |

#### Recomendación

Rediseñar completamente el modelo de User hacia el Employee + Role del SDD. La entidad Zone puede mantenerse como concepto de agrupación geográfica, pero la jerarquía de mando debe vivir en `reports_to`.

---

### 5.2 Weekly Reports

**Categoría: ⚙️ Rediseñar**

#### Lo que existe

```python
# weekly_sessions.py
class WeeklySession(Base):
    id, organizer_id, scope_type (zonal/regional/consolidated),
    start_date, end_date, status (scheduled/in_progress/completed),
    snapshot_data (JSONB)
```

```python
# strategic_feedback.py (inferido de da_service.py)
class StrategicFeedback(Base):
    author_id, target_user_id, session_id,
    question_1, question_2, question_3,
    response_text, embedding
```

Frontend: `weekly-summary`, `weekly-feedback`, `compliance-chart` components.

#### Gaps vs SDD

| Requisito SDD | Estado | Detalle |
|---|---|---|
| Weekly Report como documento escrito por nivel (JZ → Gte Ventas → Regional → COO) | ⚙️ | La PoC modela "sesiones" de reunión, no reportes escritos. El cascade 4 niveles no existe. |
| `WeeklyReport.content` (texto libre) | ⚙️ | Existe como `snapshot_data JSONB` pero sin estructura definida |
| `WeeklyReport.period_week`, `period_year` | 🔴 | No existe. Solo start_date/end_date de sesión |
| `WeeklyReport.author_id` por nivel jerárquico | ⚙️ | Existe `organizer_id` pero sin distinción de nivel |
| Cascade up: cada nivel agrega perspectiva al reporte del nivel anterior | 🔴 | No existe. La sesión no tiene `parent_report_id` |
| `WeeklyReport.ai_summary`, `focal_points` | ⚙️ | Parcialmente implementado vía Gemini pero sin estructura de output schema |
| 3 preguntas estratégicas como parte del reporte | ⚠️ | Existe `strategic_feedback` con question_1/2/3, pero como entidad separada, no integrada al reporte |

#### Recomendación

Mantener el concepto de sesión semanal pero rediseñar hacia `WeeklyReport` con cascade explícito y niveles. El `StrategicFeedback` puede evolucionar hacia las 3 preguntas del SDD integradas al reporte.

---

### 5.3 Store Visit Checklist

**Categoría: ⚠️ Challenge (ver C-01)**

#### Lo que existe

```python
# visits.py
class Visit(Base):
    id, store_id, jz_id, visit_date, raw_file_url
```

Frontend: `checklist-form` component con dos canales: formulario manual + PDF.

Módulo `ingestion` con `ingestion_pdf_service.py` para extracción AI de PDFs.

#### Gaps vs SDD

| Requisito SDD | Estado | Detalle |
|---|---|---|
| `ChecklistTemplate` con ítems fijos (HQ) | 🔴 | No existe. No hay catálogo de ítems del checklist |
| Ítems variables por visita (JZ o AI) | 🔴 | No existe estructura para ítems variables |
| 9 secciones del checklist (Exterior, Limpieza, etc.) | 🔴 | No hay estructura por sección |
| `ChecklistItem.result` (CUMPLE / NO_CUMPLE / NA) | ⚙️ | Existe `Finding.compliance` (boolean) pero no por ítem del template |
| Foto integrada inline al hallazgo (≤ 4 taps) | ⚠️ | Existe `evidences.py` pero no integrado al flujo de checklist |
| Canal digital mobile-first (tap, no type) | ⚠️ | El formulario actual es un form web, no optimizado para mobile |
| Canal PDF como alternativa | ⚠️ | La PoC lo implementó. Ver challenge C-01 |
| `auto_surface_in_next_checklist = TRUE` al crear finding | 🔴 | No existe |
| Score de visita agregado por sección | 🔴 | No existe |

#### Challenge C-01: PDF ingestion vs Checklist digital

> **Situación:** La PoC construyó un pipeline de ingesta de PDF (JZ sube el PDF del checklist físico, la IA lo extrae). El SDD especifica un checklist digital mobile-first donde el JZ llena el formulario en la app.
>
> **El problema:** El PDF ingestion es una solución de demo que invierte el flujo correcto. En lugar de que el sistema genere la estructura y el JZ la llene, el JZ usa papel y luego el sistema intenta reconstruir lo que pasó.
>
> **Recomendación:** El canal PDF puede conservarse como entrada de respaldo (para tiendas con conectividad limitada o transición desde papel), pero el canal primario debe ser el checklist digital del SDD. El pipeline de PDF no debe ser el mecanismo principal de captura de datos operativos.

---

### 5.4 36-Point Annual Review

**Categoría: 🔴 Construir desde cero**

#### Lo que existe

Nada. No hay ningún modelo, servicio, API endpoint ni componente de frontend relacionado con el 3-Point Review mensual o la biblioteca de 36 aspectos.

La única referencia indirecta es `strategic_feedback.question_1/2/3` — pero esas son las 3 preguntas del reporte semanal, no el mecanismo de evaluación de desempeño.

#### Lo que debe construirse

- `AspectLibrary` — catálogo de 36 aspectos configurables por tenant
- `ThreePointReview` — sesión de evaluación mensual
- `ThreePointReviewItem` — 3 aspectos seleccionados con calificación y comentario
- Lógica de selección: coverage gaps, aspectos nunca evaluados, señales en reportes
- API endpoints CRUD completos
- UI para ejecutor (selecciona 3 aspectos, califica, comenta) y receptor (lee su evaluación)
- AI-02: preparation brief para 3-Point Review

---

### 5.5 DA Module (Dienstaufsicht)

**Categoría: ⚙️ Rediseñar**

#### Lo que existe

```python
# da_interventions.py
class DAIntervention(Base):
    id, case_number, creator_id, target_regional_id, store_id,
    status (active/pending_validation/closed_resolved/closed_unresolved),
    proven_facts, root_cause_analysis, structural_action_plan,
    due_date, closed_at, regional_signature_hash, ceo_signature_hash

# da_findings_map.py
class DAFindingsMap(Base):  # linking DA → findings
```

`da_service.py` implementa `create_da` y `close_da` con Gemini para generar `proven_facts` y `root_cause_analysis`.

#### Gaps vs SDD

| Requisito SDD | Estado | Detalle |
|---|---|---|
| CEO puede ejecutar DA (ALL_AREAS) | ✅ | Implementado |
| COO puede ejecutar DA (OPERATIONS_ONLY) | 🔴 | `create_da` rechaza todo lo que no sea `role == "ceo"`. COO no puede crear DA. |
| `da_scope` para delimitar qué áreas puede auditar | 🔴 | No existe. CEO tiene acceso total sin restricción por área |
| DA genera findings con owner + deadline + auto-surface | ⚙️ | DA genera `proven_facts` pero los hallazgos del DA no se convierten en FollowUpItems con lifecycle propio |
| Feedback loop: findings de DA resurgen en siguiente DA | 🔴 | No existe auto-surface entre ciclos de DA |
| DA puede generar ítems variables en checklists de tiendas afectadas | 🔴 | No existe cross-module cascade |
| AI-03: DA Preparation Brief | 🔴 | No existe. La IA solo actúa durante la creación, no en la preparación |
| Firma doble (regional + ejecutor) | ⚙️ | Existe `regional_signature_hash` + `ceo_signature_hash`. Implementado a nivel de hash string, MFA pendiente según arquitectura existente |

#### Lo rescatable

- El modelo `DAIntervention` con `proven_facts`, `root_cause_analysis`, `structural_action_plan` es sólido y alineado con el SDD.
- La generación AI de `proven_facts` vía Gemini (con `_gather_store_history`) es un buen patrón reutilizable.
- `DAFindingsMap` como tabla de vinculación es correcto.
- El sistema de firma doble como concepto es válido — necesita madurar.

#### Recomendación

Refactorizar `da_service.py` para eliminar el hardcode CEO-only y añadir scope-based access. Añadir generación de FollowUpItems al crear DA. Añadir auto-surface flags.

---

### 5.6 Follow-Up System

**Categoría: ⚙️ Rediseñar**

#### Lo que existe

```python
# findings.py
FINDING_STATUS = Enum("open", "in_progress", "verified", "reopened")

class Finding(Base):
    ..., status, parent_finding_id,  # para reincidencias
    embedding  # Vector(768) para semantic matching
```

Frontend: botón "Resolver" en dashboard JZ que abre modal para cambiar status + adjuntar evidencia.

```python
# evidences.py (inferido)
class Evidence(Base):  # foto/archivo adjunto a finding
```

#### Gaps vs SDD

| Requisito SDD | Estado | Detalle |
|---|---|---|
| `FollowUpItem` como entidad separada del `Finding` | ⚙️ | La PoC mezcla Finding (el hallazgo observado) y FollowUpItem (la tarea de resolución) en una sola tabla. El SDD los separa. |
| `auto_surface_in_next_checklist` | 🔴 | No existe. Los findings no resurgen automáticamente en la siguiente visita |
| `auto_surface_in_next_da` | 🔴 | No existe |
| `auto_surface_in_next_review` | 🔴 | No existe |
| Lifecycle: OPEN → IN_PROGRESS → RESOLVED → VERIFIED | ⚙️ | Existe pero con estados ligeramente distintos (verified ≠ resolved+verified). "reopened" no está en SDD |
| `FollowUpItem.source_type` (CHECKLIST / DA / THREE_POINT / WEEKLY_REPORT) | 🔴 | No existe. No hay trazabilidad de dónde vino el finding |
| Evidencia obligatoria para VERIFIED | ✅ | Implementado: "añadir evidencia es OBLIGATORIO para marcar como verified" |
| Escalación automática si deadline vence | 🔴 | No existe job de escalación |
| `verified_by` (quién cierra) | 🔴 | No existe |

#### Lo rescatable

- El patrón de `parent_finding_id` para rastrear reincidencias es una buena idea — no está en el SDD pero puede integrarse como enriquecimiento del FollowUpItem.
- El embedding en Finding para semantic matching es valioso — ver challenge C-02.
- La evidencia obligatoria para cierre ya está implementada.

#### Challenge C-02: pgvector + Semantic Matching

> **Situación:** La PoC implementó pgvector con embeddings (Gemini gemini-embedding-001, 768 dims) sobre cada Finding. Esto permite detectar reincidencias semánticas: "vidrios sucios" y "pegotes en fachada" se reconocen como el mismo problema aunque estén escritos diferente.
>
> **El SDD no especifica esto** — el SDD usa `parent_finding_id` y `auto_surface_in_next_checklist` como mecanismo de reincidencia.
>
> **Evaluación:** El semantic matching es una capacidad valiosa y complementaria. No contradice el SDD — lo enriquece. El `parent_finding_id` + semantic matching juntos son más poderosos que cualquiera por separado.
>
> **Recomendación:** Conservar pgvector + embeddings. Documentarlo como capacidad del Follow-Up System en el SDD (adición al modelo de datos de Finding). No reemplaza los auto_surface flags — los complementa.

---

### 5.7 Dashboards (Role-Based)

**Categoría: ⚙️ Rediseñar**

#### Lo que existe

Frontend: `dashboard-jz` component — único dashboard implementado, exclusivo para Jefe de Zona.

Funcionalidades del dashboard JZ:
- Lista de findings con filtros (tienda, estado, severidad)
- Búsqueda server-side paginada
- Modal "Resolver" con cambio de estado y evidencia
- Compliance chart (gráfico de cumplimiento)

#### Gaps vs SDD

| Dashboard SDD | Estado | Detalle |
|---|---|---|
| Dashboard Jefe de Zona | ⚙️ | Existe pero incompleto. Falta: AI prep brief, open follow-ups con auto-surface, score de visita |
| Dashboard Gerente de Ventas / Logístico | 🔴 | No existe |
| Dashboard Gerente Regional | 🔴 | No existe |
| Dashboard COO | 🔴 | No existe |
| Dashboard CEO | 🔴 | No existe (aunque hay endpoints con bypass de CEO en algunas queries) |
| Dashboard Jefe de Tienda | 🔴 | No existe |
| Widget Cross-Reference Matrix (SDD 5.7) | 🔴 | No existe |
| Heatmaps por zona/región | 🔴 | No existe |
| Early warning widget | 🔴 | No existe |

#### Recomendación

El dashboard JZ es una base válida de referencia de UX. Debe extenderse (no reemplazarse) siguiendo la Widget Cross-Reference Matrix del SDD 5.7. Los otros 5 dashboards deben construirse desde cero.

---

### 5.8 AI Assistant Layer (Gemini)

**Categoría: ⚙️ Rediseñar**

#### Lo que existe

```python
# ai_client.py
- generate_structured_text(prompt, system_instruction, use_json=True)
- generate_plain_text(prompt, system_instruction)
- generate_embedding(text) → list[float] (768 dims)
- Modelo: gemini-2.5-flash-lite
- JSON output vía response_mime_type="application/json"
- Error handling: GeminiAPIError, GeminiQuotaError, GeminiTimeoutError
```

Actividades AI implementadas en la PoC:
- Normalización de hallazgos (categoria, severidad, resumen, accionable) — en ingestion
- Semantic matching de reincidencias — en follow_up (via embeddings)
- Generación de `proven_facts` y `root_cause_analysis` para DA — en da_service
- AI summary semanal (focal_points, executive brief) — en weekly_hub
- RAG/Ask GRIP conversacional — en rag_memory

#### Cobertura de los 8 modelos de prompt del SDD (5.8.7)

| ID SDD | Actividad | Estado en PoC |
|---|---|---|
| AI-01 | Store Visit Prep Brief | 🔴 No existe |
| AI-02 | 3-Point Review Prep | 🔴 No existe (módulo no existe) |
| AI-03 | DA Prep Brief | 🔴 No existe (la IA actúa durante creación, no en preparación) |
| AI-04 | Pattern Detection | ⚙️ Parcial — semantic matching existe pero no pattern detection cross-store/zone |
| AI-05 | Early Warning | 🔴 No existe |
| AI-06 | Follow-Up Automation | ⚙️ Parcial — reincidencia detection existe, no resumen de vencidos ni drafts de escalación |
| AI-07 | Performance Synthesis | ⚙️ Parcial — weekly AI summary existe pero sin output schema del SDD |
| AI-08 | Variable Checklist Items | 🔴 No existe (no hay checklist template digital) |

#### Lo rescatable

- `ai_client.py` es **el componente más reutilizable de todo el repo** ✅. La abstracción de Gemini con JSON output, error handling, embeddings y fallback es exactamente lo que el SDD necesita. No necesita rediseño — necesita que los prompts del SDD 5.8.7 se implementen sobre él.
- La lógica de `_gather_store_history` en `da_service.py` es un buen patrón de construcción de contexto para prompts.

#### Challenge C-03: RAG / Ask GRIP

> **Situación:** La PoC construyó un módulo RAG completo (pgvector + Gemini) con un componente de chat conversacional ("Ask GRIP"). El usuario puede hacer preguntas en lenguaje natural sobre los datos del sistema.
>
> **El SDD (5.8.4) sí incluye chat conversacional** — pero como interface secundaria sobre las actividades de IA ya definidas (AI-01 a AI-08), no como un RAG general sobre todos los datos.
>
> **Evaluación:** El RAG de la PoC va más lejos que el SDD en esta dimensión. Podría crear confusión sobre el alcance del AI layer.
>
> **Recomendación:** Conservar el RAG como capacidad subyacente. Pero en MVP, la IA se presenta como asistente contextual por módulo (AI-01 a AI-08), no como un chatbot general. El "Ask GRIP" puede ser una feature de Phase 2 cuando la base de datos tenga suficiente información acumulada para que el RAG sea útil.

---

### 6.0 Data Model General

**Categoría: ⚙️ Rediseñar parcialmente**

#### Entidades existentes en la PoC

| Modelo | Tabla | Reutilizable |
|---|---|---|
| `User` | `users` | ⚙️ Rediseñar → `Employee` + `Role` |
| `Zone` | `zones` | ⚙️ Conservar como agrupación geográfica, jerarquía va en `reports_to` |
| `Store` | `stores` | ✅ Reutilizable con adición de `tenant_id` |
| `Visit` | `visits` | ✅ Base válida, añadir `checklist_template_id` |
| `Finding` | `findings` | ✅ Reutilizable — añadir `source_type`, separar en `Finding` + `FollowUpItem` |
| `Evidence` | `evidences` | ✅ Reutilizable |
| `WeeklySession` | `weekly_sessions` | ⚙️ Rediseñar → `WeeklyReport` con cascade |
| `StrategicFeedback` | `strategic_feedback` | ⚙️ Evolucionar → 3 preguntas integradas al WeeklyReport |
| `DAIntervention` | `da_interventions` | ✅ Base válida — añadir scope, COO, auto-surface |
| `DAFindingsMap` | `da_findings_map` | ✅ Reutilizable |
| `FocalPoint` | `focal_points` | ⚠️ Concepto de PoC sin equivalente directo en SDD. Evaluar si sigue siendo relevante. |
| `IngestionJob` | `ingestion_jobs` | ⚠️ Solo relevante si se mantiene canal PDF como backup |

#### Entidades a construir (no existen)

| Entidad SDD | Descripción |
|---|---|
| `Role` | Entidad configurable con permission flags |
| `Tenant` | Soporte multi-tenancy (MVP = 1 tenant) |
| `ChecklistTemplate` | Plantilla con ítems fijos por tenant |
| `ChecklistTemplateItem` | Ítem fijo del template (sección, descripción, orden) |
| `ChecklistSession` | Visita digital (evolución de `Visit`) |
| `ChecklistSessionItem` | Resultado por ítem en una visita específica |
| `VariableItem` | Ítem variable agregado por JZ o AI para una visita |
| `FollowUpItem` | Entidad separada del Finding con lifecycle + auto-surface flags |
| `AspectLibrary` | Catálogo de 36 aspectos configurables |
| `ThreePointReview` | Sesión de evaluación mensual |
| `ThreePointReviewItem` | 3 aspectos evaluados en una sesión |

---

## Challenges Consolidados

| ID | Challenge | Impacto | Recomendación |
|---|---|---|---|
| **C-01** | PDF ingestion vs Checklist digital | Alto — define la UX del JZ | PDF como canal de respaldo. Checklist digital como canal primario |
| **C-02** | pgvector + Semantic Matching (reincidencias) | Medio — capacidad valiosa no en SDD | Conservar. Documentar en SDD como enriquecimiento del Follow-Up System |
| **C-03** | RAG / Ask GRIP | Medio — fuera de SDD scope MVP | Conservar la infraestructura. Scope limitado a Phase 2 |
| **C-04** | Firma doble en DA | Bajo — buena práctica de seguridad | Conservar concepto. Madurar implementación (MFA real pendiente) |
| **C-05** | HTTP async + DB sync (SQLAlchemy Session clásica) | Medio — limitación de throughput | Migrar a `AsyncSession` de SQLAlchemy para operaciones DB en coroutines |

---

## Backlog Priorizado

### 🔴 Bloqueantes MVP (sin esto el sistema no funciona)

| ID | Tarea | Módulo SDD | Referencia |
|---|---|---|---|
| B-01 | Diseñar e implementar entidad `Role` configurable (reemplaza enum hardcodeado) | 5.1 | `EMP-01`, `BR-EMP-01` |
| B-02 | Añadir `reports_to` a Employee para jerarquía flexible | 5.1 | `EMP-03` |
| B-03 | Añadir `tenant_id` a todas las tablas existentes + nueva tabla `Tenant` | 9.0 | Multi-tenancy |
| B-04 | Implementar autenticación real (JWT) con resolución de rol y scope | 5.1 | Todos los módulos |
| B-05 | Refactorizar DA para habilitar COO con scope OPERATIONS_ONLY | 5.5 | `DA-02`, `BR-DA-02` |
| B-06 | Crear `ChecklistTemplate` + `ChecklistTemplateItem` con 9 secciones | 5.3 | `CHK-01`, `CHK-02` |
| B-07 | Crear `FollowUpItem` como entidad separada con auto-surface flags | 5.6 | `FUP-01`, `BR-FUP-01` |

### ⚙️ Core MVP (funcionalidad central del sistema)

| ID | Tarea | Módulo SDD | Referencia |
|---|---|---|---|
| C-01 | Implementar checklist digital mobile-first (reemplaza form actual) | 5.3 | `CHK-03` a `CHK-07` |
| C-02 | Implementar ítems variables por visita (JZ + AI suggestions) | 5.3 | `CHK-08`, `AI-08` |
| C-03 | Rediseñar WeeklyReport con cascade 4 niveles | 5.2 | `WR-01` a `WR-05` |
| C-04 | Construir AspectLibrary + ThreePointReview completo | 5.4 | `TPR-01` a `TPR-10` |
| C-05 | Implementar auto-surface engine (job diario) | 5.6 | `FUP-05`, `BR-FUP-03` |
| C-06 | Implementar escalación automática por deadline vencido | 5.6 | `FUP-06` |
| C-07 | Construir dashboards restantes: Gte Ventas, Regional, COO, CEO, JT | 5.7 | `DASH-01` a `DASH-12` |
| C-08 | Implementar AI-01 (Store Visit Prep Brief) sobre ai_client.py existente | 5.8.7 | AI-01 |
| C-09 | Implementar AI-02 (3-Point Review Prep) | 5.8.7 | AI-02 |
| C-10 | Implementar AI-03 (DA Prep Brief) | 5.8.7 | AI-03 |
| C-11 | Implementar AI-05 (Early Warning job diario) | 5.8.7 | AI-05 |
| C-12 | Implementar AI-06 (Follow-Up Automation summary) | 5.8.7 | AI-06 |
| C-13 | UI de organigrama (frontend) — visualización del árbol jerárquico `reports_to` | 5.1 | B-02 (`/tree`, `/ancestors`) |

### ⚠️ Deuda técnica (existe pero debe corregirse)

| ID | Tarea | Origen |
|---|---|---|
| D-01 | Migrar SQLAlchemy Session clásica → AsyncSession (throughput) | C-05 arquitectura |
| D-02 | Separar entidad Finding (observación) de FollowUpItem (tarea) | Modelo de datos mixto |
| D-03 | Añadir output schema tipado a los prompts existentes (da_service, weekly_service) | 5.8.7 |
| D-04 | Definir algoritmo canónico de reincidencia DA (spec vs heurística actual) | backend_audit_report |
| D-05 | Política formal de alias chat/RAG (mantener o deprecar endpoint legacy) | backend_audit_report |
| D-06 | Revisar `FocalPoint` — evaluar si tiene equivalente en SDD o se elimina | Modelo sin mapeo |

### 🟡 Post-MVP

| ID | Tarea | Módulo SDD |
|---|---|---|
| P-01 | Ask GRIP conversacional como feature completa (RAG general) | 5.8.4 |
| P-02 | AI-04 Pattern Detection cross-store/zone (job semanal) | 5.8.7 |
| P-03 | AI-07 Performance Synthesis ejecutiva | 5.8.7 |
| P-04 | Integración KARDEX (adapter read-only) | 7.0 |
| P-05 | MFA real para firma ejecutiva de DA | 5.5 |
| P-06 | Estrategia de escalado vectorial pgvector (retención, índices HNSW) | C-02 |
| P-07 | Observabilidad: métricas, tracing, SLOs | NFR |

---

## Resumen Visual

```
MÓDULO                    POC → SDD        ESFUERZO
─────────────────────     ──────────────   ─────────────────────────
Employee / Role           ⚙️  Rediseñar    ████████░░  Alto
Autenticación             🔴  Construir    ████████░░  Alto
Multi-tenancy             🔴  Construir    ██████░░░░  Medio-Alto
Store Visit Checklist     ⚠️  Challenge    ██████░░░░  Medio-Alto
Weekly Reports            ⚙️  Rediseñar    █████░░░░░  Medio
Follow-Up System          ⚙️  Rediseñar    █████░░░░░  Medio
DA Module                 ⚙️  Rediseñar    ███░░░░░░░  Medio-Bajo
3-Point Review            🔴  Construir    ████████░░  Alto
36-Point Aspect Library   🔴  Construir    █████░░░░░  Medio
Dashboards (5 restantes)  🔴  Construir    ██████░░░░  Medio-Alto
AI Layer (6 prompts)      🔴  Construir    ████░░░░░░  Medio
─────────────────────────────────────────────────────────────────
ai_client.py              ✅  Conservar    Sin esfuerzo
Finding + Evidence        ✅  Conservar    Ajuste menor
DAIntervention (base)     ✅  Conservar    Ajuste menor
Store model               ✅  Conservar    Añadir tenant_id
Angular 21 + Tailwind     ✅  Conservar    Sin esfuerzo
```

---

*Gap Analysis generado en Fase B — 2026-03-29*
*Fuente de verdad: GRIP-SDD-Functional-Specs.md v1.0.0*
