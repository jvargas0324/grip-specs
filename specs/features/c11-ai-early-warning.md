# C-11 — AI-05 Early Warning Job

- **Estado:** borrador
- **Prioridad:** Core MVP
- **Depende de:** B-07 (FollowUpItem entity), C-01 (Checklist Digital), C-04 (Aspect Library & 3-Point Review), C-05 (Auto-Surface Engine)
- **SDD referencia:** §5.8.7 AI-05 (Early Warning), §5.8.3.3 AI-EW-01 a AI-EW-04, BR-AI-01 a BR-AI-05
- **Usa IA/Gemini:** Si (opcional — el motor primario es heuristico/rule-based)
- **Ultima actualizacion:** 2026-03-29

---

## 1. Objetivo

- **Aplicacion(es):** grip-backend, grip-frontend
- **Problema:** Los problemas sistemicos (tiendas en declive, JZ con baja tasa de cierre, issues recurrentes en multiples tiendas) se detectan tarde porque requieren cruzar datos de multiples fuentes y periodos. Un sistema de alertas tempranas permite intervenir antes de que los problemas se agraven.
- **Alcance:**
  - **grip-backend:** Job programado (cron) que ejecuta 4 tipos de deteccion heuristica (AI-EW-01 a AI-EW-04). Opcionalmente usa Gemini para enriquecer la narrativa de las alertas. Persiste warnings en DB para consulta via API.
  - **grip-frontend:** Panel de alertas tempranas en dashboard, con filtros por tipo y severidad.
  - **Fuera de alcance:** Acciones automaticas basadas en warnings (requiere aprobacion humana), integracion KARDEX para KPI trends (P-04).

## 2. Referencia a specs base

- Modelo de prompt y output: `GRIP-SDD-Functional-Specs.md` §5.8.7 AI-05
- 4 tipos de alerta: `GRIP-SDD-Functional-Specs.md` §5.8.3.3 (AI-EW-01 a AI-EW-04)
- Reglas de resiliencia IA: `GRIP-SDD-Functional-Specs.md` §5.8 (BR-AI-01 a BR-AI-05)
- Follow-Up data: `GRIP-SDD-Functional-Specs.md` §5.6
- Checklist scores: `GRIP-SDD-Functional-Specs.md` §5.3
- 3-Point Review coverage: `GRIP-SDD-Functional-Specs.md` §5.4

## 3. Actores y permisos

| Rol | Permiso |
|-----|---------|
| GTE_VENTAS | Ver warnings de sus JZ y tiendas de su scope |
| GERENTE_REGIONAL | Ver warnings de su region |
| COO | Ver warnings operativos de toda la organizacion |
| CEO | Ver todos los warnings |

Los warnings se filtran por la visibilidad jerarquica del usuario (cascada `GRIP-SDD-Functional-Specs.md` §5.1.3 EMP-05).

## 4. Flujos principales

### 4.1 Ejecucion del job (cron diario)

El job se ejecuta diariamente (configurable). Recopila el contexto definido en §5.8.7 AI-05:
- `checklist_scores_per_store_last_8_visits`
- `followup_closure_rate_per_jz_last_60d`
- `kardex_kpi_trends_last_12_weeks` (vacio en MVP hasta P-04)
- `three_point_scores_per_employee_last_3_cycles`
- `thresholds` (configurables por tenant)

Para cada tipo de alerta:

**AI-EW-01 — Performance Decline (tasa de cierre de FU):**
1. Calcular tasa de cierre de follow-ups por JZ en los ultimos 60 dias.
2. Comparar con la tasa de los 60 dias anteriores.
3. Si la tasa cae >15% en 2 meses consecutivos, generar warning con severity proporcional al delta.

**AI-EW-02 — Store Trending Down (scores de checklist):**
1. Para cada tienda, obtener scores de los ultimos 8 checklists.
2. Si los scores declinan en 4+ visitas consecutivas, generar warning.
3. Severity basada en la magnitud del declive.

**AI-EW-03 — Systemic Issue Emerging (mismos hallazgos en multiples tiendas):**
1. Agrupar findings abiertos por tipo/categoria.
2. Si el mismo tipo aparece en 5+ tiendas de 3+ DMs distintos, generar warning.
3. Severity HIGH por naturaleza (indica problema sistemico).

**AI-EW-04 — Coverage Gap (aspectos sin revisar):**
1. Para cada empleado evaluable, verificar aspectos de 3-Point Review.
2. Si un aspecto no ha sido revisado en 6+ meses, generar warning.
3. Severity basada en la antiguedad del gap.

### 4.2 Enriquecimiento AI (opcional)

1. Tras generar warnings heuristicos, opcionalmente enviar batch a Gemini para:
   - Sintetizar narrativa de patrones cruzados.
   - Ajustar confidence (CONFIRMED vs WEAK_SIGNAL).
2. Si Gemini falla, los warnings heuristicos se persisten tal cual (BR-AI-03). El motor primario es rule-based.

### 4.3 Consultar warnings (API)

1. Usuario accede al panel de alertas tempranas.
2. Frontend llama `GET /ai/early-warnings` con filtros opcionales.
3. Backend retorna warnings filtrados por visibilidad jerarquica del usuario.

### 4.4 Fallback

- El job completo es primariamente heuristico. Si Gemini no esta disponible, las alertas se generan igualmente con logica mecanica (comparacion contra thresholds).
- Flag `ai_enriched` indica si la narrativa fue generada por AI o es template mecanico.

## 5. Reglas de negocio

