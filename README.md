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

## Repos hermanos

Coloca este clone **al mismo nivel** que `grip-backend` y `grip-frontend` (por ejemplo dentro de `weekly-app/`). Abre el workspace `../GRIP.code-workspace` en Cursor/VS Code para trabajar los tres a la vez.

## Publicar en GitHub

```bash
cd grip-specs
git remote add origin https://github.com/<tu-usuario>/grip-specs.git
git push -u origin main
```

(Ajusta `main`/`dev` según tu convención.)

## Flujo SDD

1. Cambios de producto/contrato → edita primero los `.md` en `specs/`.
2. Implementa en `grip-backend` / `grip-frontend`.
3. Actualiza `backend_audit_report.md` cuando cambien endpoints o reglas R*.
