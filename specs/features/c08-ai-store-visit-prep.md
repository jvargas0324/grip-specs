# C-08 — AI-01 Store Visit Prep Brief

- **Estado:** borrador
- **Prioridad:** Core MVP
- **Depende de:** C-01 (Checklist Digital), C-02 (Variable Items), C-05 (Auto-Surface Engine)
- **SDD referencia:** §5.8.7 AI-01 (Store Visit Prep), §5.8.3.1 AI-PREP-01, BR-AI-01 a BR-AI-05
- **Usa IA/Gemini:** Si
- **Ultima actualizacion:** 2026-03-29

---

## 1. Objetivo

- **Aplicacion(es):** grip-backend, grip-frontend
- **Problema:** El JZ llega a la tienda sin contexto: no sabe que follow-ups estan pendientes, que patrones se repiten ni que areas priorizar. El Store Visit Prep Brief usa IA para generar un resumen contextualizado antes de la visita, transformando visitas reactivas en estrategicas.
- **Alcance:**
  - **grip-backend:** Endpoint que recopila contexto (ultimos 5 checklists, follow-ups abiertos, KPIs, items variables previos, patrones de tiendas par) y lo envia a Gemini para generar un brief estructurado. Cache de 30 min. Fallback deterministico si Gemini falla.
  - **grip-frontend:** Pantalla "Preparar Visita" accesible antes de iniciar un checklist, mostrando el brief con areas prioritarias, alertas y sugerencias de items variables.
  - **Fuera de alcance:** Integracion real con KARDEX (P-04, post-MVP); el campo `kardex_kpis_last_30d` estara vacio hasta entonces.

## 2. Referencia a specs base

- Modelo de prompt y output: `GRIP-SDD-Functional-Specs.md` §5.8.7 AI-01
- Preparacion asistida: `GRIP-SDD-Functional-Specs.md` §5.8.3.1 AI-PREP-01
- Reglas de resiliencia IA: `GRIP-SDD-Functional-Specs.md` §5.8 (BR-AI-01 a BR-AI-05)
- Checklist context: `GRIP-SDD-Functional-Specs.md` §5.3
- Follow-Up context: `GRIP-SDD-Functional-Specs.md` §5.6

## 3. Actores y permisos

| Rol | Permiso |
|-----|---------|
| JEFE_ZONA | Solicitar prep brief para tiendas de su zona |
| GTE_VENTAS | Solicitar prep brief para cualquier tienda de sus JZ |
| GERENTE_REGIONAL | Solicitar prep brief para cualquier tienda de su region |
| COO, CEO | Solicitar prep brief para cualquier tienda |

Permiso controlado por la visibilidad jerarquica existente (cascada `GRIP-SDD-Functional-Specs.md` §5.1.3 EMP-05).

## 4. Flujos principales

### 4.1 Solicitar brief de preparacion

1. JZ navega a la tienda y selecciona "Preparar Visita".
2. Frontend llama `GET /ai/store-visit-prep/{store_id}`.
3. Backend verifica que el usuario tiene visibilidad sobre la tienda.
4. Backend recopila el contexto (inputs definidos en §5.8.7 AI-01):
   - `last_5_checklist_sessions`: ultimos 5 ChecklistSession de la tienda con scores.
   - `open_followup_items`: FollowUpItems abiertos de la tienda (status OPEN o IN_PROGRESS).
   - `kardex_kpis_last_30d`: vacio en MVP (placeholder para P-04).
   - `previous_variable_items`: items variables usados en las ultimas 3 visitas.
   - `peer_store_patterns`: patrones detectados en tiendas de la misma zona.
5. Backend verifica cache (key: `store_visit_prep:{store_id}`, TTL 30 min).
   - Si hay cache valido, retorna directamente.
6. Backend construye prompt segun §5.8.7 AI-01 (system prompt exacto definido en SDD) y envia a Gemini.
7. Backend parsea respuesta, la cachea y la retorna al frontend.

### 4.2 Fallback por fallo de Gemini

1. Si la llamada a Gemini falla (timeout, quota, error), se activa fallback deterministico (BR-AI-03).
2. El fallback retorna follow-ups abiertos ordenados por aging (mas antiguos primero).
3. Se incluye flag `ai_generated: false` en la respuesta para que el frontend muestre indicador de modo degradado.

### 4.3 Visualizar brief en frontend

1. Frontend muestra el brief con secciones: Areas Prioritarias, Hallazgos Abiertos, Alertas KPI, Items Variables Sugeridos.
2. Si `ai_generated: false`, se muestra banner "Resumen basado en datos — IA no disponible temporalmente".
3. JZ puede proceder a iniciar el checklist desde esta pantalla.