| ID | Regla |
|----|-------|
| AI-EW-01 | Performance decline: tasa de cierre de FU cae >15% en 2 meses consecutivos. |
| AI-EW-02 | Store trending down: scores de checklist declinan en 4+ visitas consecutivas. |
| AI-EW-03 | Systemic issue: mismo tipo de hallazgo en 5+ tiendas de 3+ DMs. |
| AI-EW-04 | Coverage gap: aspecto de 3-Point Review no revisado en 6+ meses. |
| BR-AI-01 | Contexto estructurado para Gemini; nunca datos crudos. |
| BR-AI-03 | Fallo de IA no bloquea generacion de warnings. Motor heuristico es el primario. |
| BR-AI-05 | Tenant isolation en datos enviados a Gemini. |

## 6. Contrato API

### `GET /ai/early-warnings`

**Query params:**
- `warning_type` (opcional): `CHECKLIST_DECLINE | FOLLOWUP_STAGNATION | KPI_DETERIORATION | PERFORMANCE_DECLINE`
- `severity` (opcional): `CRITICAL | HIGH | MEDIUM | LOW`
- `entity_type` (opcional): `STORE | EMPLOYEE | REGION`
- `limit` (opcional, default 50): maximo de warnings a retornar.
- `offset` (opcional, default 0): paginacion.

**Response 200 (schema from §5.8.7 AI-05):**

```json
{
  "total": 0,
  "warnings": [
    {
      "warning_id": "uuid",
      "entity_type": "STORE | EMPLOYEE | REGION",
      "entity_id": "uuid",
      "entity_name": "string",
      "warning_type": "CHECKLIST_DECLINE | FOLLOWUP_STAGNATION | KPI_DETERIORATION | PERFORMANCE_DECLINE",
      "severity": "CRITICAL | HIGH | MEDIUM | LOW",
      "confidence": "CONFIRMED | WEAK_SIGNAL",
      "trend_data": [
        {
          "period": "string",
          "value": 0.0
        }
      ],
      "recommended_action": "string",
      "detected_at": "datetime",
      "ai_enriched": true
    }
  ],
  "data_sources": ["string"]
}
```

**Response 403:** Usuario sin permisos para ver warnings.

### `POST /ai/early-warnings/run` (admin/internal)

Trigger manual del job. Solo CEO o admin.

**Response 202:** Job enqueued.

## 7. Cambios en UI (frontend)

- **Panel "Alertas Tempranas"** (`/dashboard/early-warnings`):
  - Tabla/lista de warnings con columnas: entidad, tipo, severidad, confianza, fecha deteccion.
  - Filtros por tipo de warning (`warning_type`), severidad, tipo de entidad.
  - Detalle expandible con `trend_data` (grafico sparkline) y `recommended_action`.
  - Badge de confianza: CONFIRMED (solido) vs WEAK_SIGNAL (outline).
  - Badge `ai_enriched` para diferenciar narrativa AI vs template mecanico.
- **Widget en dashboard principal (C-07):**
  - Contador de warnings activos por severidad (CRITICAL en rojo, HIGH en naranja).
  - Click navega al panel completo.
- **Guard:** Visibilidad filtrada por jerarquia; no se necesita guard adicional.

## 8. Criterios de aceptacion

- [ ] **Dado** un JZ con tasa de cierre de FU que cae >15% en 2 meses, **cuando** se ejecuta el job, **entonces** se genera warning tipo `PERFORMANCE_DECLINE` (AI-EW-01).
- [ ] **Dado** una tienda con scores de checklist declinando en 4+ visitas consecutivas, **cuando** se ejecuta el job, **entonces** se genera warning tipo `CHECKLIST_DECLINE` (AI-EW-02).
- [ ] **Dado** el mismo tipo de hallazgo en 5+ tiendas de 3+ DMs, **cuando** se ejecuta el job, **entonces** se genera warning con severity HIGH indicando issue sistemico (AI-EW-03).
- [ ] **Dado** un aspecto no revisado en 6+ meses para un empleado, **cuando** se ejecuta el job, **entonces** se genera warning de coverage gap (AI-EW-04).
- [ ] **Dado** que Gemini no esta disponible, **cuando** se ejecuta el job, **entonces** los warnings se generan igualmente con logica heuristica y `ai_enriched: false`.
- [ ] **Dado** un GTE_VENTAS, **cuando** consulta warnings, **entonces** solo ve warnings de entidades dentro de su scope jerarquico (EMP-05).
- [ ] Toda llamada a Gemini (enriquecimiento) esta envuelta en `try/except` capturando `GeminiAPIError`, `GeminiQuotaError`, `GeminiTimeoutError`.
- [ ] El job es idempotente — no duplica warnings para el mismo periodo y entidad.
- [ ] El job puede ejecutarse manualmente via `POST /ai/early-warnings/run` (solo CEO/admin).

## 9. Notas

- Este feature es **primariamente heuristico/rule-based**. La IA es opcional y se usa solo para enriquecer la narrativa y ajustar confianza. El job debe funcionar al 100% sin Gemini.
- `kardex_kpi_trends_last_12_weeks` (contexto del SDD §5.8.7 AI-05) estara vacio hasta P-04. Los warnings se limitaran a metricas internas (scores de checklist, tasas de cierre) en MVP.
- El system prompt para enriquecimiento esta definido en `GRIP-SDD-Functional-Specs.md` §5.8.7 AI-05.
- La frecuencia del cron (diaria por defecto) es configurable via variable de entorno.
- C-07 (Dashboards) consume estas alertas para mostrarlas en widgets.
