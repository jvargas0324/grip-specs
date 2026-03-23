# Temas que requieren RFC vs temas que caben como ADR

## Regla de clasificacion usada

- **RFC**: cuando cambia semantica de negocio, compliance, contratos externos o procesos transversales.
- **ADR**: cuando documenta una decision tecnica de arquitectura/implementacion ya vigente o acotada internamente.

## Requieren RFC de evolucion

1. **Reincidencia DA (definicion normativa unica)**  
   Afecta negocio, auditoria, criterio de riesgo y experiencia operativa.

2. **Modelo de MFA para firma ejecutiva**  
   Afecta cumplimiento, seguridad, identidad y potencialmente procesos legales.

3. **Plan de deprecacion del alias legado de chat**  
   Impacta consumidores de API, versionado y comunicacion externa.

## Caben como ADR (sin RFC previo, o posterior al RFC cuando aplique)

1. **Persistencia hibrida PostgreSQL + pgvector** (retrospectivo).
2. **Resiliencia IA por degradacion/no bloqueo** (retrospectivo).
3. **Gobernanza R1-R6 con trazabilidad** (retrospectivo).
4. **Estrategia tecnica de escalado vectorial** (prospectivo tecnico, si no altera contrato externo).
5. **Politica de observabilidad y SLO backend** (prospectivo tecnico).

## Dependencias recomendadas

- Si un tema aparece arriba en RFC y tambien tiene implicaciones tecnicas, secuencia recomendada:
  1) RFC aprueba direccion de negocio/compliance.
  2) ADR aterriza implementacion tecnica.

