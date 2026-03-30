# C-04 — AspectLibrary (36) + ThreePointReview

- **Estado:** borrador
- **Prioridad:** Core MVP
- **Depende de:** B-01 (Role entity), B-02 (reports_to hierarchy), B-07 (FollowUpItem entity)
- **SDD referencia:** GRIP-SDD-Functional-Specs.md §5.4 (TP-01 a TP-16, BR-TP-01 a BR-TP-10), §3.3 (3-Point Review Execution Matrix)
- **Ultima actualizacion:** 2026-03-29

---

## 1. Objetivo

Implementar el modulo completo de revision de 3 puntos (ThreePointReview): la libreria de 36 aspectos (AspectLibrary), la seleccion de 3 aspectos por visita con 5 fuentes de datos, la ejecucion y calificacion de la revision, el tracking de cobertura (3 aspectos x 12 meses = 36), y los 5 efectos en cascada definidos en §5.4.5. Este es un modulo completamente nuevo sin precedente en el PoC.

## 2. Referencia a specs base

Toda la especificacion normativa vive en **GRIP-SDD-Functional-Specs.md**:

| Seccion | Contenido |
|---------|-----------|
| §3.3 | 3-Point Review Execution Matrix: quien revisa a quien |
| §5.4.1 | Modelo de cobertura: 3 aspectos x 12 meses = 36 |
| §5.4.2 | AspectLibrary: 36 aspectos codificados A01-A36 |
| §5.4.3 | Fuentes de datos A-E para seleccion de aspectos |
| §5.4.4 | Ejecucion de la revision |
| §5.4.5 | 5 efectos en cascada (post-revision) |
| §5.4.6 | Data model: AspectLibrary, Aspect, ThreePointReview, ReviewedAspect, AspectCoverage |
| §5.8.7 AI-02 | 3-Point Review Prep (suggested_aspects, coverage_alert) |

## 3. Actores y permisos

| Actor | Acciones permitidas |
|-------|---------------------|
| JZ / JL | Ejecutar 3-Point Review sobre JTs de su zona/linea |
| GV / GL | Ejecutar 3-Point Review sobre JZs/JLs; ver cobertura de su gerencia |
| Regional | Ejecutar 3-Point Review sobre GVs/GLs; ver cobertura regional |
| COO | Ver cobertura nacional; ejecutar review sobre Regionales |
| CEO | Ver dashboards de cobertura consolidada |

## 4. Flujos principales

### 4.1 Seleccion de aspectos (5 fuentes de datos — §5.4.3)

1. Al iniciar una revision, el sistema sugiere 3 aspectos basados en 5 fuentes:
   - **A. Cobertura pendiente:** Aspectos que el colaborador no ha sido evaluado en el ciclo actual (AspectCoverage).
   - **B. Follow-ups abiertos:** Aspectos relacionados con FollowUpItems OPEN o IN_PROGRESS del colaborador.
   - **C. Resultados de checklist:** Aspectos vinculados a secciones con bajo score reciente.
   - **D. Sugerencias de IA:** AI-02 sugiere aspectos basado en analisis (suggested_aspects, coverage_alert).
   - **E. Seleccion manual:** El revisor puede sobrescribir la seleccion automatica.
2. El revisor confirma o ajusta los 3 aspectos seleccionados (TP-01, TP-02).

### 4.2 Ejecucion de la revision

1. Para cada aspecto seleccionado, el revisor observa al colaborador y registra (TP-03 a TP-05):
   - Calificacion (escala definida en SDD).
   - Observaciones de texto libre.
   - Evidencia fotografica opcional.
2. El revisor puede agregar feedback cualitativo general (TP-06).
3. Al completar los 3 aspectos, finaliza la revision (TP-07).

### 4.3 Efectos en cascada (§5.4.5)

Post-finalizacion se ejecutan 5 efectos:

1. **Efecto 1 — Actualizacion de cobertura:** Se actualiza AspectCoverage para los 3 aspectos revisados (TP-08).
2. **Efecto 2 — Feedback loop a Follow-Up:** Si un aspecto tiene FollowUpItems relacionados, se actualiza el registro con resultado de la revision (TP-09). Conecta con C-05.
3. **Efecto 3 — Generacion de nuevos Follow-Ups:** Si la calificacion es insuficiente, se crea FollowUpItem con source_type = `THREE_POINT_REVIEW` (TP-10).
4. **Efecto 4 — Notificacion al colaborador:** El colaborador revisado recibe notificacion con resultados (TP-11).
5. **Efecto 5 — Alimentacion de dashboards:** Los resultados se reflejan en widgets de dashboard (TP-12). Conecta con C-07.

### 4.4 Tracking de cobertura

1. AspectCoverage registra que aspectos han sido revisados para cada colaborador en el ciclo actual (TP-13).
2. Meta: 3 aspectos por visita x 12 meses = 36 aspectos cubiertos en el anio (§5.4.1, TP-14).
3. Dashboard muestra porcentaje de cobertura por colaborador y por equipo (TP-15).
4. Alerta cuando la cobertura esta por debajo del ritmo esperado (TP-16).

## 5. Reglas de negocio

