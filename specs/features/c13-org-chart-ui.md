# C-13 — Org Chart UI

- **Estado:** borrador
- **Prioridad:** Core MVP
- **Depende de:** B-02 (reports_to hierarchy)
- **SDD referencia:** §5.1.3 (EMP-05 cascada visibilidad), §5.1.2 (reports_to), BR-EMP-02 (acyclic tree)
- **Usa IA/Gemini:** No
- **Ultima actualizacion:** 2026-03-29

---

## 1. Objetivo

- **Aplicacion(es):** grip-frontend
- **Problema:** No existe una visualizacion de la estructura organizacional. Los usuarios necesitan entender la jerarquia (quien reporta a quien) para contextualizar datos y navegar el sistema. La informacion de jerarquia existe en backend (B-02) pero no tiene representacion visual.
- **Alcance:**
  - **grip-backend:** No requiere cambios. Los endpoints ya existen (B-02): `/employees/{id}/tree`, `/employees/{id}/ancestors`, `/employees/{id}/reports`.
  - **grip-frontend:** Visualizacion de arbol organizacional (desktop) y lista colapsable (mobile). Respeta visibilidad jerarquica (EMP-05).
  - **Fuera de alcance:** Edicion de la jerarquia desde el org chart (se hace via admin de empleados), drag-and-drop para reorganizacion (post-MVP).

## 2. Referencia a specs base

- Modelo de jerarquia: `GRIP-SDD-Functional-Specs.md` §5.1.2 (reports_to)
- Cascada de visibilidad: `GRIP-SDD-Functional-Specs.md` §5.1.3 (EMP-05)
- Restriccion de arbol aciclico: `GRIP-SDD-Functional-Specs.md` §5.1 (BR-EMP-02)

## 3. Actores y permisos

| Rol | Permiso |
|-----|---------|
| JEFE_ZONA | Ver su nodo y subordinados directos |
| GTE_VENTAS | Ver su nodo, sus JZ y subordinados de sus JZ |
| GERENTE_REGIONAL | Ver todo el arbol de su region |
| COO | Ver todo el arbol operativo |
| CEO | Ver el arbol completo de la organizacion |

La visibilidad se controla por la cascada jerarquica existente (EMP-05): cada usuario ve su nodo y todo el sub-arbol debajo de el.

## 4. Flujos principales

### 4.1 Visualizar org chart (desktop >= 768px)

1. Usuario navega a "Organigrama" desde el menu principal.
2. Frontend llama `GET /employees/{user_id}/tree` para obtener el sub-arbol visible.
3. Frontend renderiza arbol jerarquico con nodos expandibles/colapsables.
4. Cada nodo muestra: nombre, rol, cantidad de reportes directos.
5. Click en un nodo navega al perfil del empleado.

### 4.2 Visualizar org chart (mobile < 768px)

1. En pantallas < 768px, el arbol se renderiza como lista colapsable (accordion).
2. Cada nivel de la jerarquia es un grupo colapsable.
3. El nodo raiz (el propio usuario) esta expandido por defecto.
4. Tap en un empleado muestra opciones: "Ver perfil", "Ver subordinados".

### 4.3 Navegar hacia arriba (ancestors)

1. Usuario puede navegar "hacia arriba" en la jerarquia usando `GET /employees/{id}/ancestors`.
2. Se muestran los ancestros hasta el nivel mas alto visible segun permisos.
3. Util para entender la cadena de reporte.

### 4.4 Ver reportes directos

1. Click/tap en "Ver reportes" de un nodo llama `GET /employees/{id}/reports`.
2. Se expande el nodo mostrando los reportes directos con nombre y rol.

## 5. Reglas de negocio

| ID | Regla |
|----|-------|
| EMP-05 | Cascada de visibilidad: cada usuario ve su nodo y todo el sub-arbol debajo. No puede ver nodos fuera de su rama jerarquica. |
| BR-EMP-02 | La jerarquia es un arbol aciclico. Un empleado tiene exactamente un `reports_to` (o null para CEO). No se permiten ciclos. |

## 6. Contrato API

**No se requieren endpoints nuevos.** Se consumen los endpoints existentes de B-02:

| Metodo | Endpoint | Descripcion |
|--------|----------|-------------|
| GET | `/employees/{id}/tree` | Sub-arbol completo del empleado |
| GET | `/employees/{id}/ancestors` | Cadena de ancestros hasta la raiz |
| GET | `/employees/{id}/reports` | Reportes directos del empleado |

## 7. Cambios en UI (frontend)

- **Pantalla "Organigrama"** (`/org-chart`):
  - **Desktop (>= 768px):** Componente de arbol jerarquico con nodos expandibles.
    - Nodo: avatar (placeholder), nombre, rol, badge con cantidad de reportes.
    - Lineas de conexion entre nodos padre-hijo.
    - Controles: expandir/colapsar todo.
  - **Mobile (< 768px):** Lista accordion con niveles indentados.
    - Cada item: nombre, rol, icono de expandir si tiene subordinados.
    - Navegacion a perfil via tap.
  - **Breadcrumb de ancestros:** muestra la cadena de reporte del nodo seleccionado.
- **Componentes nuevos:**
  - `OrgTreeComponent` (desktop tree view).
  - `OrgListComponent` (mobile list view).
  - `OrgNodeComponent` (nodo reutilizable).
- **Ruta:** `/org-chart` (standalone, accesible desde menu lateral para todos los roles).
- **Guard:** Autenticacion requerida. No se necesita guard de rol adicional; la visibilidad se controla en backend via EMP-05.

## 8. Criterios de aceptacion

- [ ] **Dado** un GTE_VENTAS, **cuando** accede al organigrama, **entonces** ve su nodo, sus JZ y los subordinados de cada JZ.
- [ ] **Dado** un JZ, **cuando** accede al organigrama, **entonces** ve solo su nodo y sus subordinados directos.
- [ ] **Dado** un CEO, **cuando** accede al organigrama, **entonces** ve el arbol completo de la organizacion.
- [ ] **Dado** un usuario en desktop (>= 768px), **cuando** visualiza el organigrama, **entonces** se renderiza como arbol con nodos expandibles y lineas de conexion.
- [ ] **Dado** un usuario en mobile (< 768px), **cuando** visualiza el organigrama, **entonces** se renderiza como lista accordion colapsable.
- [ ] **Dado** un nodo del organigrama, **cuando** el usuario hace click/tap, **entonces** puede navegar al perfil del empleado.
- [ ] **Dado** cualquier nodo, **cuando** se selecciona "Ver ancestros", **entonces** se muestra la cadena de reporte hasta el nivel visible mas alto.
- [ ] La jerarquia respeta BR-EMP-02 (arbol aciclico) y EMP-05 (cascada de visibilidad).

## 9. Notas

- Este es el C-item mas simple. Solo depende de B-02 (ya implementado) y es puramente frontend.
- No requiere Gemini ni logica de negocio compleja en backend.
- Se puede implementar en cualquier momento despues de B-02.
- Considerar libreria ligera de visualizacion de arboles para Angular (CSS-only tree layout o libreria como ngx-org-chart). Evitar D3.js completo por overhead excesivo para este caso.
