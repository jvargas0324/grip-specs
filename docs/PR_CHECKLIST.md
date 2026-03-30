# Checklist de PR (SDD)

Objetivo: que cada PR sea revisable contra la **única fuente de verdad** y que el orden **spec → backend → frontend** quede explícito.

## Antes de abrir el PR

1. **Rama** — `feature/<nombre-kebab>` alineada con `specs/features/<nombre-kebab>.md` (misma convención que en [.cursorrules](../.cursorrules)).
2. **Repos tocados** — Si el cambio es transversal, conviene PRs coordinados o un mismo número de issue enlazando:
   - primero contrato/docs en **grip-specs** (cuando el contrato o R* cambien);
   - luego **grip-backend**;
   - luego **grip-frontend**.

## Cuerpo del PR (plantilla mínima)

Copia o adapta lo siguiente. Sustituye `<nombre-feature>` por el slug real.

```markdown
## Feature
- Spec: `grip-specs/specs/features/<nombre-feature>.md` (o enlace al archivo en GitHub/GitLab).

## Criterios de aceptación
[Pega aquí la lista de la spec de feature, o enlaza la sección con ancla.]

## Cambios por repo
- [ ] grip-specs — (solo si aplica: technical-spec, api-contract-ssot, audit, nueva/edición de spec de feature)
- [ ] grip-backend — (endpoints, schemas, migraciones, servicios)
- [ ] grip-frontend — (UI, servicios por feature)

## Contrato y auditoría
- [ ] `api-contract-ssot.md` y schemas Pydantic coherentes (si hubo cambio de API o DTOs).
- [ ] `backend_audit_report.md` actualizado si cambiaron endpoints o reglas R1–R6.
- [ ] Versión en `GRIP-SDD-Functional-Specs.md` y encabezado del audit reconciliados si aplica.

## Notas para revisores
[Degradación Gemini, edge cases, flags de entorno, etc.]
```

## Revisión asistida por IA

Si usas Cursor en el review: adjunta o `@`-menciona `specs/features/<nombre-feature>.md` y las secciones de `GRIP-SDD-Functional-Specs.md` que definan el comportamiento esperado. Así el modelo valida contra spec, no contra suposiciones.

## Hotfix

Para correcciones urgentes fuera del flujo habitual, indícalo en el PR y enlaza la spec o issue que justifica la excepción.
