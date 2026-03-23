# 🛠 Technical Specification Master: Proyecto GRIP (Fase 1)
**Versión:** 7.20 (M1 canal PDF async: `error_code` + mensajes de fallo seguros en `GET /ingest/pdf/{job_id}`; columna `ingestion_jobs.error_code`)
**Alcance:** Persistencia de datos, Contratos de API, Inteligencia de Extracción y Gobierno.
**Arquitectura Core:** PostgreSQL (Relacional) + pgvector (Vectorial) + Python/FastAPI Backend + Front Angular
**Alineación:** Revisión documental 2026-03-23 (v7.20: contrato PDF job — `error_code` estable y `error_detail` solo texto UI; logs conservan detalle técnico. v7.19: pestañas checklist manual vs PDF. v7.18: email JZ sesión + combobox tienda. Trazabilidad: [backend_audit_report.md](backend_audit_report.md).

---

## 1. Esquema de Base de Datos (Relational & Vectorial Schema)

```sql
-- EXTENSIONES NECESARIAS
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "vector"; -- Soporte para Embeddings de 768 dimensiones (Google Gemini)

-- ==========================================
-- ESTRUCTURA BASE Y RBAC (Roles)
-- ==========================================
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    email VARCHAR(255) UNIQUE NOT NULL,
    role ENUM('jz', 'gv', 'regional', 'coo', 'ceo') NOT NULL,
    zone_id UUID REFERENCES zones(id), -- Usuario asignado a una zona (1 usuario -> 1 zona)
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE zones (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name VARCHAR(255) NOT NULL
);

-- ==========================================
-- MÓDULOS 1 & 2: INGESTIÓN Y FOLLOW-UP (v1.0)
-- ==========================================

-- Tabla de Tiendas
CREATE TABLE stores (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    code VARCHAR(50) UNIQUE NOT NULL, -- Ej: 'CACIQUE T002'
    name VARCHAR(255),
    zone_id UUID REFERENCES zones(id) -- Referencia a la zona de la tienda
);

-- Tabla de Visitas (Header del PDF)
CREATE TABLE visits (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    store_id UUID REFERENCES stores(id),
    jz_id UUID REFERENCES users(id), -- Jefe de Zona
    visit_date TIMESTAMP NOT NULL,
    raw_file_url TEXT, -- Link al PDF original
    created_at TIMESTAMP DEFAULT NOW()
);

-- Jobs de extracción para canal PDF asíncrono (M1)
CREATE TABLE ingestion_jobs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    jz_id UUID REFERENCES users(id),
    status ENUM('queued', 'processing', 'preview_ready', 'confirmed', 'failed') NOT NULL,
    source_file_url TEXT NOT NULL, -- object storage (retención 1 año)
    source_file_size_bytes INTEGER NOT NULL,
    source_file_pages INTEGER NOT NULL,
    extraction_result JSONB, -- header + raw_content + ai_output (preview)
    extraction_warnings JSONB, -- advertencias de extracción parcial
    missing_fields JSONB, -- campos faltantes detectados
    confidence_score NUMERIC(5,4), -- trazabilidad de calidad de extracción
    error_code VARCHAR(32), -- ai_quota | ai_unavailable | processing_failed (null si OK)
    error_detail TEXT, -- mensaje seguro para UI cuando status = failed (no trazas proveedor)
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- Tabla de Hallazgos (Cuerpo del PDF)
CREATE TABLE findings (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    visit_id UUID REFERENCES visits(id),
    store_code VARCHAR(50), -- Denormalizado para filtro directo por tienda en dashboard
    category VARCHAR(100), -- Ej: 'Limpieza', 'Precios'
    control_point TEXT, -- Texto del ítem del checklist
    compliance BOOLEAN NOT NULL, -- Cumple / No Cumple
    observation TEXT, -- Comentario del JZ
    severity_rank ENUM('low', 'medium', 'high'), -- Asignado por IA
    parent_finding_id UUID REFERENCES findings(id), -- Para Reincidencias
    status ENUM('open', 'in_progress', 'verified', 'reopened') DEFAULT 'open',
    embedding VECTOR(768), -- Vector para Gemini (gemini-embedding-001)
    ai_summary TEXT,
    suggested_action TEXT,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Tabla de Evidencias
CREATE TABLE evidences (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    finding_id UUID REFERENCES findings(id),
    evidence_url TEXT NOT NULL,
    uploaded_by UUID REFERENCES users(id), 
    created_at TIMESTAMP DEFAULT NOW()
);

-- ==========================================
-- MÓDULOS 3 & 4: WEEKLY HUB Y MEMORIA RAG (v1.1)
-- ==========================================

-- Tabla de Sesiones Weekly
CREATE TABLE weekly_sessions (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    organizer_id UUID REFERENCES users(id), -- Gerente de Ventas / Regional
    scope_type ENUM('zonal', 'regional', 'consolidated'),
    start_date TIMESTAMP NOT NULL,
    end_date TIMESTAMP NOT NULL,
    status ENUM('scheduled', 'in_progress', 'completed') DEFAULT 'scheduled',
    snapshot_data JSONB, -- Copia de los hallazgos críticos al momento de la sesión
    created_at TIMESTAMP DEFAULT NOW()
);

-- Tabla de Feedback Estratégico (Las 3 Preguntas)
CREATE TABLE strategic_feedback (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    session_id UUID REFERENCES weekly_sessions(id),
    author_id UUID REFERENCES users(id), -- COO o CEO
    target_user_id UUID REFERENCES users(id), -- A quién va dirigida la pregunta
    question_1 TEXT, -- ¿Qué pasó?
    question_2 TEXT, -- ¿Por qué pasó?
    question_3 TEXT, -- ¿Qué cambiamos?
    response_text TEXT,
    embedding VECTOR(768), -- Vector para Gemini (gemini-embedding-001)
    created_at TIMESTAMP DEFAULT NOW()
);

-- Tabla de Focal Points (Instrucciones para JZ)
CREATE TABLE focal_points (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    store_id UUID REFERENCES stores(id),
    author_id UUID REFERENCES users(id),
    instruction_text TEXT NOT NULL,
    is_active BOOLEAN DEFAULT TRUE,
    expires_at TIMESTAMP
);

-- ==========================================
-- MÓDULO 5: DA - DIENSTAUFSICHT (v1.2)
-- ==========================================

-- Tabla de Documentos DA
CREATE TABLE da_interventions (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    case_number SERIAL UNIQUE, -- Ej: DA-2026-001
    creator_id UUID REFERENCES users(id), -- Siempre el CEO
    target_regional_id UUID REFERENCES users(id), -- Gerente Regional bajo supervisión
    store_id UUID REFERENCES stores(id),
    status ENUM('active', 'pending_validation', 'closed_resolved', 'closed_unresolved') DEFAULT 'active',
    
    -- Campos de Análisis Estructural
    proven_facts TEXT NOT NULL, -- Evidencia consolidada por IA
    root_cause_analysis TEXT, -- Análisis de las "3 Preguntas" + IA
    structural_action_plan TEXT, -- El cambio profundo propuesto
    
    -- Control de Tiempos
    due_date DATE NOT NULL,
    closed_at TIMESTAMP,
    
    -- Firmas Digitales (Hashes de Validación)
    regional_signature_hash TEXT,
    ceo_signature_hash TEXT,
    
    created_at TIMESTAMP DEFAULT NOW()
);

-- Relación DA <-> Findings (Para saber qué detonó el DA)
CREATE TABLE da_findings_map (
    da_id UUID REFERENCES da_interventions(id),
    finding_id UUID REFERENCES findings(id),
    PRIMARY KEY (da_id, finding_id)
);

```

---

## 2. API & Service Map (Endpoints Contract)

**SSOT complementario (Pydantic y rutas):** [api-contract-ssot.md](api-contract-ssot.md).

### Módulo 1: Ingestión

**POST** `/api/v1/ingest`

* **Payload (JSON):**

```json
{
  "header": {
    "store_code": "STRING",
    "jz_email": "STRING",
    "visit_date": "ISO8601_TIMESTAMP"
  },
  "raw_content": [
    { "category": "STRING", "item": "STRING", "status": "BOOLEAN", "comment": "STRING" }
  ]
}

```

**POST** `/api/v1/ingest/pdf`

* **Purpose:** Canal adicional asíncrono para PDF digital (sin reemplazar `POST /api/v1/ingest`).
* **Request (`multipart/form-data`):**
  * `file`: PDF (máx 1MB, máx 2 páginas, texto digital).
* **Nota V1:** no se aceptan campos multipart adicionales (p. ej. `store_code` de validación cruzada); queda explícitamente **fuera de alcance** hasta alinear contrato e implementación (ver spec de feature M1-PDF §6.1).
* **Response (202):**
```json
{
  "job_id": "UUID",
  "status": "queued",
  "limits": { "max_size_mb": 1, "max_pages": 2 }
}
```

**GET** `/api/v1/ingest/pdf/{job_id}`

* **Purpose:** Consultar estado y preview de extracción.
* **Campos de error (siempre presentes en el JSON; valores nulos si no aplica):**
  * `error_code`: `ai_quota` | `ai_unavailable` | `processing_failed` | `null`.
  * `error_detail`: texto en español apto para mostrar al usuario si `status === "failed"`; **no** exponer mensajes crudos del proveedor IA. Detalle técnico solo en logs del servidor.
* **Response (200):**
```json
{
  "job_id": "UUID",
  "status": "queued|processing|preview_ready|confirmed|failed",
  "error_code": null,
  "error_detail": null,
  "result": {
    "header": { "store_code": "STRING", "jz_email": "STRING", "visit_date": "ISO8601_TIMESTAMP" },
    "raw_content": [
      { "category": "STRING", "item": "STRING", "status": "BOOLEAN", "comment": "STRING" }
    ],
    "ai_output": [
      {
        "summary": "STRING",
        "severity_rank": "low|medium|high",
        "suggested_action": "STRING",
        "confidence": "FLOAT"
      }
    ],
    "warnings": ["STRING"],
    "missing_fields": ["STRING"]
  }
}
```

* **Nota:** Con `status: "failed"`, `result` es `null` y `error_code` / `error_detail` describen el fallo de forma segura.

**POST** `/api/v1/ingest/pdf/{job_id}/confirm`

* **Purpose:** Confirmar preview y persistir `visits` + `findings`.
* **Payload (JSON):**
```json
{ "confirm": true }
```
* **Response (200):** `{ "status": "confirmed", "visit_id": "UUID", "findings_created": 0, "warnings": ["STRING"] }`
* **Rule:** Si la extracción es incompleta, el sistema debe **guardar parcial + advertencias**.

### Módulo 2: Findings (Frontend API)

**GET** `/api/v1/findings`

* **Query Params (opcionales):**
  * `page`: `INT` (default `1`, min `1`).
  * `page_size`: `INT` (default `20`, rango `1..100`).
  * `store_code`: `STRING` (filtro principal por tienda, coherente con dashboard zonal).
  * `store_id`: `UUID` (filtra por tienda; se resuelve vía `visits.store_id`).
  * `status`: `open | in_progress | verified | reopened`.
  * `category`: `STRING`.
  * `search`: `STRING` (búsqueda textual básica sobre categoría, punto de control, resumen IA, acción sugerida, severidad, observación y estado).
  * `date_start`, `date_end`: `DATE` (filtro por `findings.created_at`).
* **Response (JSON, paginada):**
```json
{
  "items": [
    {
      "id": "UUID",
      "visit_id": "UUID",
      "store_code": "STRING|null",
      "category": "STRING",
      "control_point": "STRING",
      "compliance": false,
      "observation": "STRING",
      "ai_summary": "STRING",
      "suggested_action": "STRING",
      "severity_rank": "low|medium|high|null",
      "parent_finding_id": "UUID|null",
      "status": "open|in_progress|verified|reopened",
      "created_at": "ISO8601_TIMESTAMP"
    }
  ],
  "total": 0,
  "page": 1,
  "page_size": 20,
  "total_pages": 1
}
```
* **Orden:** `created_at desc`.
* **Scope zonal:** para roles no-CEO, el backend filtra por `current_user.zone_id` **solo si** `zone_id` está definido (en desarrollo mock: header `X-User-Zone-Id`). Si falta, **no** se aplica filtro zonal (comportamiento heredado para pruebas; en producto los usuarios territoriales deben tener zona asignada).

**GET** `/api/v1/stores/options`

* **Purpose:** Endpoint dedicado para poblar el dropdown de tiendas sin depender del response de findings.
* **Query Params (opcionales):**
  * `search`: `STRING` (filtro por `stores.code` o `stores.name`).
* **Response (JSON):**
```json
{
  "items": [
    { "store_code": "STRING", "store_name": "STRING|null" }
  ]
}
```
* **Scope zonal:** para roles no-CEO con `zone_id` definido, retorna solo tiendas de esa zona; para CEO retorna todas. Sin `zone_id` en rol no-CEO, no se aplica filtro zonal (igual que `GET /findings`).

**PATCH** `/api/v1/findings/{id}/status`

* **Payload (JSON):** `{ "status": "open" | "in_progress" | "verified" | "reopened", "evidence_url"?: "STRING", "resolution_notes"?: "STRING" }`
* **Lógica:** Si el payload incluye `evidence_url` con contenido, se debe crear un registro en la tabla `evidences` sin importar si el estado es 'verified' o 'in_progress'. La evidencia es obligatoria para `status === 'verified'` (R1).

### Módulo 3: Weekly Hub

**GET** `/api/v1/weekly/summary`

* **Query Params:**
  * `store_id`: `UUID` (opcional). Filtra hallazgos a una tienda específica. Ideal para el Gerente Regional.
  * `store_code`: `STRING` (opcional). Alternativa a `store_id`; se resuelve al store correspondiente (ej: `CACIQUE T002`).
  * `region_id`: `UUID` (opcional). Agrega todas las tiendas de la zona. Para vista consolidada.
  * `week_number`: `INT` (opcional).
  * `start_date`, `end_date`: `DATE` (opcional). Filtro por rango de fechas de visitas.
* **Logic:** Llama a `get_weekly_summary`. Agrega el cumplimiento y devuelve el TOP 5 de hallazgos con `severity_rank = 'high'`. Gemini genera `focal_points` y `executive_brief` (Resumen Ejecutivo).

**POST** `/api/v1/weekly/feedback`

* **Payload:** `{ "session_id": "UUID", "questions": ["...", "...", "..."], "target_id": "UUID" }`
* **Logic:** Registra el esquema de 3 preguntas del COO/CEO. Dispara una notificación push al Gerente Regional.

### Módulo 4: RAG Query / Ask GRIP (Chat)

* **Contrato canónico (producto / Angular):** **`POST /api/v1/chat`**. Es el path que usa el SPA (`ChatService` → base URL `/api/v1` + `/chat`). En código actual `get_current_user` usa **headers mock** (`X-User-Id`, `X-User-Role`, …); la referencia a JWT describe el **objetivo** de hardening cuando exista auth real (ver audit §B WARNING ask-grip vs chat).

**POST** `/api/v1/chat`

* **Payload (JSON):**
```json
{
  "message": "STRING",
  "store_code": "STRING (opcional)"
}
```
* **Logic:** Convierte `message` en query RAG. Si `store_code` está presente, filtra `findings` por tienda (`visits.store_id` → `stores.code`). Ejecuta búsqueda vectorial sobre `findings.embedding` y `strategic_feedback.embedding`. Gemini genera la respuesta contextual.
* **Response:** `{ "answer": "STRING", "evidence": [{ "source_type", "source_id", "score", "excerpt" }] }`

**POST** `/api/v1/query/ask-grip` *(alias / integración)*

* **Payload:** `{ "query": "STRING", "store_code": "STRING (opcional)" }` (schema compartido: `AskGripRequest` en `app/schemas/rag.py`; `query` equivale a `message` del endpoint canónico).
* **Logic:** Misma implementación de negocio que `/api/v1/chat` (`rag_service.ask_grip`). Expuesto en `grip-backend/app/api/rag.py`.
* **Auth:** Hoy el router **no** aplica `get_current_user` en la firma del handler (diferencia respecto a `/api/v1/chat`). Tratar como **deuda de seguridad / alineación** hasta que se unifique; no eliminar el alias sin avisar a integraciones que lo consuman.

### Módulo 5: DA (Acto de Gobierno)

**POST** `/api/v1/da/create`

* **Restricción:** `role == 'ceo'` (valor canónico en RBAC; no usar alias `SUPER_ADMIN` en contratos).
* **Logic (regla objetivo):** El expediente debe basarse en hallazgos críticos de la tienda; la profundidad de reincidencia se modela con la cadena `parent_finding_id` (ancestros hasta `NULL`). *Umbral orientativo de negocio:* cadena de longitud ≥ 3 (tres o más eslabones de no-cumplimiento vinculados). No existe columna `reincidencia_count` en el esquema §1.
* **Implementación vigente (referencia código):** `grip-backend/app/services/da_service.py` — `_gather_store_history` selecciona findings de los últimos 90 días con `severity_rank = 'high'` **o** `parent_finding_id IS NOT NULL`, y enlaza hasta 50 IDs en `da_findings_map`. Cuando se implemente el conteo explícito de cadena ≥ 3, actualizar este párrafo y el servicio.
* **Payload:** `{ "target_regional_id": "UUID", "store_id": "UUID", "due_date": "DATE" }`.

**PATCH** `/api/v1/da/close`

* **Logic:** Proceso de cierre en dos pasos.
1. El Regional carga el `structural_action_plan` y firma (`regional_signature_hash`).
2. El CEO valida el resultado y ejecuta la firma final (`ceo_signature_hash`).


* **Constraint:** El estado no cambia a `closed_resolved` sin ambos hashes de firma (`regional_signature_hash` y `ceo_signature_hash`). Esta exigencia es parte del contrato del endpoint, no del ID **R6** (R6 = indexación RAG post-cierre, ver §4).

---

## 3. Orquestación de IA y Prompts (Gemini AI Blueprint)

### A. Extractor y Severidad (Módulo 1)

* **Model:** `gemini-2.5-flash-lite`
* **System Prompt:** "Eres un auditor experto en retail. Recibirás un hallazgo operativo. Tu tarea es: 1. Resumir la observación en máximo 10 palabras. 2. Asignar una severidad (Low/Medium/High) basándote en el riesgo de pérdida de dinero, seguridad física o daño de imagen. Devuelve solo JSON."
* **Canal PDF async (v1):** para `POST /api/v1/ingest/pdf`, la IA extrae cabecera (`store_code`, `jz_email`, `visit_date`) y checklist (`category`, `item`, `status`, `comment`) y genera el mismo output esperado de M1 (`summary`, `severity_rank`, `suggested_action`) antes de fase de confirmación.
* **Trazabilidad de extracción:** registrar `confidence_score`, `warnings` y `missing_fields` por job para auditoría de calidad (objetivo de feature: 95% en dataset objetivo).

### B. The Semantic Matcher (Módulo 2)

* **Model:** `gemini-embedding-001` (768 dimensiones)
* **Logic:** 1. Busca en `findings` por `store_id` y `control_point` con fecha anterior a la actual.
2. Compara el `embedding`. Si la similitud semántica es alta (distancia de coseno < 0.15) y ambos son `compliance: FALSE`.
3. Setea `parent_finding_id` del nuevo hallazgo con el ID del hallazgo anterior.
4. Envía notificación de "Reincidencia Crítica" si el conteo de la cadena es $\ge 2$.

### C. The Executive Summarizer (Módulo 3)

* **Model:** `gemini-2.5-flash-lite`
* **Logic:** Toma los `findings` de severidad `high`.
* **System Prompt:** "Resume para un ejecutivo ocupado. Ve al grano, resalta el riesgo de negocio. Detecta si el reporte del Gerente contradice la data cruda de reincidencias."

### D. Flujo de Ingestión en Memoria - RAG (Módulo 4)

* **Trigger:** Cuando `weekly_sessions.status` cambia a `completed`.
* **Proceso:** Toma `findings` y `strategic_feedback` de la sesión -> Genera resumen semántico (`gemini-2.5-flash-lite`) -> Convierte en vector (`gemini-embedding-001`) -> Guarda con metadatos `{store_id, region_id, date}`.
* **Ask GRIP Prompt:** "Eres la memoria operativa de GRIP. Responde preguntas estratégicas basándote únicamente en el contexto recuperado. Si detectas un patrón de falla en varias tiendas, menciónalo como un riesgo sistémico."

### E. DA - Generación y Causa Raíz (Módulo 5)

* **Model:** `gemini-2.5-flash-lite` (Aprovechando su ventana de contexto extendida).
* **Generación Expediente:** "Eres un Auditor Forense. Analiza el historial de visitas, respuestas al COO y fotos de los últimos 3 meses. Redacta los 'Hechos Probados' del DA."
* **Causa Raíz:** "Basado en la persistencia del fallo, ¿es este un problema de recursos, de capacitación o de negligencia de supervisión? Aplica los 5 Whys."

### F. Resiliencia y degradación (implementación)

Normativa alineada con el cliente IA en `grip-backend/app/core/ai_client.py` y con las reglas de proyecto en `.cursorrules` (integración Gemini).

* **Excepciones tipadas:** Usar `GeminiAPIError` (base), `GeminiQuotaError` (cuota / rate limit) y `GeminiTimeoutError` (timeout) para fallos de la API de Google; el cliente las eleva tras mapear excepciones de `google.api_core`.
* **Envolver llamadas:** Toda invocación a `generate_structured_text`, `generate_plain_text`, `generate_embedding` (o equivalentes async del cliente) en servicios debe ir en `try/except` capturando al menos `GeminiAPIError` y `GeminiQuotaError` donde aplique, o delegar en un manejador de ruta que las traduzca a HTTP (ej. ingestión: `app/api/ingestion.py`).
* **No bloquear transacciones críticas:** Un fallo de IA no debe impedir persistir el resultado principal cuando el negocio lo exija. **Referencia:** cierre de DA (`da_service.close_da`): si el embedding R6 falla, se registra warning y el `commit` del cierre sigue ejecutándose.
* **M1 PDF (degradación específica):** en extracción incompleta del PDF, persistir datos parciales tras confirmación del usuario y devolver advertencias explícitas (`warnings`) en la respuesta.
* **M1 PDF (jobs async — fallos IA):** ante error en el worker de extracción, persistir `status=failed`, `error_code` estable y `error_detail` con mensaje legible para UI; registrar la excepción completa solo en logs. El `GET /api/v1/ingest/pdf/{job_id}` expone al cliente solo esos valores saneados (no propagar `str(exc)` del proveedor IA como `error_detail`).
* **Asincronía:** Las funciones que llaman al SDK de Gemini deben ser `async` y usarse desde rutas/servicios async para no bloquear el event loop de FastAPI.
* **Trazabilidad en código:** `ingestion_service`, `weekly_service`, `rag_service`, `da_service`; verificación documental en [backend_audit_report.md](backend_audit_report.md) (sección resiliencia IA).

---

## 4. Reglas de Validación y Negocio (Hard Constraints)

* **R1 (Bloqueo de Cierre):** No se permite cambiar `finding.status` a 'verified' si la tabla `evidences` no contiene al menos 1 registro vinculado a ese `finding_id`.
* **R2 (Persistencia):** Un hallazgo marcado como 'reopened' por el Gerente de Ventas hereda automáticamente la severidad 'high' independientemente del análisis inicial de la IA.
* **R3 (Visibilidad Directa / Anti-Sesgo):** Usuario con rol **`ceo`** (único valor RBAC; no usar `SUPER_ADMIN` en API ni en código). En **`GET /api/v1/weekly/summary`**, el CEO **no** debe quedar restringido por filtros de región/zona que sí aplican a otros roles (datos consolidados / drill-down). *Evolución:* aplicar la misma política explícitamente en **`GET /api/v1/findings`** si el producto exige listado global sin filtro territorial para el CEO.
* **R4 (Bloqueo por DA activo):** Si una tienda tiene un `da_interventions.status == 'active'`, el sistema resalta los check-lists de esa tienda con un "Flag de Intervención" y **bloquea** en API las transiciones de hallazgo a `verified` o `in_progress` que eludan el gobierno del DA (`PATCH /api/v1/findings/{id}/status`).
* **R5 (Escalamiento de Firma):** El cierre de un DA debe exigir MFA (Multi-Factor Authentication) para generar o validar el `ceo_signature_hash` cuando la integración MFA esté completa (hoy puede validarse solo presencia del hash; ver audit).
* **R6 (Memoria Post-DA):** Al cerrar un DA (`closed_resolved`), el texto de resolución / plan estructural se indexa para el Módulo 4 (RAG), p. ej. persistiendo embedding asociado al contenido (puede implementarse vía `strategic_feedback` u otra tabla vectorial; ver `da_service.close_da`).

**Trazabilidad:** Los comentarios en `grip-backend` deben citar el mismo **R1–R6** que esta sección. Resumen código ↔ R*: [backend_audit_report.md](backend_audit_report.md) §C y §Correspondencia.

---

## 5. Definición de Interfaz (Technical UI State)

* **State: Pre-Visita (Jefe de Zona):** * Carga: `focal_points.filter(store_id=current, active=true)`
* Carga: `findings.filter(store_id=current, status='open')`


* **State: Weekly Review (COO):**
* Carga: `weekly_sessions.map(region_compliance_chart)`
* Formulario: `strategic_feedback` con validación de 3 campos obligatorios.



---

## 6. Arquitectura Frontend (Angular)

* **Sistema visual (tema corporativo):** Tokens de color (variables CSS + clases Tailwind `grip-*`), tipografía Inter / JetBrains Mono, y tema **claro/oscuro** (`class` `dark` en `html`) con persistencia en cliente. Detalle y criterios de aceptación: [specs/features/ui-board-refresh-v1.md](features/ui-board-refresh-v1.md). Sin cambio de contrato API.

* **Rutas principales (Fase 1):**
  * `/dashboard`: Dashboard del Jefe de Zona (Pre-Visita), muestra hallazgos abiertos y acciones principales.
  * `/checklist`: Formulario de Ingesta para que el Jefe de Zona registre una nueva visita y sus hallazgos.
  * `/weekly`: Resumen Semanal (Gerente Regional, COO, CEO). Generador de resumen ejecutivo por IA basado en hallazgos de tienda.
  * `/ask-grip`: Ask GRIP (Chat). Asistente conversacional RAG para consultas en lenguaje natural sobre hallazgos y rendimiento de tiendas.

* **Componentes:**
  * `DashboardJzComponent` (`features/ingestion`): Lista hallazgos vía `GET /api/v1/findings` con **filtros y paginación server-side** (búsqueda, estado, categoría, fechas, `store_code`). Opciones del filtro de tienda se cargan con `GET /api/v1/stores/options` (desacoplado). La tabla incluye al menos Fecha, Estado (badges semánticos), **código de tienda** (`store_code`), categoría, punto de control, severidad, resumen IA y acciones. Vista de Pre-Visita y acceso al formulario.
  * `ResolveFindingModal` (o lógica integrada en el Dashboard): permite al usuario actualizar el estado de los hallazgos (verified, in_progress) con evidencia obligatoria para verified (Regla R1).
  * `ChecklistFormComponent` (`features/ingestion`): Formulario basado en **Reactive Forms** con **cabecera de visita compartida** (tienda, email sesión, fecha) y **dos pestañas** que separan el canal JSON del canal PDF: (1) **Checklist manual** — hallazgo ítem a ítem + envío `POST /api/v1/ingest` (Cancelar / Enviar); (2) **PDF async** — upload, estado, preview y confirmación (sin usar el submit del formulario JSON). Campos de cabecera:
    * `store_code` (combobox: lista de tiendas de la zona del usuario vía `StoresService.getStoreOptions()`; búsqueda en panel con **filtrado a partir del 3.er carácter** sobre código o nombre, manteniendo la lista completa si el término tiene menos de 3 caracteres).
    * `jz_email`: **no editable**; se envía en el payload de `POST /api/v1/ingest` con el **email de la sesión** (`AuthService.getCurrentUser().email`), no como campo de entrada manual.
    * `visit_date`
    * En la pestaña manual: `category`, `item`, `status` (Cumple / No Cumple), `comment` (comentarios del JZ)

* **Servicios:**
  * `FindingsService` (`features/ingestion/services`):
    * `getFindings(params)`: `GET /api/v1/findings` paginado (params: `page`, `page_size`, `store_code`, `status`, `category`, `search`, `date_start`, `date_end`, etc.).
    * `getAllFindings()` / `getOpenFindings()`: atajos legacy sobre `getFindings` con `page_size` amplio.
    * `ingestChecklist(payload)`: `POST /api/v1/ingest` (Módulo 1).
    * `updateFindingStatus(id, payload)`: `PATCH /api/v1/findings/{id}/status`.
  * `StoresService` (`features/ingestion/services/stores.service.ts`): `getStoreOptions(search?)` → `GET /api/v1/stores/options` (query opcional `search`) para el filtro de tiendas del **dashboard** y para poblar el **selector de tienda del formulario de ingesta** (carga inicial sin `search` = catálogo de la zona; refinamiento en cliente con umbral de 3 caracteres según §6).

* **Regla de UI (R1):** El formulario de resolución debe invalidarse si `status === 'verified'` y `evidence_link` está vacío; el botón de envío debe permanecer deshabilitado en ese caso.
* **Lógica del Modal de Resolución:** Tras una actualización exitosa, el modal debe cerrarse, el formulario debe resetearse y la tabla debe recargar los datos automáticamente.

* **Módulo 3 - Weekly Summary:**
  * `WeeklySummaryComponent` (`features/weekly/weekly-summary/`): Pantalla para Gerente Regional, COO, CEO. Input de `store_code` (o `store_id`), botón "Generar Resumen de IA". Estado de carga (spinner) con texto "La IA está analizando los hallazgos...". Secciones: Resumen Ejecutivo (párrafo) y Puntos Focales (grid de tarjetas).
  * `WeeklyService` (`features/weekly/services/weekly.service.ts`): `getWeeklySummary(params)`. Llama a `GET /api/v1/weekly/summary` con `store_id` o `store_code` y opcionalmente `start_date`, `end_date`.

* **Módulo 4 - Ask GRIP (Chat):**
  * `AskGripComponent` (`features/chat/ask-grip/`): Interfaz conversacional estilo ChatGPT. Selector opcional de `store_code`. Contenedor scrollable con historial de mensajes (burbujas: usuario con acento corporativo / superficie elevada; IA con superficie muted). Input inferior + botón enviar. Estado de carga "La IA está pensando...".
  * `ChatService` (`features/chat/services/chat.service.ts`): `sendMessage(payload: { message, store_code? })`. Llama a `POST /api/v1/chat` con el payload y devuelve `{ answer, evidence }`.


---

## 7. Stack Tecnológico Estricto (Ecosistema Python)

* **Framework Backend:** FastAPI (Python 3.10+).
* **ORM & Migraciones:** SQLAlchemy 2.0 + Alembic + `pgvector.sqlalchemy`.
* **Validación:** Pydantic (V2).
* **Base de Datos:** PostgreSQL 16+ con extensión `pgvector` (vectores `VECTOR(768)`).
* **Proveedor IA:** Google Generative AI SDK (`google-generativeai`). **Generación de texto:** `gemini-2.5-flash-lite`. **Embeddings:** `gemini-embedding-001` (768 dimensiones).
* **Seguridad:** FastAPI Dependencies (RBAC + JWT).
