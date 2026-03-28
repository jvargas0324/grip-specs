# ADR-001: Persistencia hibrida PostgreSQL + pgvector

- **Estado**: aceptado (retrospectivo)
- **Fecha**: 2026-03-23
- **Tipo**: retrospectivo

## Titulo

Persistencia hibrida relacional + vectorial como base del backend.

## Decision a registrar

El backend usa PostgreSQL como sistema transaccional principal y pgvector para capacidades semanticas (embeddings e indices de similitud) en entidades de conocimiento.

## Evidencia encontrada en el proyecto

- `specs/technical-spec.md`: define arquitectura core con PostgreSQL + pgvector y modelo con embeddings.
- `grip-backend/alembic/versions/0001_create_extensions.py`: habilita extension `vector`.
- `grip-backend/alembic/versions/0002_create_all_tables.py`: crea indices HNSW para embeddings.
- `grip-backend/alembic/versions/0003_embedding_768.py`: consolida dimension `Vector(768)`.
- `grip-backend/app/db/models/findings.py` y `grip-backend/app/db/models/strategic_feedback.py`: campos de embedding vectorial.

## Problema que resuelve

Permite combinar integridad transaccional del dominio operativo con recuperacion semantica para flujos IA/RAG.

## Riesgos o limitaciones

- Crecimiento de costo y latencia por volumen de embeddings.
- Necesidad de politica operacional de retencion/indexado para mantener rendimiento.
- Dependencia de tuning del motor pgvector segun carga real.

## Notas de alcance

No se observa en este ADR una politica completa de capacidad/SLO; eso queda como decision prospectiva separada.

