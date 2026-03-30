# C-07 — Dashboards (8 Roles)

- **Estado:** borrador
- **Prioridad:** Core MVP
- **Depende de:** B-01 a B-07 (todos los B-items), C-01 (Checklist), C-03 (Weekly Report), C-04 (ThreePointReview), C-05 (Auto-surface), C-06 (Escalation)
- **SDD referencia:** GRIP-SDD-Functional-Specs.md §5.7 (DB-01 a DB-12, BR-DB-01 a BR-DB-06), §5.7.3.1 a §5.7.3.8, §5.7.4
- **Ultima actualizacion:** 2026-03-29

---

## 1. Objetivo

Implementar los 8 dashboards por rol definidos en §5.7.3, cada uno con su "pregunta principal" y widgets especificos. Los dashboards consolidan datos de checklists, reportes semanales, revisiones de 3 puntos, follow-ups, escalaciones y Forgotten Findings. La matriz de widgets (§5.7.4) define que widgets aparecen en cada dashboard.

## 2. Referencia a specs base

Toda la especificacion normativa vive en **GRIP-SDD-Functional-Specs.md**:

| Seccion | Contenido |
|---------|-----------|
| §5.7.1 | Principios generales de dashboards |
| §5.7.2 | Widgets disponibles (DB-01 a DB-12) |
| §5.7.3.1 | Dashboard JT |
| §5.7.3.2 | Dashboard JZ |
| §5.7.3.3 | Dashboard JL |
| §5.7.3.4 | Dashboard GV |
| §5.7.3.5 | Dashboard GL |
| §5.7.3.6 | Dashboard Regional |
| §5.7.3.7 | Dashboard COO |
| §5.7.3.8 | Dashboard CEO |
| §5.7.4 | Widget cross-reference matrix (widgets x roles) |

## 3. Actores y permisos

| Actor | Dashboard | Pregunta principal |
|-------|-----------|-------------------|
| JT | §5.7.3.1 | Como esta mi tienda hoy? |
| JZ | §5.7.3.2 | Como estan mis tiendas esta semana? |
| JL | §5.7.3.3 | Como esta mi linea esta semana? |
| GV | §5.7.3.4 | Como va mi gerencia de ventas? |
| GL | §5.7.3.5 | Como va mi gerencia de linea? |
| Regional | §5.7.3.6 | Como va mi region? |
| COO | §5.7.3.7 | Como va la operacion nacional? |
| CEO | §5.7.3.8 | Como va el negocio? |

Cada usuario solo ve el dashboard correspondiente a su rol. Los datos se filtran por su scope organizacional (B-02).

## 4. Flujos principales

### 4.1 Carga de dashboard

1. Al acceder al dashboard, el frontend detecta el rol del usuario (B-01, B-04).
2. Se consultan los widgets correspondientes segun la matriz §5.7.4.
3. Cada widget se carga con datos filtrados por el scope del usuario (B-02 /scope).
4. Los widgets se renderizan en layout responsive.

### 4.2 Widgets disponibles (DB-01 a DB-12)

| Widget ID | Nombre | Datos | Visible desde |
|-----------|--------|-------|---------------|
| DB-01 | Checklist Score actual | Score del ultimo checklist por tienda | JT+ |
| DB-02 | Checklist Trend | Tendencia de scores de checklist (ultimas N semanas) | JZ+ |
| DB-03 | Follow-Up Status | Conteo de follow-ups por status (OPEN, IN_PROGRESS, RESOLVED, VERIFIED, ESCALATED) | JT+ |
| DB-04 | Escalation Count | Numero de escalaciones activas | GV+ |
| DB-05 | Weekly Report Status | Estado de reportes semanales (enviados vs pendientes) | JZ+ |
| DB-06 | 3-Point Coverage | Porcentaje de cobertura de aspectos por colaborador | JZ+ |
| DB-07 | Top Issues | Items de follow-up mas recurrentes (mas veces resurgidos) | GV+ |
| DB-08 | Forgotten Findings | Items OPEN sin actualizacion > N dias (BR-DB-03) | GV+ |
| DB-09 | Store Ranking | Ranking de tiendas por score de checklist | GV+ |
| DB-10 | Team Performance | Metricas de desempeno por equipo | Regional+ |
| DB-11 | National Summary | KPIs consolidados nacionales | COO+ |
| DB-12 | Executive Overview | Vista ejecutiva con indicadores clave | CEO |

