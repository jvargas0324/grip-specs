# C-10 — AI-03 DA Preparation Brief

- **Estado:** borrador
- **Prioridad:** Core MVP
- **Depende de:** B-05 (DA COO Scope), C-01 (Checklist Digital), C-03 (Weekly Report Cascade), C-05 (Auto-Surface Engine)
- **SDD referencia:** §5.8.7 AI-03 (DA Prep), §5.8.3.1 AI-PREP-03, §5.5.3 (6 Data Sources), BR-AI-01 a BR-AI-05, BR-DA-01
- **Usa IA/Gemini:** Si
- **Ultima actualizacion:** 2026-03-29

---

## 1. Objetivo

- **Aplicacion(es):** grip-backend, grip-frontend
- **Problema:** La preparacion de un Diagnostico de Area (DA) requiere sintetizar informacion de multiples fuentes: DAs anteriores, hallazgos abiertos, KPIs regionales, reportes semanales y escalaciones. Sin asistencia, el CEO/COO invierte tiempo excesivo recopilando datos y puede omitir hallazgos criticos.
- **Alcance:**
  - **grip-backend:** Endpoint que recopila las 6 fuentes de datos (§5.5.3), respeta el scope del rol (CEO=ALL_AREAS, COO=OPERATIONS_ONLY per BR-DA-01), y envia a Gemini para generar un brief ejecutivo. Sin cache (baja frecuencia, alta criticidad).
  - **grip-frontend:** Pantalla de preparacion DA con resumen ejecutivo, hallazgos no resueltos, areas de riesgo, anomalias KPI y areas de foco recomendadas.
  - **Fuera de alcance:** Ejecucion del DA (feature separada), Source 6 (datos financieros, no disponible en MVP).

## 2. Referencia a specs base

- Modelo de prompt y output: `GRIP-SDD-Functional-Specs.md` §5.8.7 AI-03
- Preparacion asistida: `GRIP-SDD-Functional-Specs.md` §5.8.3.1 AI-PREP-03
- 6 Data Sources del DA: `GRIP-SDD-Functional-Specs.md` §5.5.3
- Scope por rol (CEO/COO): `GRIP-SDD-Functional-Specs.md` §5.5 (BR-DA-01)
- Reglas de resiliencia IA: `GRIP-SDD-Functional-Specs.md` §5.8 (BR-AI-01 a BR-AI-05)

## 3. Actores y permisos

| Rol | Permiso | Scope |
|-----|---------|-------|
| CEO | Solicitar DA prep brief | ALL_AREAS — todas las areas de diagnostico |
| COO | Solicitar DA prep brief | OPERATIONS_ONLY — solo areas operativas (BR-DA-01) |

Ningun otro rol puede solicitar DA prep brief.

## 4. Flujos principales

### 4.1 Solicitar brief de preparacion DA

1. CEO/COO navega a "Preparar Diagnostico de Area".
2. Frontend llama `GET /ai/da-prep`.
3. Backend identifica el rol del solicitante y aplica scope:
   - **CEO:** `scope = ALL_AREAS`
   - **COO:** `scope = OPERATIONS_ONLY` (filtrado por `da_scope` en Role, per BR-DA-01)
4. Backend recopila las 6 fuentes de datos (§5.5.3):
   - **Source 1:** `previous_das`: DAs anteriores del scope del solicitante.
   - **Source 2:** `open_da_findings`: hallazgos de DA abiertos (no resueltos).
   - **Source 3:** `regional_kpi_summary_last_quarter`: resumen de KPIs regionales del ultimo trimestre.
   - **Source 4:** `weekly_reports_last_8_weeks`: reportes semanales de las ultimas 8 semanas.
   - **Source 5:** `escalated_followups`: follow-ups escalados pendientes.
   - **Source 6:** (datos financieros) — no disponible en MVP, se envia vacio.
5. Backend construye prompt segun §5.8.7 AI-03 (system prompt exacto definido en SDD) con el scope aplicado y envia a Gemini.
6. Backend parsea respuesta y la retorna al frontend.
7. **No se cachea** la respuesta (baja frecuencia, alta criticidad — datos deben estar siempre frescos).

### 4.2 Fallback por fallo de Gemini

1. Si la llamada a Gemini falla, se activa fallback deterministico (BR-AI-03).
2. El fallback retorna hallazgos de DA abiertos ordenados por aging (mas antiguos primero), sin sintesis ejecutiva.
3. Se incluye flag `ai_generated: false` en la respuesta.

### 4.3 Visualizar brief en frontend

