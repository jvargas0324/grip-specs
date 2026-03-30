# Diagnostico Arquitectonico Backend (estado vigente)

- **Fecha:** 2026-03-23
- **Spec base evaluada:** v7.20

## Alcance y metodo

- Alcance: estado actual del sistema backend y su coherencia con SDD.
- Fuentes usadas: `specs/GRIP-SDD-Functional-Specs.md`, `specs/GRIP-SDD-Functional-Specs.md`, `specs/api-contract-ssot.md`, `specs/backend_audit_report.md`, y estructura/implementacion en `grip-backend/app/**`, `grip-backend/alembic/**`.
- Criterio: solo inferencias con evidencia observable. Si falta evidencia, se declara explicitamente.

## 1) Estado actual

- Stack vigente: FastAPI + SQLAlchemy + Alembic + PostgreSQL/pgvector + Pydantic v2 + Gemini.
- Arquitectura por capas vigente en backend: `api/`, `services/`, `db/models/`.
- Contrato API gobernado por SSOT dual:
  - especificacion humana en `specs/GRIP-SDD-Functional-Specs.md` (seccion API),
  - validacion ejecutable en `grip-backend/app/schemas/` segun `specs/api-contract-ssot.md`.
- Persistencia hibrida relacional + vectorial:
  - modelos con embeddings (`Vector(768)`) y migraciones de pgvector/HNSW en Alembic.
- Resiliencia IA definida como degradacion controlada:
  - fallas de Gemini no deben bloquear transacciones criticas (reflejado en specs y audit report).

## 2) Problemas detectados (riesgos tecnicos y escalabilidad)

- **Alto**: consistencia de seguridad en alias de chat/RAG.
  - Evidencia en audit report: warning por diferencia de proteccion entre endpoint canonico y alias legado.
- **Medio-alto**: firma ejecutiva con MFA no plenamente implementada.
  - Evidencia en audit report: warning de R5 (control parcial).
- **Medio**: criterios de reincidencia DA no totalmente normados de forma unica.
  - Evidencia: diferencia entre umbral orientativo de spec y heuristica implementada.
- **Medio**: capa HTTP async con acceso DB sincrono (Session clasica).
  - Impacto potencial: limitacion de throughput bajo concurrencia alta.
- **Medio**: ausencia observable de estrategia formal de observabilidad avanzada.
  - No se evidencia estandar operativo de metricas/tracing/SLO en docs base.
- **Medio**: no hay politica explicitada de escalado vectorial a largo plazo.
  - Faltan definiciones de retencion, crecimiento de indices, capacidad y SLO de busqueda.

## 3) Decisiones ya tomadas de facto (con evidencia)

1. Persistencia hibrida PostgreSQL + pgvector como base de conocimiento operativo y semantico.
2. Degradacion controlada para IA (no bloqueo de operaciones criticas).
3. Arquitectura por capas (`api/services/models`) con tipado Pydantic v2.
4. Gobernanza funcional por reglas R1-R6 con trazabilidad spec-codigo.
5. Endpoint canonico de chat y coexistencia temporal de alias legado.
6. SDD como politica de evolucion (spec primero, implementacion despues).

## 4) Decisiones que faltan por tomar

- Politica final del alias legado de chat/RAG (mantener, deprecar, o retirar).
- Definicion canonica de reincidencia DA (algoritmo normativo y pruebas de aceptacion).
- Estrategia de escalado vectorial (indexado, retencion, costo y latencia objetivo).
- Politica de resiliencia formal por dependencia externa (timeouts/retries/circuit breaking).
- Politica de observabilidad y SLO de endpoints criticos.
- Estrategia formal de autenticacion reforzada para firma ejecutiva (MFA real).

## 5) Lista priorizada de ADR candidatos

Ver documento: `definitions/adr-candidates-prioritized.md`.

## 6) Que requiere RFC y que cabe como ADR

Ver documento: `definitions/rfc-vs-adr-recommendations.md`.