## 5. Reglas de negocio

| ID | Regla |
|----|-------|
| BR-AI-01 | Toda llamada a Gemini debe incluir contexto estructurado; nunca enviar datos crudos sin preprocesar. |
| BR-AI-02 | Las sugerencias de IA son orientativas; el JZ decide que items incluir en la visita. |
| BR-AI-03 | Un fallo de IA nunca bloquea el flujo de visita. Degradar a fallback deterministico. |
| BR-AI-04 | El cache de prep brief tiene TTL de 30 minutos. Invalidar si se completa un checklist o follow-up en la tienda. |
| BR-AI-05 | Los datos enviados a Gemini deben respetar el tenant isolation; nunca incluir datos de otro tenant en el prompt. |

## 6. Contrato API

### `GET /ai/store-visit-prep/{store_id}`

**Request:** Path param `store_id` (UUID).

**Response 200 (schema from §5.8.7 AI-01):**

```json
{
  "store_id": "uuid",
  "generated_at": "datetime",
  "ai_generated": true,
  "cached": false,
  "priority_areas": ["string"],
  "open_findings_summary": {
    "total": 0,
    "overdue": 0,
    "critical": [
      {
        "finding_id": "uuid",
        "description": "string",
        "aging_days": 0
      }
    ]
  },
  "kpi_alerts": ["string"],
  "suggested_variable_items": [
    {
      "description": "string",
      "category": "string",
      "priority": "HIGH | MEDIUM | LOW",
      "justification": "string",
      "source_type": "AI_SUGGESTED | HISTORICAL | PEER_PATTERN"
    }
  ],
  "data_sources": ["string"]
}
```

**Response 403:** Usuario sin visibilidad sobre la tienda.

**Response 404:** Tienda no encontrada.

## 7. Cambios en UI (frontend)

- **Pantalla "Preparar Visita"** (`/stores/{store_id}/visit-prep`):
  - Card "Areas Prioritarias" con lista ordenada por relevancia.
  - Card "Hallazgos Abiertos" con resumen (`total`, `overdue`) y lista de criticos.
  - Card "Alertas KPI" (vacio en MVP, mostrar placeholder "Sin datos de KARDEX").
  - Card "Items Variables Sugeridos" con chips de categoria y prioridad; boton para aceptar sugerencias y agregarlas al checklist (via C-02).
  - Banner de modo degradado cuando `ai_generated: false`.
  - Boton "Iniciar Checklist" que navega a C-01.
- **Guard:** Reutilizar guard de visibilidad jerarquica existente.

## 8. Criterios de aceptacion

- [ ] **Dado** un JZ con tiendas asignadas, **cuando** solicita prep brief para una tienda de su zona, **entonces** recibe respuesta con `priority_areas`, `open_findings_summary`, `kpi_alerts` y `suggested_variable_items`.
- [ ] **Dado** que Gemini responde correctamente, **cuando** se solicita el mismo brief dentro de 30 min, **entonces** se retorna del cache (`cached: true`).
- [ ] **Dado** que Gemini falla (timeout, quota o error), **cuando** se solicita un brief, **entonces** se retorna fallback con follow-ups ordenados por aging y `ai_generated: false`.
- [ ] **Dado** un JZ, **cuando** solicita prep brief para una tienda fuera de su zona, **entonces** recibe 403.
- [ ] **Dado** que se completa un checklist en la tienda, **cuando** se solicita prep brief, **entonces** el cache previo esta invalidado y se genera nuevo brief.
- [ ] **Dado** que KARDEX no esta integrado (MVP), **cuando** se genera el brief, **entonces** `kpi_alerts` esta vacio y el frontend muestra placeholder correspondiente.
- [ ] Toda llamada a Gemini esta envuelta en `try/except` capturando `GeminiAPIError`, `GeminiQuotaError`, `GeminiTimeoutError`.

## 9. Notas

- La integracion con KARDEX (P-04) no esta en MVP. El campo `kardex_kpis_last_30d` se envia vacio al prompt. Cuando P-04 se implemente, solo se necesita poblar ese campo.
- El system prompt exacto esta definido en `GRIP-SDD-Functional-Specs.md` §5.8.7 AI-01. No duplicar aqui; referenciar el SDD.
- `peer_store_patterns` se calcula a partir de checklists de tiendas en la misma zona; si no hay datos suficientes, se omite del contexto.