1. Frontend muestra el brief con secciones: Resumen Ejecutivo, Hallazgos No Resueltos, Areas de Riesgo, Anomalias KPI, Areas de Foco Recomendadas.
2. Cada hallazgo no resuelto muestra `aging_days`, `cycles_open` y `severity`.
3. Areas de riesgo muestran nivel de riesgo y evidencia de soporte.
4. Si `ai_generated: false`, banner de modo degradado.

## 5. Reglas de negocio

| ID | Regla |
|----|-------|
| BR-DA-01 | CEO ve ALL_AREAS; COO ve OPERATIONS_ONLY. El scope se aplica tanto al contexto enviado a Gemini como a los resultados retornados. |
| BR-AI-01 | Toda llamada a Gemini debe incluir contexto estructurado; nunca enviar datos crudos sin preprocesar. |
| BR-AI-02 | Las sugerencias de IA son orientativas; el CEO/COO decide el foco del DA. |
| BR-AI-03 | Un fallo de IA nunca bloquea la preparacion del DA. Degradar a fallback deterministico. |
| BR-AI-05 | Los datos enviados a Gemini deben respetar el tenant isolation. |

## 6. Contrato API

### `GET /ai/da-prep`

**Request:** Sin parametros (scope inferido del rol del usuario autenticado).

**Response 200 (schema from §5.8.7 AI-03):**

```json
{
  "scope": "ALL_AREAS | OPERATIONS_ONLY",
  "generated_at": "datetime",
  "ai_generated": true,
  "executive_summary": "string",
  "unresolved_findings": [
    {
      "finding_id": "uuid",
      "description": "string",
      "aging_days": 0,
      "cycles_open": 0,
      "severity": "CRITICAL | HIGH | MEDIUM | LOW"
    }
  ],
  "risk_areas": [
    {
      "area": "string",
      "risk_level": "HIGH | MEDIUM | LOW",
      "evidence": ["string"]
    }
  ],
  "kpi_anomalies": [
    {
      "metric": "string",
      "value": 0.0,
      "expected": 0.0,
      "delta": 0.0
    }
  ],
  "recommended_focus_areas": ["string"],
  "data_sources": ["string"]
}
```

**Response 403:** Usuario sin rol CEO o COO.

## 7. Cambios en UI (frontend)

- **Pantalla "Preparar DA"** (`/da/prep`):
  - Card "Resumen Ejecutivo" con texto generado por AI (`executive_summary`).
  - Card "Hallazgos No Resueltos" con tabla: descripcion, aging, ciclos abiertos, severidad. Ordenados por severidad descendente.
  - Card "Areas de Riesgo" con nivel de riesgo (badge de color) y evidencia colapsable.
  - Card "Anomalias KPI" con tabla: metrica, valor actual, valor esperado, delta. Nota MVP: Source 6 no disponible.
  - Card "Areas de Foco Recomendadas" con lista priorizada.
  - Banner de modo degradado cuando `ai_generated: false`.
  - Boton "Iniciar DA" que navega al flujo de creacion de DA.
- **Guard:** Solo roles CEO y COO pueden acceder.

## 8. Criterios de aceptacion

- [ ] **Dado** un CEO, **cuando** solicita DA prep, **entonces** recibe brief con scope `ALL_AREAS` incluyendo `executive_summary`, `unresolved_findings`, `risk_areas`, `kpi_anomalies` y `recommended_focus_areas`.
- [ ] **Dado** un COO, **cuando** solicita DA prep, **entonces** recibe brief con scope `OPERATIONS_ONLY` y los datos filtrados solo a areas operativas (BR-DA-01).
- [ ] **Dado** que Gemini falla, **cuando** se solicita DA prep, **entonces** se retorna fallback con hallazgos abiertos ordenados por aging y `ai_generated: false`.
- [ ] **Dado** un usuario con rol distinto a CEO o COO, **cuando** solicita DA prep, **entonces** recibe 403.
- [ ] **Dado** que se solicita DA prep, **cuando** se recibe respuesta, **entonces** no proviene de cache (siempre datos frescos).
- [ ] **Dado** que Source 6 (financiero) no esta disponible en MVP, **cuando** se genera el brief, **entonces** el campo se omite del contexto sin error.
- [ ] Toda llamada a Gemini esta envuelta en `try/except` capturando `GeminiAPIError`, `GeminiQuotaError`, `GeminiTimeoutError`.

## 9. Notas

- Este es un endpoint de **baja frecuencia y alta criticidad**. No se cachea. Cada invocacion genera una llamada fresca a Gemini.
- Source 6 (datos financieros) no esta disponible en MVP. El prompt se estructura para funcionar sin este campo.
- El system prompt exacto esta definido en `GRIP-SDD-Functional-Specs.md` §5.8.7 AI-03. No duplicar aqui.
- B-05 (DA COO Scope) ya implementa `da_scope` en el modelo Role. Este spec lo consume.