### 4.3 Widget cross-reference matrix (§5.7.4)

| Widget | JT | JZ | JL | GV | GL | Regional | COO | CEO |
|--------|----|----|----|----|----|----|-----|-----|
| DB-01 | x | x | x | x | x | x | x | x |
| DB-02 | | x | x | x | x | x | x | x |
| DB-03 | x | x | x | x | x | x | x | x |
| DB-04 | | | | x | x | x | x | x |
| DB-05 | | x | x | x | x | x | x | x |
| DB-06 | | x | x | x | x | x | x | x |
| DB-07 | | | | x | x | x | x | x |
| DB-08 | | | | x | x | x | x | x |
| DB-09 | | | | x | x | x | x | x |
| DB-10 | | | | | | x | x | x |
| DB-11 | | | | | | | x | x |
| DB-12 | | | | | | | | x |

> **Nota:** Esta matriz es una aproximacion basada en §5.7.3 y §5.7.4. La version normativa esta en el SDD.

### 4.4 Drill-down

1. Cada widget permite drill-down a los datos detallados.
2. Ejemplo: click en DB-08 (Forgotten Findings) abre lista filtrada de follow-ups olvidados.
3. El drill-down respeta el scope del usuario.

## 5. Reglas de negocio

| ID | Regla | Origen SDD |
|----|-------|------------|
| BR-DB-01 | Cada dashboard muestra solo datos dentro del scope organizacional del usuario | §5.7 BR-DB-01 |
| BR-DB-02 | Los scores de checklist se agregan por tienda, zona, gerencia segun el nivel | §5.7 BR-DB-02 |
| BR-DB-03 | Forgotten Findings: FollowUpItem OPEN sin actualizacion > N dias (configurable por tenant) | §5.7 BR-DB-03 |
| BR-DB-04 | Los widgets se actualizan en tiempo real o near-real-time (definicion exacta en SDD) | §5.7 BR-DB-04 |
| BR-DB-05 | El ranking de tiendas (DB-09) se calcula sobre los ultimos 30 dias | §5.7 BR-DB-05 |
| BR-DB-06 | Los dashboards deben cargar en menos de 3 segundos | §5.7 BR-DB-06 |

## 6. Contrato API

Base path: `/api/v1/`

| Metodo | Endpoint | Descripcion |
|--------|----------|-------------|
| GET | `/dashboards/me` | Obtener configuracion de dashboard para el usuario autenticado (widgets segun rol) |
| GET | `/dashboards/widgets/checklist-score` | DB-01: Score actual por tienda(s) en scope |
| GET | `/dashboards/widgets/checklist-trend` | DB-02: Tendencia de scores |
| GET | `/dashboards/widgets/followup-status` | DB-03: Conteo de follow-ups por status |
| GET | `/dashboards/widgets/escalation-count` | DB-04: Escalaciones activas |
| GET | `/dashboards/widgets/weekly-report-status` | DB-05: Estado de reportes semanales |
| GET | `/dashboards/widgets/coverage` | DB-06: Cobertura de 3-Point Review |
| GET | `/dashboards/widgets/top-issues` | DB-07: Issues mas recurrentes |
| GET | `/dashboards/widgets/forgotten-findings` | DB-08: Forgotten Findings |
| GET | `/dashboards/widgets/store-ranking` | DB-09: Ranking de tiendas |
| GET | `/dashboards/widgets/team-performance` | DB-10: Desempeno por equipo |
| GET | `/dashboards/widgets/national-summary` | DB-11: Resumen nacional |
| GET | `/dashboards/widgets/executive-overview` | DB-12: Vista ejecutiva |

### Schemas principales

**DashboardConfig:**
```json
{
  "role": "GV",
  "widgets": ["DB-01", "DB-02", "DB-03", "DB-04", "DB-05", "DB-06", "DB-07", "DB-08", "DB-09"],
  "scope": {
    "type": "gerencia",
    "id": "uuid"
  }
}
```

