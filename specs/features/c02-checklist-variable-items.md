# C-02 — Variable Items per Visit

- **Estado:** borrador
- **Prioridad:** Core MVP
- **Depende de:** C-01 (Checklist digital mobile-first), C-05 (Auto-surface engine)
- **SDD referencia:** GRIP-SDD-Functional-Specs.md §5.3.5 (CL-05, CL-06, BR-CL-04), §5.8.7 AI-01 y AI-08
- **Ultima actualizacion:** 2026-03-29

---

## 1. Objetivo

Permitir que cada sesion de checklist incluya items variables ademas de los items fijos del template. Los items variables provienen de cuatro fuentes distintas (manual, follow-ups abiertos, sugerencias de IA, patrones entre pares) y son de alcance de sesion (session-scoped). AI-08 es la especializacion de AI-01 que sugiere items variables automaticamente.

## 2. Referencia a specs base

Toda la especificacion normativa vive en **GRIP-SDD-Functional-Specs.md**:

| Seccion | Contenido |
|---------|-----------|
| §5.3.5 | Variable items: definicion, fuentes, alcance de sesion |
| §5.8.7 AI-01 | Store Visit Prep Brief (output incluye suggested_variable_items) |
| §5.8.7 AI-08 | Variable Checklist Item Suggestions (especializacion de AI-01) |
| §5.6 FU-09 | Auto-surface de FollowUpItems en siguiente checklist |

## 3. Actores y permisos

| Actor | Acciones permitidas |
|-------|---------------------|
| JZ / JL | Agregar variable items manualmente al iniciar sesion; ver sugerencias de IA |
| JT | Ver variable items asignados a su sesion (no puede agregar) |
| GV / GL | Ver reportes con items variables evaluados |

## 4. Flujos principales

### 4.1 Inyeccion de variable items al crear sesion

1. Al crear una ChecklistSession (C-01 §4.1), el backend consulta las fuentes de variable items.
2. **Fuente OPEN_FOLLOWUP:** Se buscan FollowUpItems con `auto_surface_in_next_checklist = true` para la tienda (C-05).
3. **Fuente AI_SUGGESTION:** Si AI esta disponible, se invoca AI-08 para obtener `suggested_variable_items` (degradacion elegante si IA falla).
4. **Fuente PEER_PATTERN:** Se consultan patrones de incumplimiento en tiendas similares.
5. Los items de estas fuentes se insertan como VariableItem vinculados a la ChecklistSession.
6. Se presenta al JZ/JL la lista de variable items sugeridos para confirmacion.

### 4.2 Adicion manual por JZ/JL

1. Antes de iniciar la ejecucion, el JZ/JL puede agregar variable items manualmente (CL-05).
2. Cada item manual se registra con source = `MANUAL`.
3. Una vez iniciada la ejecucion, no se pueden agregar mas items (CL-06).

### 4.3 Respuesta a variable items

1. Los variable items aparecen en una seccion separada "Items Variables" dentro de la sesion.
2. Se responden igual que los items fijos: CUMPLE / NO_CUMPLE / NA.
3. Si responde NO_CUMPLE, aplica la misma logica de auto-generacion de follow-up que C-01.

### 4.4 Invocacion de AI-08

1. Backend construye el prompt con contexto de la tienda: follow-ups abiertos, scores recientes, KPIs.
2. Se invoca AI-08 (especializacion de AI-01 Store Visit Prep Brief).
3. Output schema: `suggested_variable_items` (lista de items con titulo, justificacion, prioridad).
4. Si la IA falla (timeout, quota, error), la sesion se crea sin sugerencias de IA — nunca se bloquea.

## 5. Reglas de negocio

| ID | Regla | Origen SDD |
|----|-------|------------|
| BR-CL-04 | Los variable items son session-scoped: existen solo dentro de la sesion que los contiene, no se propagan al template | §5.3 BR-CL-04 |

## 6. Contrato API

Base path: `/api/v1/`

| Metodo | Endpoint | Descripcion |
|--------|----------|-------------|
| GET | `/checklists/sessions/{session_id}/variable-items` | Listar variable items de la sesion |
| POST | `/checklists/sessions/{session_id}/variable-items` | Agregar variable item manual (solo JZ/JL, solo antes de iniciar) |
| DELETE | `/checklists/sessions/{session_id}/variable-items/{item_id}` | Eliminar variable item manual antes de iniciar |
| POST | `/checklists/sessions/{session_id}/variable-items/ai-suggest` | Solicitar sugerencias de AI-08 |

### Schemas principales

**VariableItemCreate (manual):**
```json
{
  "title": "string",
  "description": "string | null",
  "source": "MANUAL"
}
```

**VariableItemResponse:**
```json
{
  "id": "uuid",
  "session_id": "uuid",
  "title": "string",
  "description": "string | null",
  "source": "MANUAL | OPEN_FOLLOWUP | AI_SUGGESTION | PEER_PATTERN",
  "source_reference_id": "uuid | null",
  "ai_justification": "string | null"
}
```

**AI-08 Output Schema:**
```json
{
  "suggested_variable_items": [
    {
      "title": "string",
      "justification": "string",
      "priority": "HIGH | MEDIUM | LOW",
      "related_followup_id": "uuid | null"
    }
  ]
}
```

## 7. Cambios en UI (frontend)

| Pantalla | Descripcion |
|----------|-------------|
| Variable Items Panel | Seccion dentro de Checklist Execution que muestra items variables con badge de fuente (Manual, AI, Follow-Up, Patron) |
| AI Suggestions Drawer | Panel lateral con sugerencias de AI-08, boton para aceptar/rechazar cada una |
| Add Manual Item | Formulario inline para agregar variable item manual (visible solo para JZ/JL antes de iniciar) |

### Requisitos UX

- Los items variables se distinguen visualmente de los fijos (badge de color por fuente).
- Las sugerencias de IA muestran la justificacion en texto colapsable.
- Si IA no esta disponible, se muestra aviso sutil sin bloquear el flujo.

## 8. Criterios de aceptacion

- [ ] Al crear sesion, se inyectan variable items de fuentes OPEN_FOLLOWUP y PEER_PATTERN automaticamente.
- [ ] Un JZ/JL puede agregar variable items manuales antes de iniciar la ejecucion (CL-05).
- [ ] No se pueden agregar items una vez iniciada la ejecucion (CL-06).
- [ ] Los variable items son session-scoped: no modifican el template (BR-CL-04).
- [ ] AI-08 retorna sugerencias con titulo, justificacion y prioridad.
- [ ] Si AI-08 falla, la sesion se crea normalmente sin sugerencias (degradacion elegante).
- [ ] Cada fuente de variable item se identifica correctamente (MANUAL, OPEN_FOLLOWUP, AI_SUGGESTION, PEER_PATTERN).
- [ ] NO_CUMPLE en un variable item genera FollowUpItem igual que en items fijos (BR-CL-05).
- [ ] Endpoints protegidos con JWT (B-04) y tenant isolation (B-03).

## 9. Notas

- B-06 ya creo la tabla VariableItem (migracion 0016). C-02 agrega los endpoints y la logica de fuentes.
- La integracion con C-05 (Auto-Surface Engine) es bidireccional: C-05 marca follow-ups para resurgir en checklists, y C-02 los consume como fuente OPEN_FOLLOWUP.
- AI-08 es una especializacion del prompt AI-01 (Store Visit Prep Brief) descrito en §5.8.7. El output de AI-01 incluye `suggested_variable_items` que AI-08 extiende.
- La llamada a AI-08 debe estar en `try/except` capturando errores de Gemini (ver GRIP-SDD-Functional-Specs.md §5.8).
