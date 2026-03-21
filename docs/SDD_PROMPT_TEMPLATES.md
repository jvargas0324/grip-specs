# Plantillas de prompt SDD (Cursor)

Uso recomendado: copiar la plantilla que corresponda al chat o Composer, sustituir los marcadores y **citar siempre los SSOT con `@`** (por ejemplo `@specs/technical-spec.md`) para que el modelo cargue el contrato antes de codificar.

Marcadores habituales:

- `<nombre-feature>` — slug kebab-case, alineado con `specs/features/<nombre-feature>.md` y rama `feature/<nombre-feature>`.

---

## 1. Solo especificación (sin código)

Objetivo: actualizar documentos maestros y/o la spec de feature; **no** implementar en repos de código en el mismo paso.

```text
Tarea: solo documentación / contrato (SDD).

Contexto obligatorio — revisar y citar con @:
- specs/technical-spec.md
- specs/functional-spec.md (si cambian journeys o índice R1–R6)
- specs/api-contract-ssot.md (si cambia el mapa spec ↔ Pydantic)
- specs/features/<nombre-feature>.md

Pedido concreto:
[Describe qué secciones cambian y qué decisiones de producto/contrato aplican.]

Restricciones:
- No modificar código en grip-backend ni grip-frontend en este paso.
- Si afecta a reglas R1–R6 o versión global, indica qué actualizar en specs/backend_audit_report.md y el campo Versión en technical-spec.md según el flujo del proyecto.
```

---

## 2. Implementación acotada a una feature

Objetivo: código alineado a una spec ya definida y contrato actualizado.

```text
Tarea: implementar la feature <nombre-feature>.

Contexto obligatorio — revisar y citar con @:
- specs/features/<nombre-feature>.md (criterios de aceptación)
- specs/technical-spec.md (API, R1–R6, stack)
- specs/api-contract-ssot.md (schemas y rutas en grip-backend)

Orden de trabajo:
1. grip-specs — solo si falta cerrar contrato o criterios (si ya está hecho, saltar).
2. grip-backend — schemas Pydantic, routers, servicios, migraciones Alembic si aplica.
3. grip-frontend — componentes standalone, servicios por feature, Tailwind.

Rama: feature/<nombre-feature> en cada repo tocado.

Restricciones:
- Comentarios de negocio en backend: # Implementa R*:
- Llamadas a Gemini: try/except y degradación según technical-spec.md §3 y §3.F.
```

---

## 3. Cierre de feature (PR / alineación)

Objetivo: cerrar el círculo SDD antes de merge: criterios, auditoría y versión.

```text
Tarea: cierre de feature <nombre-feature>.

Contexto obligatorio — revisar y citar con @:
- specs/features/<nombre-feature>.md (marcar o validar criterios de aceptación)
- specs/backend_audit_report.md (trazabilidad R* y endpoints si cambiaron)
- specs/technical-spec.md (reconciliar Versión si el contrato o R1–R6 cambiaron)
- specs/functional-spec.md §6.0 si el índice R1–R6 funcional debe actualizarse

Pedido concreto:
[Checklist: criterios cumplidos, audit actualizado, versión alineada, sin contradicciones entre docs.]

No añadir lógica nueva salvo que sea imprescindible para cumplir la spec ya acordada.
```

---

## Referencia rápida de SSOT

| Documento | Rol |
|-----------|-----|
| `specs/technical-spec.md` | Contrato técnico, R1–R6, API, DB |
| `specs/functional-spec.md` | Journeys, índice R1–R6 funcional |
| `specs/api-contract-ssot.md` | Mapa technical-spec ↔ Pydantic |
| `specs/backend_audit_report.md` | Trazabilidad implementación vs spec |
| `specs/features/<nombre>.md` | Spec y criterios por entrega |

Más detalle: [README.md](../README.md) y [.cursorrules](../.cursorrules).
