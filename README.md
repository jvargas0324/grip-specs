# grip-specs — SSOT de GRIP (SDD)

Repositorio dedicado a **especificaciones**, **auditoría de contrato** y **reglas de Cursor** para el proyecto GRIP. No incluye código de aplicación.

## Contenido

| Ruta | Propósito |
|------|-----------|
| `.cursorrules` | Reglas de la IA en Cursor (SDD, stack, R1–R6). |
| `specs/technical-spec.md` | API, esquema de datos, prompts, reglas R1–R6 técnicas. |
| `specs/functional-spec.md` | Journeys, gobernanza, índice R1–R6 funcional. |
| `specs/api-contract-ssot.md` | Mapa spec ↔ Pydantic en el repo `grip-backend`. |
| `specs/backend_audit_report.md` | Trazabilidad implementación vs spec. |
| `specs/features/` | Plantilla y specs por feature. |
| `docs/` | Planes, plantillas SDD y checklist de PR (ver tabla abajo). |

### Documentos de flujo (SDD + Cursor)

| Ruta | Propósito |
|------|-----------|
| [docs/SDD_PROMPT_TEMPLATES.md](docs/SDD_PROMPT_TEMPLATES.md) | Plantillas de prompt: solo spec, implementación, cierre. |
| [docs/PR_CHECKLIST.md](docs/PR_CHECKLIST.md) | Checklist de PR y orden spec → backend → frontend. |

## Repos hermanos

Coloca este clone **al mismo nivel** que `grip-backend` y `grip-frontend` (por ejemplo `weekly-app/grip-specs`, `weekly-app/grip-backend`, …).

### Workspace multi-root (recomendado)

En Cursor o VS Code: **File → Open Workspace from File…** → abre [`GRIP.code-workspace`](GRIP.code-workspace). Así quedan indexados los tres repos en una sola ventana y puedes citar specs y código sin cambiar de proyecto. El workspace define exclusiones de búsqueda y de file watcher para `node_modules`, entornos virtuales Python (`venv`/`.venv`), `__pycache__`, `dist` y `.angular`, para aligerar indexación.

### Citas `@` en Cursor

En chat o Composer, usa **@** para adjuntar archivos de este repo (por ejemplo `@specs/technical-spec.md`, `@specs/api-contract-ssot.md`, `@specs/features/mi-feature.md`). Eso carga el contrato y las reglas R1–R6 **antes** de pedir implementación en `grip-backend` o `grip-frontend`, en línea con el flujo SDD.

En clones locales con los tres repos al mismo nivel, la raíz de `grip-backend` y `grip-frontend` incluye un `AGENTS.md` con enlaces al SSOT por si trabajas con una sola carpeta abierta en el IDE.

## Publicar en GitHub

Remoto publicado: [github.com/jvargas0324/grip-specs](https://github.com/jvargas0324/grip-specs) (rama `main`).

```bash
git clone https://github.com/jvargas0324/grip-specs.git grip-specs
```

## Flujo SDD

1. Cambios de producto/contrato → edita primero los `.md` en `specs/`.
2. Implementa en `grip-backend` / `grip-frontend`.
3. Actualiza `backend_audit_report.md` cuando cambien endpoints o reglas R*.

Plantillas de prompt y checklist de PR: [docs/SDD_PROMPT_TEMPLATES.md](docs/SDD_PROMPT_TEMPLATES.md), [docs/PR_CHECKLIST.md](docs/PR_CHECKLIST.md).