| ID | Regla | Origen SDD |
|----|-------|------------|
| BR-TP-01 | Cada ThreePointReview evalua exactamente 3 aspectos | §5.4 BR-TP-01 |
| BR-TP-02 | Los aspectos se seleccionan de la AspectLibrary (codigos A01-A36) | §5.4 BR-TP-02 |
| BR-TP-03 | No se puede repetir un aspecto ya cubierto en el ciclo actual salvo justificacion explicita | §5.4 BR-TP-03 |
| BR-TP-04 | El revisor solo puede evaluar a colaboradores en su scope (reports_to) | §5.4 BR-TP-04 |
| BR-TP-05 | La cobertura se resetea al inicio de cada ciclo anual | §5.4 BR-TP-05 |
| BR-TP-06 | Una calificacion insuficiente genera automaticamente un FollowUpItem | §5.4 BR-TP-06 |
| BR-TP-07 | El colaborador revisado debe poder ver sus resultados | §5.4 BR-TP-07 |
| BR-TP-08 | La matriz de ejecucion (§3.3) define quien puede revisar a quien segun rol | §5.4 BR-TP-08 |
| BR-TP-09 | AI-02 debe degradar elegantemente si no esta disponible | §5.4 BR-TP-09 |
| BR-TP-10 | Los resultados alimentan los dashboards por rol (§5.7) | §5.4 BR-TP-10 |

## 6. Contrato API

Base path: `/api/v1/`

| Metodo | Endpoint | Descripcion |
|--------|----------|-------------|
| GET | `/aspects` | Listar AspectLibrary completa (A01-A36) |
| GET | `/aspects/{code}` | Detalle de un aspecto por codigo |
| POST | `/three-point-reviews` | Crear nueva ThreePointReview |
| GET | `/three-point-reviews/{review_id}` | Obtener review con aspectos revisados |
| PATCH | `/three-point-reviews/{review_id}` | Actualizar review (solo si no finalizada) |
| POST | `/three-point-reviews/{review_id}/complete` | Finalizar review (dispara efectos en cascada) |
| GET | `/three-point-reviews` | Listar reviews con filtros (reviewer, reviewee, fecha) |
| POST | `/three-point-reviews/{review_id}/aspects` | Registrar resultado de un aspecto revisado |
| GET | `/coverage/{employee_id}` | Obtener AspectCoverage de un colaborador |
| GET | `/coverage/team` | Obtener cobertura agregada del equipo |
| POST | `/three-point-reviews/suggest-aspects` | Solicitar sugerencias de AI-02 |

### Schemas principales

**ThreePointReviewCreate:**
```json
{
  "reviewee_id": "uuid",
  "store_id": "uuid",
  "aspect_codes": ["A05", "A12", "A28"]
}
```

**ReviewedAspectCreate:**
```json
{
  "aspect_code": "A05",
  "rating": 3,
  "observation": "string",
  "photo_ids": ["uuid"]
}
```

**AspectCoverageResponse:**
```json
{
  "employee_id": "uuid",
  "cycle_year": 2026,
  "total_aspects": 36,
  "covered_aspects": 18,
  "coverage_percentage": 50.0,
  "covered_codes": ["A01", "A02", "A05"],
  "pending_codes": ["A03", "A04", "A06"]
}
```

**AI-02 Output Schema:**
```json
{
  "suggested_aspects": [
    {"code": "A12", "reason": "string"},
    {"code": "A05", "reason": "string"},
    {"code": "A28", "reason": "string"}
  ],
  "coverage_alert": "string | null"
}
```

## 7. Cambios en UI (frontend)

| Pantalla | Descripcion |
|----------|-------------|
| Aspect Library | Catalogo consultable de 36 aspectos con busqueda y filtros |
| Start Review | Selector de colaborador y tienda, sugerencia de aspectos con fuentes, boton confirmar |
| Review Execution | Formulario por aspecto: calificacion, observacion, foto. Navegacion entre los 3 aspectos |
| Review Summary | Resumen post-finalizacion con calificaciones, follow-ups generados, cobertura actualizada |
| Coverage Dashboard | Heatmap de cobertura por colaborador (36 celdas), porcentaje por equipo |
| AI Suggestions | Panel con sugerencias de AI-02 y coverage_alert |

### Requisitos UX

- La seleccion de aspectos muestra la fuente (cobertura, follow-up, checklist, IA, manual) con badge visual.
- El heatmap de cobertura usa colores para indicar aspectos cubiertos vs pendientes.
- Mobile-friendly para ejecucion en tienda.

## 8. Criterios de aceptacion

- [ ] La AspectLibrary contiene 36 aspectos codificados A01-A36 (TP-01, BR-TP-02).
- [ ] Se pueden seleccionar exactamente 3 aspectos para una revision (BR-TP-01).
- [ ] Las 5 fuentes de datos (A-E) funcionan para sugerir aspectos (§5.4.3).
- [ ] Al finalizar la revision se ejecutan los 5 efectos en cascada (§5.4.5, TP-08 a TP-12).
- [ ] La cobertura se actualiza correctamente al finalizar (TP-13, TP-14).
- [ ] Calificacion insuficiente genera FollowUpItem con source_type = THREE_POINT_REVIEW (TP-10, BR-TP-06).
- [ ] El revisor solo puede evaluar colaboradores en su scope (BR-TP-04, BR-TP-08).
- [ ] No se repiten aspectos cubiertos en el ciclo sin justificacion (BR-TP-03).
- [ ] AI-02 degrada elegantemente si no esta disponible (BR-TP-09).
- [ ] Endpoints protegidos con JWT (B-04) y tenant isolation (B-03).

## 9. Notas

- Este es un modulo completamente nuevo. No existe precedente en el PoC.
- El data model (AspectLibrary, Aspect, ThreePointReview, ReviewedAspect, AspectCoverage) se define en §5.4.6 del SDD.
- La Execution Matrix de §3.3 define las relaciones revisor-revisado segun el rol organizacional; se implementa usando la jerarquia reports_to de B-02.
- El seed de los 36 aspectos (A01-A36) debe ser parte de la migracion inicial del modulo.
- AI-02 (3-Point Review Prep) se invoca con `try/except` capturando errores de Gemini.
