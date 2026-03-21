# ui-board-refresh-v1

> Feature de **solo presentación**: sistema visual corporativo (tokens, tipografía, tema claro/oscuro) aplicado a todas las vistas del SPA. Referencia de diseño: frames corporativos “Weekly Reports” (light / dark); colores y fuentes alineados a guía de marca GRIP.

## 1. Objetivo

- **Aplicación(es):** **grip-frontend** únicamente.
- **Problema o necesidad:** Unificar aspecto con la guía corporativa (Figma de marca), mejorar legibilidad y ofrecer **dark mode** y **light mode** coherentes en todo el producto.
- **Alcance:** Layout principal (`MainLayoutComponent`), `/dashboard`, `/checklist`, `/weekly`, `/ask-grip`. **Sin** cambios en `grip-backend`, contratos HTTP, ni reglas R1–R6. **Sin** cambios de copy de producto salvo textos puramente de UI (p. ej. etiqueta del toggle de tema).

## 2. Referencia a specs base

- [functional-spec.md](../functional-spec.md): journeys y roles sin cambio funcional.
- [technical-spec.md §6](../technical-spec.md): rutas y componentes existentes; esta feature añade **sistema visual** documentado aquí.
- Reglas R1–R6: **no se modifican**; los badges semánticos (éxito/alerta) mantienen significado (p. ej. verificado / evidencia).

## 3. Actores y permisos

- Todos los roles que usan el SPA (jz, gv, regional, coo, ceo): mismos permisos que hoy; solo cambia la capa visual.

## 4. Flujos principales

1. El usuario abre cualquier ruta del shell: ve layout, sidebar y contenido con tokens corporativos.
2. El usuario alterna **tema claro/oscuro** desde el header: la preferencia persiste entre sesiones (`localStorage`).
3. Primera visita sin preferencia guardada: se respeta `prefers-color-scheme` del sistema.

## 5. Reglas de negocio

1. Ninguna regla de negocio nueva.
2. Los estados de hallazgos y validación R1 en formularios permanecen iguales; solo estilos (colores/spacing) se adaptan al tema.

## 6. Contrato API

- **Sin cambios.** Ningún endpoint nuevo o modificado.

## 7. Cambios en UI (frontend)

- **Tailwind:** `darkMode: 'class'`; colores semánticos `grip-*` basados en variables CSS; fuentes **Inter** (sans) y **JetBrains Mono** (mono, p. ej. metadatos o bloques técnicos).
- **`ThemeService`:** clase `dark` en `document.documentElement`, persistencia `grip-theme`, init vía `APP_INITIALIZER`.
- **Componentes:** `main-layout`, `dashboard-jz`, `checklist-form`, `weekly-summary`, `ask-grip`: reemplazo de `slate-*` / `blue-*` por tokens; burbujas de chat y CTAs usan acento corporativo (naranja) para primario; estados éxito/error conservan semántica verde/rojo con variantes `dark:`.
- **Logo:** marca “G” / “GRIP” en acento hasta disponer de SVG oficial (pendiente equipo marca).

### Tabla de tokens (SSOT visual)

| Token (CSS / Tailwind) | Light | Dark | Uso |
|------------------------|-------|------|-----|
| `--grip-page` / `bg-grip-page` | `#E5E5E1` | `#1A1C2C` | Fondo página principal |
| `--grip-surface` | `#F4F4F1` | `#242736` | Paneles, filtros |
| `--grip-surface-elevated` | `#FFFFFF` | `#2E3248` | Cards, header bar, inputs |
| `--grip-border` | `#C8C8C2` | `#3D4159` | Bordes y divisores |
| `--grip-text` | `#1A1A18` | `#F4F4F5` | Texto principal |
| `--grip-text-muted` | `#5C5C56` | `#A1A1AA` | Labels secundarios |
| `--grip-accent` | `#EA580C` | `#F97316` | Marca, CTA primario, nav activo |
| `--grip-sidebar-*` | Fondo carbón / texto claro | Navy más profundo | Sidebar (contraste con contenido) |

*Nota:* Los hex deben alinearse con variables del Figma corporativo cuando estén publicadas; esta tabla es la referencia implementada en código.

### Tipografía

- **Sans (UI):** Inter — títulos, navegación, formularios.
- **Mono:** JetBrains Mono — opcional para subtítulos técnicos, timestamps o metadata (uso discreto).

## 8. Criterios de aceptación

- Dado el usuario en **cualquier** ruta (`/dashboard`, `/checklist`, `/weekly`, `/ask-grip`), cuando cambia el tema, entonces **toda** la superficie visible (sidebar, header, contenido) refleja light o dark sin texto ilegible.
- Dado un usuario nuevo sin `localStorage`, cuando el sistema prefiere dark, entonces la primera pintura sigue el esquema oscuro (tras init).
- Dado el toggle de tema, cuando el usuario recarga la página, entonces se mantiene la última elección.
- Dado el formulario de resolución de hallazgos, cuando `status === verified` sin evidencia, entonces el botón sigue deshabilitado (R1) y los mensajes de error siguen siendo perceptibles en ambos modos.
- `npm run build` en `grip-frontend` completa sin errores.

## 9. Notas

- Referencias visuales exportadas: imágenes light/dark en el workspace de diseño (frames “Weekly Reports”).
- **Pendiente:** asset SVG/logo oficial si la marca lo exige pixel-perfect.
