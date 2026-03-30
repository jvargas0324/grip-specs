# C-01 — Checklist Digital Mobile-First

- **Estado:** borrador
- **Prioridad:** Core MVP
- **Depende de:** B-06 (ChecklistSession, ChecklistSessionItem, VariableItem tables), B-07 (FollowUpItem entity)
- **SDD referencia:** GRIP-SDD-Functional-Specs.md §5.3 (CL-01 a CL-13, BR-CL-01 a BR-CL-06)
- **Ultima actualizacion:** 2026-03-29

---

## 1. Objetivo

Entregar una experiencia mobile-first para completar checklists de visita en tienda. El llenado debe ser tap-not-type, completarse en menos de 10 minutos (CL-07), generar un score summary al finalizar (CL-09), y crear automaticamente FollowUpItems a partir de respuestas "No Cumple" (CL-10).

## 2. Referencia a specs base

Toda la especificacion normativa vive en **GRIP-SDD-Functional-Specs.md**:

| Seccion | Contenido |
|---------|-----------|
| §5.3.1 | Estructura de ChecklistTemplate: secciones e items |
| §5.3.2 | Flujo de ejecucion de ChecklistSession |
| §5.3.3 | Tipos de respuesta (CUMPLE / NO_CUMPLE / NA / PHOTO) |
| §5.3.4 | Score calculation y summary |
| §5.3.5 | Variable items (detalle en C-02) |
| §5.3.6 | Integracion con Follow-Up (auto-generacion) |
| §5.3.7 | Fotos adjuntas |
| Appendix B | Endpoints de API |

## 3. Actores y permisos

| Actor | Acciones permitidas |
|-------|---------------------|
| JT (Jefe de Tienda) | Ejecutar checklist en su tienda, ver historico propio |
| JZ / JL | Ejecutar checklist en tiendas de su zona/linea, ver historico de zona/linea |
| GV / GL | Ver checklists de su gerencia, ver scores agregados |
| Regional / COO / CEO | Ver dashboards con KPIs de checklist |

## 4. Flujos principales

### 4.1 Iniciar sesion de checklist

1. Usuario selecciona tienda y confirma inicio (CL-01).
2. Backend crea ChecklistSession con status `IN_PROGRESS` (CL-02).
3. Se cargan las secciones e items del ChecklistTemplate activo (CL-03).
4. Se inyectan variable items si existen (ver C-02) (CL-05, CL-06).

### 4.2 Responder items (tap-not-type)

1. Para cada item, el usuario toca CUMPLE, NO_CUMPLE o NA (CL-04).
2. Si responde NO_CUMPLE, se habilita campo de observacion (texto libre) y captura de foto opcional (CL-08).
3. Las fotos se suben como adjuntos vinculados al ChecklistResponse (CL-12).
4. El progreso se guarda en cada respuesta (no requiere submit final para persistir parciales) (CL-11).

### 4.3 Finalizar y generar score

1. Al completar todos los items obligatorios, el usuario toca "Finalizar" (CL-09).
2. Backend calcula score por seccion y score total (CL-09, BR-CL-01).
3. Backend marca ChecklistSession como `COMPLETED` (CL-13).
4. Se presenta pantalla resumen con score por seccion y total.

### 4.4 Auto-generacion de follow-ups

1. Por cada respuesta NO_CUMPLE, el backend crea un FollowUpItem (CL-10, BR-CL-05).
2. El FollowUpItem se asigna al JT de la tienda con source_type = `CHECKLIST`.
3. Se establece deadline segun BR-CL-06.
4. El FollowUpItem hereda la referencia a la ChecklistSession y al item especifico.

## 5. Reglas de negocio

| ID | Regla | Origen SDD |
|----|-------|------------|
| BR-CL-01 | El score se calcula como porcentaje de items CUMPLE sobre total de items aplicables (excluyendo NA) | §5.3 BR-CL-01 |
| BR-CL-02 | Un checklist no puede finalizarse si quedan items obligatorios sin responder | §5.3 BR-CL-02 |
| BR-CL-03 | Solo puede haber un ChecklistSession activo por tienda por dia | §5.3 BR-CL-03 |
| BR-CL-04 | Los variable items son de alcance de sesion (session-scoped) | §5.3 BR-CL-04 |
| BR-CL-05 | Cada respuesta NO_CUMPLE genera automaticamente un FollowUpItem | §5.3 BR-CL-05 |
| BR-CL-06 | El deadline del FollowUpItem generado se calcula segun la prioridad del item | §5.3 BR-CL-06 |

