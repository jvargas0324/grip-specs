# C-03 — Weekly Report Cascade 4 Levels

- **Estado:** borrador
- **Prioridad:** Core MVP
- **Depende de:** B-01 (Role entity), B-02 (reports_to hierarchy), B-04 (JWT Auth)
- **SDD referencia:** GRIP-SDD-Functional-Specs.md §5.2 (WR-01 a WR-08, BR-WR-01 a BR-WR-03), §3.5 (Weekly Report Cascade)
- **Ultima actualizacion:** 2026-03-29

---

## 1. Objetivo

Implementar el sistema de reportes semanales con cascada de 5 niveles definido en el Ritmo Semanal (§3.5). Cada nivel consolida los reportes de sus subordinados y agrega su propia narrativa. El flagging de items para follow-up conecta con el sistema de FollowUpItem (WR-04, BR-WR-03).

## 2. Referencia a specs base

Toda la especificacion normativa vive en **GRIP-SDD-Functional-Specs.md**:

| Seccion | Contenido |
|---------|-----------|
| §3.5 | Ritmo Semanal: definicion de los 5 niveles de cascada |
| §5.2.1 | Modelo WeeklyReport: campos, estados, relaciones |
| §5.2.2 | Flujo de creacion y envio |
| §5.2.3 | Cascada: consolidacion por nivel |
| §5.2.4 | Flagging para follow-up |
| Appendix B | Endpoints de API |

## 3. Actores y permisos

| Actor | Acciones permitidas |
|-------|---------------------|
| JT | Crear y enviar reporte semanal de su tienda |
| JZ / JL | Crear reporte consolidando reportes de JTs; enviar a GV/GL |
| GV / GL | Crear reporte consolidando reportes de JZs/JLs; enviar a Regional |
| Regional | Crear reporte consolidando reportes de GVs/GLs; enviar a COO |
| COO | Crear reporte consolidando reportes de Regionales; enviar a CEO |
| CEO | Leer reportes de COO; flag items para follow-up |
| Cualquier nivel superior | Flag items de reportes de subordinados para follow-up (WR-04) |

## 4. Flujos principales

### 4.1 Niveles de cascada (§3.5)

| Nivel | Autor | Destinatario | Contenido |
|-------|-------|--------------|-----------|
| 1 | JZ / JL | GV / GL | Resumen de visitas a tiendas de la semana |
| 2 | GV / GL | Regional | Consolidacion de zona/linea |
| 3 | Regional | COO | Consolidacion regional |
| 4 | COO | CEO | Resumen ejecutivo nacional |
| 5 | CEO | (lectura) | Vista consolidada final |

### 4.2 Crear reporte semanal

1. El autor selecciona semana y anio (WR-01).
2. Backend crea WeeklyReport con status = `DRAFT` (WR-02).
3. El autor redacta el contenido (texto libre con formato) (WR-03).
4. Puede guardar como borrador multiples veces.

### 4.3 Enviar reporte

1. El autor marca el reporte como `SUBMITTED` (WR-05).
2. El reporte queda visible para su supervisor directo (via reports_to de B-02).
3. No se puede editar despues de enviar (WR-06).

### 4.4 Consolidacion por nivel superior

1. El supervisor ve los reportes enviados por sus subordinados directos (WR-07).
2. Puede crear su propio reporte referenciando los de sus subordinados.
3. Se presenta vista lado-a-lado: reportes recibidos + editor del propio.

### 4.5 Flag para follow-up

1. Cualquier nivel superior puede marcar un item de un reporte como "flagged for follow-up" (WR-04).
2. Backend crea un FollowUpItem con source_type = `WEEKLY_REPORT` (BR-WR-03).
3. El flagged_by se registra en el WeeklyReport (WR-08).

## 5. Reglas de negocio