**WidgetData (ejemplo DB-01):**
```json
{
  "widget_id": "DB-01",
  "title": "Checklist Score Actual",
  "data": [
    {
      "store_id": "uuid",
      "store_name": "Tienda Centro",
      "last_score": 87.5,
      "last_checklist_date": "2026-03-28"
    }
  ],
  "updated_at": "datetime"
}
```

**ForgottenFindingsWidget (DB-08):**
```json
{
  "widget_id": "DB-08",
  "title": "Forgotten Findings",
  "total_count": 8,
  "by_priority": {
    "HIGH": 3,
    "MEDIUM": 4,
    "LOW": 1
  },
  "oldest_days": 15
}
```

## 7. Cambios en UI (frontend)

| Pantalla | Descripcion |
|----------|-------------|
| Dashboard Home | Layout responsivo con grid de widgets segun rol del usuario |
| Widget Cards | Cada widget como card con titulo, valor principal, mini-grafico o lista |
| Drill-Down Views | Vistas de detalle accesibles desde cada widget |
| Filters | Selector de periodo (semana, mes, trimestre) aplicable a todos los widgets |

### Requisitos UX

- Layout responsive: 1 columna en mobile, 2-3 columnas en tablet/desktop.
- Widgets con loading skeleton mientras cargan.
- Refresh manual por pull-to-refresh en mobile.
- Graficos con libreria liviana (Chart.js o similar via Angular wrapper).
- Color coding consistente: verde (bien), amarillo (atencion), rojo (critico).

### Dashboards por rol (resumen visual)

- **JT:** DB-01, DB-03. Vista simple de "mi tienda".
- **JZ/JL:** DB-01, DB-02, DB-03, DB-05, DB-06. Vista de "mis tiendas esta semana".
- **GV/GL:** DB-01 a DB-09. Vista completa de gerencia con escalaciones y forgotten findings.
- **Regional:** DB-01 a DB-10. Agrega team performance.
- **COO:** DB-01 a DB-11. Agrega resumen nacional.
- **CEO:** DB-01 a DB-12. Vista ejecutiva completa.

## 8. Criterios de aceptacion

- [ ] Cada rol ve solo los widgets correspondientes segun la matriz §5.7.4 (BR-DB-01).
- [ ] Los datos se filtran por el scope organizacional del usuario (B-02).
- [ ] DB-01: muestra score correcto del ultimo checklist por tienda (C-01).
- [ ] DB-03: conteo correcto de follow-ups por status (B-07, C-05).
- [ ] DB-04: escalaciones activas reflejan items de C-06.
- [ ] DB-05: estado de reportes semanales refleja datos de C-03.
- [ ] DB-06: cobertura de 3-Point Review refleja datos de C-04.
- [ ] DB-08: Forgotten Findings calcula correctamente items OPEN > N dias (BR-DB-03, C-05).
- [ ] DB-09: ranking de tiendas sobre ultimos 30 dias (BR-DB-05).
- [ ] Dashboard carga en menos de 3 segundos (BR-DB-06).
- [ ] 8 dashboards implementados: JT, JZ, JL, GV, GL, Regional, COO, CEO (§5.7.3.1 a §5.7.3.8).
- [ ] Endpoints protegidos con JWT (B-04) y tenant isolation (B-03).

## 9. Notas

- El SDD define 8 dashboards (§5.7.3.1 a §5.7.3.8), no 7. Los 8 roles con dashboard son: JT, JZ, JL, GV, GL, Regional, COO, CEO.
- La widget cross-reference matrix en §5.7.4 es la referencia normativa para que widgets aparecen en cada dashboard. La tabla en §4.3 de esta spec es una aproximacion; consultar el SDD para la version definitiva.
- C-07 es el ultimo feature del Core MVP y depende de todos los anteriores para tener datos que mostrar.
- La estrategia de caching (Redis, materialized views, etc.) es decision de implementacion del backend. El requisito es que cargue en < 3s (BR-DB-06).
- Los widgets son componentes Angular standalone reutilizables. Cada widget recibe su configuracion y datos via Input/Output.
