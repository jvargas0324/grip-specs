# ADR Candidates Priorizados (backend escalable)

## Criterio de lectura

- `tipo`: `retrospectivo` cuando ya existe evidencia fuerte de decision vigente.
- `tipo`: `prospectivo` cuando aun falta definicion formal.
- Priorizacion basada en riesgo operativo/escalabilidad y advertencias existentes.

## 1) Unificar autenticacion y ciclo de vida del alias de chat/RAG

- **decision a registrar**: el alias legado de chat debe tener la misma politica de autenticacion que el endpoint canonico o ser deprecado.
- **evidencia encontrada**: warning en `specs/backend_audit_report.md` sobre inconsistencia de proteccion entre alias y endpoint canonico.
- **problema que resuelve**: reduce superficie de acceso inconsistente y riesgo de bypass.
- **riesgos o limitaciones**: impacto en clientes integrados al alias; requiere plan de migracion.
- **prioridad**: alta
- **tipo**: prospectivo

## 2) Formalizar criterio de reincidencia DA

- **decision a registrar**: definir algoritmo canonico (regla unica) para reincidencia de DA y su trazabilidad en pruebas.
- **evidencia encontrada**: `specs/GRIP-SDD-Functional-Specs.md` describe umbral orientativo; audit report refleja implementacion heuristica no totalmente equivalente.
- **problema que resuelve**: evita decisiones de negocio ambiguas y comportamientos no deterministas.
- **riesgos o limitaciones**: impacto en reglas de negocio, UX, auditoria y reportes historicos.
- **prioridad**: alta
- **tipo**: prospectivo

## 3) Politica de MFA real para firma ejecutiva (R5)

- **decision a registrar**: exigir MFA verificable para firma de cierre DA y evidencias de identidad.
- **evidencia encontrada**: warning en `specs/backend_audit_report.md` indicando validacion parcial actual.
- **problema que resuelve**: mejora control de identidad y cumplimiento en operaciones sensibles.
- **riesgos o limitaciones**: complejidad de integracion con proveedor de identidad y friccion operativa.
- **prioridad**: alta
- **tipo**: prospectivo

## 4) Persistencia hibrida PostgreSQL + pgvector

- **decision a registrar**: mantener base relacional-operativa con enriquecimiento vectorial en entidades clave.
- **evidencia encontrada**:
  - `specs/GRIP-SDD-Functional-Specs.md` (arquitectura core y modelo con embeddings),
  - migraciones Alembic con extension vector e indices HNSW en `grip-backend/alembic/versions/`,
  - modelos con `Vector(768)` en `grip-backend/app/db/models/`.
- **problema que resuelve**: combina integridad transaccional con recuperacion semantica para RAG/IA.
- **riesgos o limitaciones**: costo de indexado y crecimiento de almacenamiento/latencia bajo alta cardinalidad.
- **prioridad**: media
- **tipo**: retrospectivo

## 5) Resiliencia IA por degradacion y no bloqueo de transacciones criticas

- **decision a registrar**: ante fallo IA, persistir resultado principal y degradar campos IA no criticos.
- **evidencia encontrada**:
  - `specs/GRIP-SDD-Functional-Specs.md` (seccion de resiliencia),
  - trazabilidad en `specs/backend_audit_report.md`,
  - manejo de excepciones/fallback en servicios de `grip-backend/app/services/`.
- **problema que resuelve**: evita indisponibilidad funcional por dependencia externa IA.
- **riesgos o limitaciones**: salida parcial puede afectar calidad percibida si no hay reintento posterior.
- **prioridad**: media
- **tipo**: retrospectivo

## 6) Gobernanza por reglas R1-R6 con trazabilidad spec-codigo

- **decision a registrar**: mantener reglas R1-R6 como contrato normativo y trazabilidad obligatoria en backend.
- **evidencia encontrada**:
  - definicion en `specs/GRIP-SDD-Functional-Specs.md`,
  - indice funcional en `specs/GRIP-SDD-Functional-Specs.md`,
  - mapeo en `specs/backend_audit_report.md`.
- **problema que resuelve**: alinea negocio, API y codigo bajo una unica fuente verificable.
- **riesgos o limitaciones**: costo de mantenimiento documental y disciplina de equipo requerida.
- **prioridad**: media
- **tipo**: retrospectivo

## 7) Estrategia de escalado vectorial y capacidad

- **decision a registrar**: politicas de indexado, retencion, archivado, presupuesto de latencia/costo para busqueda vectorial.
- **evidencia encontrada**: existe uso de pgvector/HNSW, pero no se observa politica operacional completa en docs base.
- **problema que resuelve**: previene degradacion progresiva de rendimiento y costos no controlados.
- **riesgos o limitaciones**: requiere datos de carga reales y metas de SLO.
- **prioridad**: media
- **tipo**: prospectivo

## 8) Politica de observabilidad y SLO backend

- **decision a registrar**: estandar de logs estructurados, metricas y trazas para endpoints criticos.
- **evidencia encontrada**: salud basica y logging existente; no se evidencia especificacion operativa integral.
- **problema que resuelve**: reduce MTTR y mejora control de capacidad.
- **riesgos o limitaciones**: sobrecosto inicial de instrumentacion y operacion.
- **prioridad**: baja-media
- **tipo**: prospectivo

