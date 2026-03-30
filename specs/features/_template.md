# [Nombre de la feature]

> Copia este archivo a `specs/features/<nombre>.md` para cada nueva feature. Rellena las secciones y úsalo como SSOT antes de implementar (SDD). Ejemplo rellenado: [_example-sdd-reference.md](_example-sdd-reference.md).

- **Estado:** borrador | spec-cerrada | en-implementación | completada | abandonada
- **Spec base alineada:** vX.XX (versión de `GRIP-SDD-Functional-Specs.md` contra la que se redactó/actualizó)
- **Última actualización:** YYYY-MM-DD

## 1. Objetivo

- **Aplicación(es):** Indicar una o varias: **grip-backend** | **grip-frontend**. Si aplica a varias, listar y describir qué parte toca cada una en "Alcance".
- **Problema o necesidad:** ¿Qué se resuelve o para quién?
- **Alcance:** Por aplicación: qué entra y qué queda fuera (grip-backend, grip-frontend). Permisos si aplica.

## 2. Referencia a specs base

- Reglas de negocio existentes que aplican: [GRIP-SDD-Functional-Specs.md](../GRIP-SDD-Functional-Specs.md) (secciones X, Y).
- Contrato y arquitectura: [GRIP-SDD-Functional-Specs.md](../GRIP-SDD-Functional-Specs.md). Nuevos endpoints o cambios deben alinearse con la API actual.
- Si la feature usa **IA / Gemini:** [GRIP-SDD-Functional-Specs.md §3](../GRIP-SDD-Functional-Specs.md) y §3.F (resiliencia).
- Si la feature toca **reglas R1–R6** o estados de hallazgos / DA / CEO: [GRIP-SDD-Functional-Specs.md §4](../GRIP-SDD-Functional-Specs.md) y [GRIP-SDD-Functional-Specs.md §6.0](../GRIP-SDD-Functional-Specs.md).

## 3. Actores y permisos

- Qué roles pueden usar esta feature.
- Permisos nuevos (si aplica) y en qué operaciones.

## 4. Flujos principales

- Descripción en lenguaje de negocio (paso a paso).
- Alternativas y excepciones (ej. validación fallida, sin permiso).

## 5. Reglas de negocio

- Lista numerada de reglas específicas de esta feature.
- Validaciones, restricciones, estados permitidos.

## 6. Contrato API

- **Endpoints nuevos o modificados:** método, path, request/response (campos y tipos).
- Si es poco: tabla o lista; si es mucho: fragmento OpenAPI o archivo en `specs/features/<nombre>/`.

## 7. Cambios en UI (frontend)

- Pantallas o componentes nuevos.
- Rutas, permisos/guards, integración con ApiService.

## 8. Criterios de aceptación

- Formato "Dado … cuando … entonces …" por flujo o regla clave.
- Lista corta y comprobable.

## 9. Notas

- Dependencias con otras features, migraciones de datos, etc.