## 6. Contrato API

Base path: `/api/v1/`

| Metodo | Endpoint | Descripcion |
|--------|----------|-------------|
| POST | `/checklists/sessions` | Crear nueva ChecklistSession para una tienda |
| GET | `/checklists/sessions/{session_id}` | Obtener sesion con items y respuestas |
| PATCH | `/checklists/sessions/{session_id}` | Actualizar status de sesion (finalizar) |
| POST | `/checklists/sessions/{session_id}/responses` | Registrar respuesta a un item |
| PATCH | `/checklists/sessions/{session_id}/responses/{response_id}` | Actualizar respuesta existente |
| POST | `/checklists/sessions/{session_id}/responses/{response_id}/photos` | Subir foto adjunta |
| GET | `/checklists/sessions/{session_id}/summary` | Obtener score summary |
| GET | `/checklists/sessions` | Listar sesiones con filtros (tienda, fecha, status) |

### Schemas principales

**ChecklistSessionCreate:**
```json
{
  "store_id": "uuid",
  "template_id": "uuid"
}
```

**ChecklistResponseCreate:**
```json
{
  "item_id": "uuid",
  "value": "CUMPLE | NO_CUMPLE | NA",
  "observation": "string | null"
}
```

**ChecklistSummary:**
```json
{
  "session_id": "uuid",
  "total_score": 85.5,
  "sections": [
    {
      "section_id": "uuid",
      "section_name": "string",
      "score": 90.0,
      "total_items": 5,
      "cumple": 4,
      "no_cumple": 1,
      "na": 0
    }
  ],
  "follow_ups_generated": 3
}
```

## 7. Cambios en UI (frontend)

| Pantalla | Descripcion |
|----------|-------------|
| Checklist Home | Lista de checklists pendientes y completados, filtro por tienda |
| Checklist Execution | Vista mobile-first con secciones colapsables, botones tap CUMPLE/NO_CUMPLE/NA, barra de progreso |
| Observation Drawer | Panel deslizante para texto de observacion y boton de camara |
| Photo Capture | Captura o seleccion de foto, preview con opcion de eliminar |
| Score Summary | Vista resumen post-finalizacion con grafico de barras por seccion |
| Follow-Up Preview | Lista de follow-ups generados automaticamente al finalizar |

### Requisitos UX mobile

- Botones de respuesta con area minima de 48x48 dp.
- Swipe horizontal entre secciones.
- Guardado automatico por respuesta (sin boton "Guardar" intermedio).
- Funcionalidad offline: persistir respuestas en IndexedDB y sincronizar al reconectar.

## 8. Criterios de aceptacion

- [ ] Un JT puede iniciar un checklist, responder todos los items con taps y finalizar en menos de 10 minutos (CL-07).
- [ ] El score por seccion y total se calcula correctamente al finalizar (CL-09, BR-CL-01).
- [ ] No se puede finalizar con items obligatorios sin responder (BR-CL-02).
- [ ] Solo un checklist activo por tienda por dia (BR-CL-03).
- [ ] Cada NO_CUMPLE genera un FollowUpItem con source_type = CHECKLIST (CL-10, BR-CL-05).
- [ ] Las fotos se suben y vinculan correctamente al ChecklistResponse (CL-12).
- [ ] El progreso parcial se persiste sin necesidad de finalizar (CL-11).
- [ ] La UI funciona correctamente en pantallas de 360px de ancho minimo.
- [ ] Todos los endpoints responden con tenant isolation (B-03).
- [ ] Endpoints protegidos con JWT (B-04).

## 9. Notas

- B-06 ya creo las tablas ChecklistSession, ChecklistSessionItem y VariableItem (migracion 0016). C-01 construye los endpoints y la UX sobre esa base.
- B-07 ya creo la tabla FollowUpItem (migracion 0017). C-01 usa esa tabla para la auto-generacion.
- Los variable items se detallan en C-02; C-01 solo necesita renderizarlos si estan presentes en la sesion.
- El prefix de API en la implementacion es `/api/v1/` (decision de implementacion vs Appendix B del SDD que usa `/api/`).
