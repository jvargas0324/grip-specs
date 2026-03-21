# Plan de Ejecución — Módulo 2: Follow-up & Regla R1 (Evidencia Obligatoria)

**Metodología:** SDD (Spec-Driven Development)  
**Alcance:** Dashboard con actualización de estado de hallazgos + validación R1 en frontend y backend

---

## PASO 1: Actualización de Specs (SDD)

### 1.1 `specs/technical-spec.md`

| Cambio | Detalle |
|--------|---------|
| **Versión** | De `7.4 (Python + Gemini + Angular Forms Edition)` a `7.5 (Follow-up & R1 Evidence Edition)` |
| **Sección 6 – Arquitectura Frontend** | Añadir: |
| | • **Componente:** `ResolveFindingModal` (o lógica integrada en el Dashboard) para actualizar estado de hallazgos |
| | • **Regla de UI (R1):** "El formulario se considera inválido si `status === 'verified'` y `evidence_link` está vacío; el botón de envío debe deshabilitarse en ese caso" |

### 1.2 `specs/functional-spec.md`

| Cambio | Detalle |
|--------|---------|
| **Módulo 2** | Especificar que el usuario **interactúa desde el Dashboard** con los hallazgos para cambiar su estado (botón "Resolver" por hallazgo) |
| **Regla de negocio visual** | Añadir explícitamente: *"El sistema no permitirá marcar un hallazgo como 'verified' sin adjuntar una evidencia (enlace o texto) justificativa."* |

---

## PASO 2: Desarrollo Backend (necesario para R1 end-to-end)

**Contexto:** El endpoint actual `PATCH /api/v1/findings/{id}/status` solo acepta `{ status }`. La R1 exige que para `status === 'verified'` exista evidencia en la tabla `evidences`. Hoy no existe endpoint para crear evidencias, por lo que cualquier intento de `verified` devuelve 400.

| Tarea | Detalle |
|-------|---------|
| **Extender payload del PATCH** | Modificar `FindingStatusUpdateModel` para aceptar opcionales: `evidence_url?: str`, `resolution_notes?: str` |
| **Lógica R1 en backend** | Si `status === 'verified'` y se envía `evidence_url`: crear registro en `evidences` con `finding_id`, `evidence_url` y `uploaded_by = current_user.id` antes de actualizar el status |
| **Archivo** | `grip-backend/app/api/findings.py` |

---

## PASO 3: Desarrollo Frontend (Angular)

### 3.1 `findings.service.ts`

| Tarea | Detalle |
|-------|---------|
| **Nuevo método** | `updateFindingStatus(id: string, payload: { status: string, resolution_notes?: string, evidence_link?: string })` |
| **Implementación** | `return this.api.patch(\`/findings/\${id}/status\`, payload)` |
| **Nota** | Si el backend acepta `evidence_url`, enviar el valor de `evidence_link` como `evidence_url` en el payload |

### 3.2 Dashboard — Columna "Acciones" y Botón "Resolver"

| Archivo | Cambio |
|---------|--------|
| `dashboard-jz.component.html` | Añadir columna `<th>Acciones</th>` en la tabla |
| | En cada fila: botón "Resolver" que invoca `openResolveModal(f)` |
| `dashboard-jz.component.ts` | Propiedades: `selectedFinding: Finding \| null`, `showResolveModal = false`, métodos `openResolveModal(finding)` y `closeResolveModal()` |

### 3.3 Modal de Resolución (Tailwind / HTML/CSS puro)

| Elemento | Detalle |
|----------|---------|
| **Ubicación** | Inline en `dashboard-jz.component.html` con overlay absoluto/fijo (Tailwind: `fixed inset-0 bg-black/50 z-50 flex items-center justify-center`) |
| **Contenido** | Mostrar `ai_summary` del hallazgo seleccionado |
| **Controles** | Select para `status` (opciones: `in_progress`, `verified`) e input texto para `evidence_link` |
| **Validación R1** | Si `status === 'verified'`, el input `evidence_link` se vuelve requerido; usar Reactive Forms con `Validators.required` condicional |
| **Botón enviar** | `[disabled]="resolveForm.invalid"` |
| **Envío** | Llamar a `updateFindingStatus()`, cerrar modal, mensaje de éxito (console.log o alert simple), recargar `getOpenFindings()` |

### 3.4 Dependencias

| Import | Uso |
|--------|-----|
| `ReactiveFormsModule`, `FormBuilder`, `FormGroup`, `Validators` | Formulario y validaciones del modal |
| `FormsModule` (opcional) | Si se usa `[(ngModel)]` en lugar de Reactive Forms |

---

## Orden de Ejecución

1. Actualizar `specs/technical-spec.md` (versión + Sección 6)
2. Actualizar `specs/functional-spec.md` (Módulo 2 + regla de evidencia)
3. Extender backend: `FindingStatusUpdateModel` + lógica de creación de evidencia en `findings.py`
4. Añadir `updateFindingStatus` en `findings.service.ts`
5. Modificar `dashboard-jz.component.ts`: estado del modal, formulario, handlers
6. Modificar `dashboard-jz.component.html`: columna Acciones, botón Resolver, markup del modal
7. Verificar flujo: Resolver → modal → validación R1 → envío → cierre → recarga de hallazgos

---

## Mapeo de Campos API

| Frontend | Backend |
|----------|---------|
| `evidence_link` | `evidence_url` |

Conviene mapear `evidence_link` → `evidence_url` en el payload enviado al backend para alinear con la tabla `evidences.evidence_url`.