| ID | Regla | Origen SDD |
|----|-------|------------|
| BR-WR-01 | Solo puede existir un WeeklyReport por autor, semana y anio | §5.2 BR-WR-01 |
| BR-WR-02 | Un reporte en status SUBMITTED no puede ser editado | §5.2 BR-WR-02 |
| BR-WR-03 | Al flag-ear un item se crea un FollowUpItem con source_type = WEEKLY_REPORT y referencia al reporte | §5.2 BR-WR-03 |

## 6. Contrato API

Base path: `/api/v1/`

| Metodo | Endpoint | Descripcion |
|--------|----------|-------------|
| POST | `/weekly-reports` | Crear nuevo WeeklyReport (DRAFT) |
| GET | `/weekly-reports/{report_id}` | Obtener reporte por ID |
| PATCH | `/weekly-reports/{report_id}` | Actualizar contenido (solo si DRAFT) |
| POST | `/weekly-reports/{report_id}/submit` | Cambiar status a SUBMITTED |
| GET | `/weekly-reports` | Listar reportes con filtros (author, week, year, status) |
| GET | `/weekly-reports/subordinates` | Listar reportes de subordinados directos para consolidacion |
| POST | `/weekly-reports/{report_id}/flag` | Marcar item para follow-up (crea FollowUpItem) |
| DELETE | `/weekly-reports/{report_id}/flag` | Desmarcar flag (solo si FollowUpItem aun esta OPEN) |

### Schemas principales

**WeeklyReportCreate:**
```json
{
  "week_number": 13,
  "year": 2026,
  "content": "string (markdown)"
}
```

**WeeklyReportResponse:**
```json
{
  "id": "uuid",
  "author_id": "uuid",
  "week_number": 13,
  "year": 2026,
  "content": "string",
  "status": "DRAFT | SUBMITTED",
  "flagged_for_followup": false,
  "flagged_by": "uuid | null",
  "created_at": "datetime",
  "submitted_at": "datetime | null"
}
```

**FlagRequest:**
```json
{
  "reason": "string",
  "priority": "HIGH | MEDIUM | LOW"
}
```

## 7. Cambios en UI (frontend)

| Pantalla | Descripcion |
|----------|-------------|
| Weekly Report List | Lista de reportes propios (DRAFT/SUBMITTED), filtro por semana |
| Report Editor | Editor markdown con preview, boton Guardar Borrador y Enviar |
| Subordinate Reports | Vista de reportes recibidos de subordinados, opcion de flag |
| Consolidation View | Panel dividido: reportes recibidos a la izquierda, editor propio a la derecha |
| Flag Dialog | Modal para confirmar flag con razon y prioridad |

### Requisitos UX

- El editor soporta formato basico (negritas, listas, encabezados).
- Indicador visual claro de status DRAFT vs SUBMITTED.
- Badge de conteo de reportes pendientes de subordinados.

## 8. Criterios de aceptacion

- [ ] Un JZ puede crear, editar y enviar un reporte semanal (WR-01 a WR-05).
- [ ] Un reporte SUBMITTED no puede editarse (WR-06, BR-WR-02).
- [ ] Solo un reporte por autor/semana/anio (BR-WR-01).
- [ ] Un GV ve los reportes de sus JZs subordinados (WR-07, usa B-02 reports_to).
- [ ] Flag de un item crea FollowUpItem con source_type = WEEKLY_REPORT (WR-04, BR-WR-03).
- [ ] La cascada funciona correctamente en los 5 niveles: JZ->GV->Regional->COO->CEO (§3.5).
- [ ] Endpoints protegidos con JWT (B-04) y tenant isolation (B-03).

## 9. Notas

- El PoC tiene un modelo WeeklySession que debe ser reemplazado por WeeklyReport con los campos del SDD (author_id, week_number, year, content, status [DRAFT, SUBMITTED], flagged_for_followup, flagged_by).
- B-02 proporciona los endpoints /tree, /ancestors y /scope que C-03 usa para resolver la jerarquia de subordinados.
- El SDD en §3.5 define 5 niveles de cascada dentro del Ritmo Semanal. La numeracion de niveles sigue la jerarquia organizacional.
