# GRIP Platform - Software Design Document (SDD)
## Functional Specification - Specs Driven Development

| Metadata | Value |
|---|---|
| **Version** | 1.0.0 |
| **Date** | 2026-03-29 |
| **Status** | ✅ Approved — Source of Truth |
| **Author** | Javier Vargas — AI-Assisted (Claude) |
| **Methodology** | Specs Driven Development (SDD) |
| **Approved by** | Javier Vargas |
| **Next phase** | Fase B — Auditoría de implementación existente |

---

## Table of Contents

1. [Vision & Problem Statement](#1-vision--problem-statement)
   - 1.5 [System Scope & Boundaries](#15-system-scope--boundaries)
2. [System Glossary](#2-system-glossary)
3. [User Roles & Personas](#3-user-roles--personas)
4. [Architectural Overview](#4-architectural-overview)
   - 4.4 [Flujo Funcional Bottom-to-Top](#44-flujo-funcional-bottom-to-top)
5. [Module Specifications](#5-module-specifications)
   - 5.1 [Employee Database (Hub Central)](#51-employee-database-hub-central)
   - 5.2 [Weekly Reports](#52-weekly-reports)
   - 5.3 [Store Visit Checklist](#53-store-visit-checklist)
   - 5.4 [36-Point Annual Review](#54-36-point-annual-review)
   - 5.5 [DA Module (Dienstaufsicht)](#55-da-module-dienstaufsicht)
   - 5.6 [Follow-Up System (Core)](#56-follow-up-system-core)
   - 5.7 [Dashboards (Role-Based)](#57-dashboards-role-based)
   - 5.8 [AI Assistant Layer](#58-ai-assistant-layer)
6. [Data Model](#6-data-model)
7. [Integration Specifications](#7-integration-specifications)
8. [Non-Functional Requirements](#8-non-functional-requirements)
9. [Configuration & Multi-Tenancy](#9-configuration--multi-tenancy)
10. [Phasing Strategy](#10-phasing-strategy)
11. [Acceptance Criteria](#11-acceptance-criteria)

---

## 1. Vision & Problem Statement

### 1.1 Vision

GRIP (Gestao e Revisao Integrada de Performance) es una plataforma de control operacional diseñada para retail hard discount. Su propósito es digitalizar, unificar y potenciar con inteligencia artificial los mecanismos de control que garantizan la ejecución operativa en tienda.

### 1.2 Problem Statement

En operaciones de retail hard discount, los mecanismos de control (visitas a tienda, reportes semanales, revisiones de desempeño, auditorías) se ejecutan en documentos aislados (Word, Excel, email). Esto genera:

- **Hallazgos olvidados**: findings que nunca se resuelven porque no existe un sistema de seguimiento centralizado.
- **Pérdida de contexto histórico**: cada visita/revisión parte de cero sin acceso a datos previos.
- **Imposibilidad de detectar patrones**: sin datos estructurados, los patrones sistémicos son invisibles.
- **Evaluaciones subjetivas**: las revisiones de desempeño carecen de evidencia acumulada.
- **Tiempo desperdiciado**: los managers invierten tiempo excesivo en preparación manual de visitas y reportes.

### 1.3 Key Metric

> **Target: 0 hallazgos olvidados (forgotten findings)**

Todo finding que ingresa al sistema debe tener un propietario, una fecha límite, y ser rastreado hasta su cierre con evidencia.

### 1.4 Design Principles

| Principio | Regla |
|---|---|
| **SIMPLICITY** | Fewer features, not more |
| **SPEED** | Tap, not type |
| **TRANSPARENCY** | See, don't block |
| **TRUST** | Enable decisions, don't require approvals |

### 1.5 System Scope & Boundaries

> **GRIP es una plataforma de control operacional — no un ERP, no un BI, no un HRIS.**

#### Cobertura por Rol

```
ROL                 COBERTURA GRIP    HERRAMIENTAS EXTERNAS QUE PERSISTEN
──────────────────  ───────────────   ──────────────────────────────────────────
Jefe de Tienda      ░░░░░░░░░░  10%   KARDEX, WhatsApp
Jefe de Zona        ████████░░  80%   WhatsApp (comunicación informal)
Gte. Ventas/Log.    ███████░░░  70%   Excel (finanzas), PowerPoint
Gte. Regional       ██████░░░░  60%   HR system, BI/ERP, PowerPoint, inmobiliaria
COO                 █████░░░░░  50%   ERP/BI, PowerPoint (board), RRHH ejecutivo
CEO                 ████░░░░░░  40%   ERP/BI, board tools, estrategia
CFO / CTO / CPO     █░░░░░░░░░  10%   Sus propios sistemas funcionales
```

#### Lo que GRIP sí reemplaza

| Herramienta actual | Reemplazada por |
|---|---|
| Papel / cuadernos | Store Visit Checklist Module |
| Word / PDF impreso para visitas | Checklist digital mobile-first |
| Excel para tracking de hallazgos | Follow-Up System (auto-surface) |
| Word / Excel / Email para reportes semanales | Weekly Report Module |
| Email como mecanismo de seguimiento de DA | DA Module con SLA y ciclo de vida |
| Memoria humana para historial de desempeño | Employee Hub + 3-Point Review history |
| Preparación manual de visitas y revisiones | AI preparation brief |

#### Lo que GRIP no reemplaza

| Herramienta / Sistema | Razón |
|---|---|
| **KARDEX** (ERP / inventario) | GRIP consume sus datos vía integración — no los replica |
| Sistemas de RRHH / nómina | Fuera de scope funcional |
| Herramientas de BI / finanzas | Fuera de scope — GRIP no es una herramienta analítica financiera |
| PowerPoint para presentaciones ejecutivas | GRIP genera síntesis AI, pero la presentación formal es externa |
| WhatsApp / email (comunicación informal) | GRIP notifica, pero no reemplaza la comunicación entre personas |
| Sistemas de expansión / inmobiliaria | Dominios especializados fuera de scope |

#### Integración externa requerida en MVP

| Sistema | Tipo | Dirección | Datos |
|---|---|---|---|
| **KARDEX** | ERP / Inventario | Read-only → GRIP | Ventas, inventario, merma, cash por tienda |

> En MVP, KARDEX es la **única** integración externa. Todas las demás fuentes de datos son generadas dentro de GRIP por los propios usuarios.

---

## 2. System Glossary

| Concepto | Definicion | Contexto |
|---|---|---|
| **CEO** | Chief Executive Officer | Maximo nivel. Ejecuta DA sobre todas las areas (Operations, Finance, IT, etc.). Visibilidad total. |
| **COO** | Chief Operations Officer | Ejecuta DA exclusivamente sobre Operaciones (Tiendas + Logistica). Preside Comite Operativo. Supervisa Gerentes Regionales. |
| **CFO** | Chief Financial Officer | Directores Nacionales de Contabilidad y Tesoreria. Asiste al Comite Operativo. |
| **CTO** | Chief Technology Officer | Infra y Operaciones, Desarrollo de Software, I+D+I. |
| **CPO** | Chief Product Officer | Gerentes de Categoria y Producto. Preside Comite de Categoria y Producto. |
| **Gerente Regional** | Regional Manager | Responsable de una region completa. Supervisa multiples areas funcionales (Ventas, Logistica, Admin, IT, Expansion, Inmobiliaria, Supply Chain). Nivel corporativo-operativo. |
| **Gerente de Ventas** | Sales Manager (Area Manager) | Reporta al Gerente Regional. Supervisa directamente a los Jefes de Zona. Capa intermedia entre regional y operacion de tiendas. |
| **Gerente Logistico** | Logistics Manager (Area Manager) | Reporta al Gerente Regional. Supervisa Jefes de Recibo, Picking y Despacho. Parte del scope MVP. |
| **Jefe de Zona** | District Manager (DM) | Responsable de ~7 tiendas. Ejecuta visitas, checklists, weekly reports. Reporta al Gerente de Ventas. |
| **Jefe de Tienda** | Store Manager | Responsable de una tienda. Acompana al Jefe de Zona en checklists. Reporta al Jefe de Zona. |
| **Store** | Tienda | Unidad operativa basica. Identificada por codigo (ej: CACIQUE T002). |
| **Finding** | Hallazgo | Observacion durante checklist, DA, o cualquier mecanismo de control. Tiene estado, owner y deadline. |
| **Follow-Up Item** | Item de seguimiento | Finding que requiere accion correctiva. Lifecycle: Open > In Progress > Resolved > Verified. |
| **3-Point Review** | Revision mensual de 3 aspectos | Mecanismo donde cada manager (desde Gerente de Ventas hacia arriba) selecciona 3 de 36 aspectos para revisar con cada reporte directo. |
| **36-Point** | Biblioteca de 36 aspectos | Catalogo configurable de competencias/aspectos a evaluar anualmente. |
| **DA** | Dienstaufsicht / Auditoria Directiva | Revision periodica. CEO ejecuta sobre todas las areas; COO ejecuta solo sobre Operaciones. Genera findings estructurados. |
| **Weekly Report** | Reporte semanal | Reporte por nivel jerarquico. Fluye hacia arriba (cascade up). Cada nivel agrega perspectiva. |
| **Checklist** | Lista de verificacion de tienda | Formato con items fijos (definidos por HQ) + items variables (agregados por DM o sugeridos por AI/SM). |
| **Aspect** | Aspecto evaluable | Unidad dentro de la biblioteca de 36 puntos. Configurable por empresa. |
| **KARDEX** | Sistema de inventario/ERP | Fuente externa de KPIs operacionales (ventas, inventario, merma, cash). |
| **Tenant** | Empresa/Operador | Entidad organizacional top-level en multi-tenancy. |
| **Comite Operativo** | Operations Committee | Presidido por COO. Asisten CFO y areas operativas. Espacio de decision cross-funcional. |
| **Comite de Cat y Producto** | Category & Product Committee | Presidido por CPO. Asisten Gerentes Regionales y areas relevantes. |

---

## 3. User Roles & Personas

### 3.1 Role Hierarchy

#### 3.1.1 GRIP Brief (Generic Model)

```
CEO/COO
  └── VP
       └── SM (Senior Manager)
            └── DM (District Manager)
                 └── Store Manager
```

#### 3.1.2 Tiendas Ritmo (Real - First Client)

```
                         CEO
          ┌──────┬───────┼────────┬──────┐
         COO    CFO     CTO     CPO    ...
          |
    Gerentes Regionales ─────────────────── Oficinas Regionales
      ├── Gte. Administrativo y Contable
      │     └── Tesoreria, Contabilidad, RRHH → Staff
      ├── Gte. Ventas ◄──────────────────── MVP Scope (Ventas)
      │     ├── Jefe de Zona (DM)
      │     │     ├── Jefe de Tienda
      │     │     └── Staff
      │     └── ...
      ├── Gte. Logistico ◄──────────────── MVP Scope (Logistica)
      │     ├── Jefe de Recibo
      │     │     └── Staff
      │     ├── Jefe de Picking
      │     │     └── Staff
      │     └── Jefe de Despacho
      │           └── Staff
      ├── Gte. de Expansion → Staff
      ├── Gte. de IT → Staff
      ├── Gte. de Inmobiliaria → Staff
      └── Comprador (Supply Chain) → Staff
```

#### 3.1.3 Role Mapping: GRIP Generic → Ritmo Real

| GRIP Generic Role | Ritmo Real Role | Notes |
|---|---|---|
| CEO | CEO | DA sobre todas las areas |
| COO | COO | DA solo sobre Operaciones. Preside Comite Operativo |
| VP | **No existe en Ritmo** | El COO conecta directo con Gerentes Regionales. El sistema debe soportar jerarquias sin este nivel |
| SM (Senior Manager) | Gerente Regional | Pero con scope mas amplio: 7+ areas funcionales, no solo ventas |
| *(no existia)* | **Gerente de Ventas** | Capa real entre Regional y Jefe de Zona. Ejecuta 3-Point Review con Jefes de Zona |
| *(no existia)* | **Gerente Logistico** | Supervisa Recibo, Picking, Despacho. Incluido en MVP |
| DM (District Manager) | Jefe de Zona | Match directo. ~7 tiendas |
| Store Manager | Jefe de Tienda | Match directo |
| *(no existia)* | Jefes Logisticos | Jefe de Recibo, Picking, Despacho — nuevos roles operativos |

#### 3.1.4 Implicaciones de Arquitectura

El sistema NO debe asumir una jerarquia fija de N niveles. Debe soportar:

1. **Jerarquia dinamica basada en `reports_to`**: cualquier profundidad de arbol
2. **Niveles opcionales**: Ritmo no tiene VP; otro cliente podria no tener Gerente de Ventas
3. **Multiples areas funcionales bajo un mismo manager**: el Gerente Regional tiene 7+ reportes de areas distintas
4. **Roles configurables por tenant**: los nombres y permisos de roles se configuran per-tenant, no hard-coded

### 3.2 Role Specifications (Ritmo Context)

#### 3.2.1 Jefe de Zona (DM - District Manager)

| Attribute | Detail |
|---|---|
| **Ritmo title** | Jefe de Zona |
| **Reports to** | Gerente de Ventas |
| **Scope** | ~7 tiendas |
| **Primary device** | Smartphone (in-store), Laptop (office) |
| **Key workflows** | Store Visit Checklist, Weekly Report, Follow-up resolution |
| **Visibility** | Sus tiendas, sus follow-ups, sus reviews |
| **Frequency** | ~2 visitas/semana por tienda, weekly reports |

**Functional needs:**
- `FN-DM-01`: Ejecutar checklist durante visita a tienda (< 10 min)
- `FN-DM-02`: Redactar weekly report con texto libre
- `FN-DM-03`: Ver y resolver follow-up items asignados
- `FN-DM-04`: Recibir preparacion AI antes de visita
- `FN-DM-05`: Capturar fotos como evidencia en checklist
- `FN-DM-06`: Agregar items variables al checklist

**No ejecuta 3-Point Review** (solo recibe como reviewee).

#### 3.2.2 Jefe de Tienda (Store Manager)

| Attribute | Detail |
|---|---|
| **Ritmo title** | Jefe de Tienda |
| **Reports to** | Jefe de Zona |
| **Scope** | 1 tienda |
| **Primary device** | Smartphone / Terminal en tienda |
| **Key workflows** | Acompanar checklist, resolver follow-ups asignados |
| **Visibility** | Su tienda, sus follow-ups |

**Functional needs:**
- `FN-ST-01`: Participar (acompanar) en la ejecucion del checklist
- `FN-ST-02`: Ver y resolver follow-up items asignados a su tienda
- `FN-ST-03`: Ver historico de checklists de su tienda
- `FN-ST-04`: Recibir notificaciones de follow-ups vencidos

#### 3.2.3 Gerente de Ventas (Sales Manager / Area Manager)

| Attribute | Detail |
|---|---|
| **Ritmo title** | Gerente de Ventas |
| **Reports to** | Gerente Regional |
| **Scope** | Todos los Jefes de Zona de su regional |
| **Primary device** | Laptop, Tablet |
| **Key workflows** | 3-Point Review con Jefes de Zona, Follow-up oversight, Weekly report |
| **Visibility** | Todos los Jefes de Zona bajo su mando, todas sus tiendas, follow-ups de su equipo |
| **Frequency** | Monthly 3-Point Reviews, weekly follow-up review, weekly report |

**Functional needs:**
- `FN-GV-01`: Ejecutar 3-Point Review mensual con cada Jefe de Zona
- `FN-GV-02`: Ver dashboard de follow-ups de todos sus Jefes de Zona
- `FN-GV-03`: Leer weekly reports de sus Jefes de Zona
- `FN-GV-04`: Redactar weekly report agregando perspectiva sobre su equipo de ventas
- `FN-GV-05`: Recibir sugerencias AI de aspectos a revisar por Jefe de Zona
- `FN-GV-06`: Generar feedback que alimente los items variables del checklist del Jefe de Zona
- `FN-GV-07`: Escalar follow-ups vencidos

**Es el primer nivel que ejecuta 3-Point Reviews** (como reviewer).

#### 3.2.4 Gerente Logistico (Logistics Manager / Area Manager)

> **Nota arquitectonica:** El Gerente Logistico es un rol **analogo al Gerente de Ventas** pero en el contexto del CEDI (Centro de Distribucion). Ambos son Area Managers que reportan al Gerente Regional, ejecutan 3-Point Reviews con sus reportes directos, escriben weekly reports y gestionan follow-ups. La diferencia es el dominio: Ventas opera sobre tiendas; Logistica opera sobre el CEDI (Recibo, Picking, Despacho). El sistema debe tratar ambos roles de forma simetrica — mismas capacidades, diferente contexto operativo.

| Attribute | Detail |
|---|---|
| **Ritmo title** | Gerente Logistico |
| **Reports to** | Gerente Regional |
| **Scope** | CEDI: Jefes de Recibo, Picking y Despacho |
| **Primary device** | Laptop |
| **Key workflows** | 3-Point Review con jefes logisticos, Follow-up oversight, Weekly report |
| **Visibility** | Sus jefes logisticos, KPIs de CEDI, follow-ups de su equipo |
| **Analogia** | Mismo patron funcional que Gerente de Ventas, diferente dominio |

**Functional needs (espejo de Gte. Ventas en contexto CEDI):**
- `FN-GL-01`: Ejecutar 3-Point Review mensual con cada jefe logistico (Recibo, Picking, Despacho)
- `FN-GL-02`: Ver dashboard de follow-ups de su equipo logistico
- `FN-GL-03`: Leer weekly reports de sus jefes
- `FN-GL-04`: Redactar weekly report sobre operacion logistica (analogo a FN-GV-04)
- `FN-GL-05`: Ver KPIs logisticos del CEDI (tiempos de recibo, picking accuracy, despacho on-time)
- `FN-GL-06`: Recibir findings de DA relevantes a logistica
- `FN-GL-07`: Generar feedback que alimente follow-ups de sus jefes (analogo a FN-GV-06)
- `FN-GL-08`: Escalar follow-ups vencidos (analogo a FN-GV-07)

#### 3.2.5 Gerente Regional (Regional Manager)

| Attribute | Detail |
|---|---|
| **Ritmo title** | Gerente Regional |
| **Reports to** | COO |
| **Scope** | Region completa: multiples areas funcionales (Ventas, Logistica, Admin, IT, Expansion, Inmobiliaria, Supply Chain) |
| **Primary device** | Laptop / Desktop |
| **Key workflows** | 3-Point Review con Area Managers, DA findings review, weekly report consolidado, team performance |
| **Visibility** | Todas las areas funcionales de su region, todas las tiendas, todos los follow-ups |

**Functional needs:**
- `FN-GR-01`: Ejecutar 3-Point Review mensual con cada Area Manager (Gte Ventas, Gte Logistico, etc.)
- `FN-GR-02`: Dashboard consolidado de su region (cross-funcional)
- `FN-GR-03`: Leer weekly reports de todos sus Area Managers
- `FN-GR-04`: Redactar weekly report consolidando la region + perspectiva propia
- `FN-GR-05`: Revisar y actuar sobre DA findings asignados a su region
- `FN-GR-06`: Ver KPIs operacionales consolidados de su region
- `FN-GR-07`: Escalar follow-ups criticos al COO
- `FN-GR-08`: Recibir early warnings de AI sobre tendencias en su region

#### 3.2.6 COO (Chief Operations Officer)

| Attribute | Detail |
|---|---|
| **Ritmo title** | COO |
| **Reports to** | CEO |
| **Scope** | Operaciones completas: todas las regiones, tiendas y logistica |
| **Primary device** | Laptop / Desktop |
| **Key workflows** | DA execution (solo Operaciones), 3-Point Review con Gerentes Regionales, company-wide ops dashboards |
| **Visibility** | Todas las regiones + tiendas + logistica. NO ve areas de CFO, CTO, CPO |

**Functional needs:**
- `FN-COO-01`: Ejecutar DA sobre Operaciones (Tiendas + Logistica)
- `FN-COO-02`: Ejecutar 3-Point Review mensual con cada Gerente Regional
- `FN-COO-03`: Dashboard consolidado de operaciones
- `FN-COO-04`: Recibir weekly reports consolidados de Gerentes Regionales
- `FN-COO-05`: Presidir Comite Operativo (consumir datos cross-funcionales)
- `FN-COO-06`: Ver y gestionar escalaciones criticas de toda la operacion
- `FN-COO-07`: Solicitar performance summaries a la AI

#### 3.2.7 CEO (Chief Executive Officer)

| Attribute | Detail |
|---|---|
| **Ritmo title** | CEO |
| **Reports to** | Board / Shareholders |
| **Scope** | Toda la empresa, todas las areas |
| **Primary device** | Laptop / Desktop |
| **Key workflows** | DA execution (todas las areas), 3-Point Review con C-suite, company-wide dashboards |
| **Visibility** | Total sin restricciones |

**Functional needs:**
- `FN-CEO-01`: Ejecutar DA sobre TODAS las areas (Operations, Finance, IT, Product, etc.)
- `FN-CEO-02`: Ejecutar 3-Point Review mensual con COO (y otros C-level si aplica)
- `FN-CEO-03`: Dashboard consolidado de toda la empresa
- `FN-CEO-04`: Ver DA status y findings por area
- `FN-CEO-05`: Solicitar performance summaries a la AI
- `FN-CEO-06`: Recibir panorama completo semanal via weekly reports cascadeados
- `FN-CEO-07`: Visibilidad de todos los early warnings y patrones detectados

#### 3.2.8 Jefes Logisticos (MVP Scope)

> **Nota arquitectonica:** Los Jefes Logisticos son **analogos al Jefe de Zona** pero en el contexto del CEDI. Asi como el Jefe de Zona reporta al Gerente de Ventas y opera sobre tiendas, los Jefes Logisticos reportan al Gerente Logistico y operan sobre areas del CEDI.

| Ritmo Title | Reports To | Dominio | Analogo a | GRIP Function |
|---|---|---|---|---|
| **Jefe de Recibo** | Gerente Logistico | Recepcion de mercancia en CEDI | Jefe de Zona | Weekly reports, follow-up resolution, recibe 3-Point Review |
| **Jefe de Picking** | Gerente Logistico | Preparacion de pedidos | Jefe de Zona | Weekly reports, follow-up resolution, recibe 3-Point Review |
| **Jefe de Despacho** | Gerente Logistico | Despacho y distribucion | Jefe de Zona | Weekly reports, follow-up resolution, recibe 3-Point Review |

Estos roles NO ejecutan checklists de tienda (su contexto es CEDI, no tienda). Sin embargo, el sistema podria soportar en el futuro un "Checklist de CEDI" configurable con estructura analoga al Store Visit Checklist.

**Functional needs (por cada jefe logistico):**
- `FN-JL-01`: Redactar weekly report sobre su area del CEDI
- `FN-JL-02`: Ver y resolver follow-up items asignados
- `FN-JL-03`: Recibir 3-Point Review mensual de su Gerente Logistico
- `FN-JL-04`: Ver KPIs de su area especifica

#### 3.2.9 System Administrator

| Attribute | Detail |
|---|---|
| **Scope** | Configuracion del tenant |
| **Key workflows** | Configurar aspect library, checklist templates, roles, stores, jerarquia |

**Functional needs:**
- `FN-ADMIN-01`: CRUD de tiendas (codigo, nombre, ubicacion, DM asignado)
- `FN-ADMIN-02`: CRUD de empleados y asignacion de roles/jerarquia
- `FN-ADMIN-03`: Configurar Aspect Library (36-point categories)
- `FN-ADMIN-04`: Configurar checklist template (items fijos por seccion)
- `FN-ADMIN-05`: Configurar frecuencia de DA
- `FN-ADMIN-06`: Gestionar integraciones (KARDEX)
- `FN-ADMIN-07`: Configurar roles y permisos por tenant
- `FN-ADMIN-08`: Configurar estructura organizacional (jerarquia flexible basada en reports_to)

### 3.3 3-Point Review Execution Matrix

Clarificacion de quien ejecuta 3-Point Reviews como **reviewer** vs quien lo **recibe** como reviewee:

| Reviewer (pregunta) | Reviewee (responde) | Frecuencia |
|---|---|---|
| CEO | COO, CFO, CTO, CPO | Mensual, 3 aspectos por C-level |
| COO | Gerentes Regionales | Mensual, 3 aspectos por Regional |
| Gerente Regional | Gte. Ventas, Gte. Logistico, otros Area Managers | Mensual, 3 aspectos por Area Manager |
| Gerente de Ventas | Jefes de Zona | Mensual, 3 aspectos por Jefe de Zona |
| Gerente Logistico | Jefes de Recibo, Picking, Despacho | Mensual, 3 aspectos por Jefe |

**Roles que NO ejecutan 3-Point Review como reviewer:**
- Jefe de Zona (solo recibe)
- Jefe de Tienda (solo recibe/participa)
- Jefes Logisticos (solo reciben)
- Staff (no participa)

### 3.4 DA Execution Matrix

| Executor | Scope de DA | Target (quien recibe el DA) |
|---|---|---|
| CEO | Todas las areas: Operations, Finance, IT, Product, etc. | Gerentes Regionales (y potencialmente otros C-level) |
| COO | Solo Operaciones: Tiendas + Logistica | Gerentes Regionales |

### 3.5 Weekly Report Cascade (Ritmo)

```
Level 5: Jefe de Zona / Jefes Logisticos
          Escriben sobre actividades, issues y resultados de su area
                              |
                              v
Level 4: Gerente de Ventas / Gerente Logistico (Area Managers)
          Agregan reports de sus jefes + perspectiva funcional
                              |
                              v
Level 3: Gerente Regional
          Consolida todas las areas + perspectiva regional
                              |
                              v
Level 2: COO
          Consolida regiones + insights operativos estrategicos
                              |
                              v
Level 1: CEO
          Recibe panorama completo → usa para DA y 3-Point Reviews
```

---

## 4. Architectural Overview

### 4.1 High-Level Architecture

```
+------------------------------------------------------------------+
|                        CLIENT LAYER                               |
|              Responsive Web Application (SPA)                     |
|         Mobile-first | PWA-capable | Tap-oriented UI              |
+------------------------------------------------------------------+
                              |
                         HTTPS / REST + WebSocket
                              |
+------------------------------------------------------------------+
|                        API LAYER                                  |
|                    RESTful API Gateway                             |
|              Auth | Rate Limiting | Tenant Resolution             |
+------------------------------------------------------------------+
                              |
        +---------------------+---------------------+
        |                     |                     |
+---------------+    +----------------+    +------------------+
| CORE MODULES  |    |  AI ENGINE     |    | INTEGRATION      |
|               |    |                |    |                  |
| - Employees   |    | - Preparation  |    | - KARDEX Adapter |
| - Stores      |    | - Patterns     |    | - Export/Import  |
| - Checklists  |    | - Early Warning|    | - Notifications  |
| - Weekly Rpts |    | - Follow-up    |    |                  |
| - 3-Point Rev |    |   Automation   |    |                  |
| - DA          |    | - Performance  |    |                  |
| - Follow-ups  |    |   Synthesis    |    |                  |
| - Dashboards  |    |                |    |                  |
+---------------+    +----------------+    +------------------+
        |                     |                     |
+------------------------------------------------------------------+
|                      DATA LAYER                                   |
|            PostgreSQL (relational) + Object Storage (photos)      |
|                   Tenant-scoped data isolation                    |
+------------------------------------------------------------------+
```

### 4.2 Data Flow Overview

```
         DATA SOURCES                    CONTROL MECHANISMS              OUTPUTS
+-------------------------+        +-------------------------+    +------------------+
| Weekly Reports          |------->|                         |--->| Follow-Up Items  |
| Store Visit Checklists  |------->|    EMPLOYEE DATABASE    |--->| Performance Eval |
| 3-Point Review Records  |------->|       (Hub Central)     |--->| Dashboards       |
| DA Findings             |------->|                         |--->| AI Insights      |
| Operational KPIs        |------->|                         |--->| Early Warnings   |
| (from KARDEX)           |        +-------------------------+    +------------------+
+-------------------------+               |
                                          v
                                   +-------------+
                                   | AI ENGINE   |
                                   | Cross-data  |
                                   | analysis    |
                                   +-------------+
```

### 4.3 Tenant Isolation Strategy

- Cada tenant (empresa) opera en un espacio de datos completamente aislado
- Schema-based isolation en PostgreSQL (un schema por tenant)
- Tenant resolution via subdomain o header en cada request
- Configuraciones (aspect library, checklist templates, DA frequency) son per-tenant

---

### 4.4 Flujo Funcional Bottom-to-Top

> **Propósito:** Esta sección es una referencia conceptual. No define requisitos ni reglas de negocio — su función es proveer el modelo mental completo del sistema antes de entrar a los módulos detallados del Section 5. Al auditar la implementación existente, este flujo sirve como hilo conductor para identificar qué módulo revisar en cada capa.

El flujo describe cómo la información operacional nace en la tienda y sube, capa por capa, hasta el nivel ejecutivo — y cómo las directrices bajan en sentido contrario.

---

#### Capa 0 — La Tienda (Jefe de Tienda)

**Rol:** Jefe de Tienda
**Contexto:** Unidad operativa mínima del sistema. El JT no genera reportes al sistema directamente, pero es el sujeto de observación de todos los mecanismos de control.

**Qué produce hacia el sistema:**
- Acompaña al Jefe de Zona durante el **Store Visit Checklist** → su desempeño en tienda queda registrado
- Recibe **Follow-Up Items** asignados a él (hallazgos pendientes de resolver)
- Sus indicadores operacionales (ventas, merma, inventario) llegan vía **KARDEX**

**Qué recibe desde arriba:**
- Items variables en el checklist (agregados por su JZ o sugeridos por AI)
- Findings asignados con deadline

**Efecto en el sistema:** El JT es el nodo terminal de ejecución. Todo lo que el sistema mide comienza aquí — pero él no opera la plataforma directamente en MVP.

---

#### Capa 1 — Zona (Jefe de Zona)

**Rol:** Jefe de Zona
**Contexto:** Primer operador activo del sistema. Gestiona ~7 tiendas. Es quien alimenta el sistema con datos de campo.

**Qué produce hacia el sistema:**
- **Store Visit Checklist** por tienda visitada → genera findings, photos, items con estado
- **Weekly Report** nivel JZ → resumen de sus tiendas en la semana
- **Follow-Up Items** asignados a JTs o a sí mismo → con owner, deadline, evidencia
- KPIs por tienda tomados de KARDEX → enriquecen su Weekly Report

**Qué recibe del sistema:**
- Superficie automática de **findings no cerrados** de visitas anteriores (auto-surface flags)
- **3-Point Review** recibido de su Gerente de Ventas → 3 aspectos evaluados de su desempeño
- AI suggestions de items variables para su próxima visita basadas en patrones previos

**Efecto en el sistema:**
- Sus checklists son la fuente primaria de findings operativos
- Sus Weekly Reports son la base del cascade semanal hacia arriba
- Sus follow-ups pendientes resurgen automáticamente en la siguiente visita al mismo store

---

#### Capa 2 — Área (Gerente de Ventas / Gerente Logístico)

**Rol:** Gerente de Ventas (contexto tiendas) | Gerente Logístico (contexto CEDI)
**Contexto:** Capa intermedia entre la operación de campo y la dirección regional. Gestiona múltiples Jefes de Zona (Ventas) o Jefes de Recibo/Picking/Despacho (Logística).

**Qué produce hacia el sistema:**
- **Weekly Report** nivel Área → agrega los reportes de sus JZs, añade perspectiva de zona, identifica tendencias
- **3-Point Review** ejecutado sobre cada uno de sus JZs → 3 aspectos seleccionados de los 36, con calificación y comentario
- **Follow-Up Items** de nivel área → hallazgos que superan la capacidad de resolución del JZ
- Recibe un **3-Point Review** de su Gerente Regional → es evaluado como sujeto

**Qué recibe del sistema:**
- Dashboard consolidado de sus JZs con KPIs, follow-ups pendientes, scores de checklist
- Superficie automática de findings que sus JZs no cerraron → escalados a él
- AI preparation brief antes de cada 3-Point Review con sus JZs (histórico de aspectos, patrones, comparativa peers)

**Efecto en el sistema:**
- Es el primer nivel que ejecuta 3-Point Reviews → el ciclo de evaluación mensual nace aquí
- Sus Weekly Reports son el primer nivel de síntesis real (no solo datos de campo)
- Los findings escalados desde sus JZs son ahora de su responsabilidad en el Follow-Up System

---

#### Capa 3 — Regional (Gerente Regional)

**Rol:** Gerente Regional
**Contexto:** Nivel corporativo-operativo. Responsable de una región completa con múltiples áreas funcionales (Ventas, Logística, Admin, IT, Expansión, Inmobiliaria, Supply Chain).

**Qué produce hacia el sistema:**
- **Weekly Report** nivel Regional → agrega reportes de sus Gerentes de Área, identifica patrones regionales, propone decisiones cross-funcionales
- **3-Point Review** ejecutado sobre cada uno de sus Gerentes de Área → evaluación mensual con 3 aspectos seleccionados
- Es el **sujeto receptor de DAs** ejecutadas por COO o CEO → sus findings quedan vinculados a su perfil
- **Follow-Up Items** de nivel regional → hallazgos sistémicos que requieren acción a nivel región

**Qué recibe del sistema:**
- Dashboard regional: KPIs consolidados de todas las tiendas bajo su región, heatmaps de desempeño, follow-ups vencidos
- **DA findings** asignados a él o a su equipo con deadline y seguimiento
- AI early warning: alertas de degradación en tiendas/zonas antes de que se conviertan en crisis
- 3-Point Review recibido del COO (o CEO) → es evaluado como sujeto

**Efecto en el sistema:**
- Sus reportes son el último nivel de síntesis antes de llegar al C-suite
- Los DA findings que recibe crean follow-ups formales con escalada automática si no se cierran
- Su 3-Point Review hacia sus Gerentes de Área completa el ciclo de evaluación mensual de la capa media

---

#### Capa 4 — Ejecutivo (COO / CEO)

**Rol:** COO (operaciones), CEO (todas las áreas)
**Contexto:** Nivel de decisión estratégica. No operan el sistema diariamente — lo consumen para decidir y ejecutar auditorías directivas puntuales.

**Qué produce hacia el sistema:**
- **DA (Dienstaufsicht)** → el mecanismo de control más poderoso del sistema:
  - CEO: scope ALL_AREAS → puede auditar cualquier área funcional, cualquier Gerente Regional
  - COO: scope OPERATIONS_ONLY → audita Ventas + Logística, dirigido a Gerentes Regionales
  - Genera findings formales con owner asignado, deadline, y seguimiento automático
- **3-Point Review** ejecutado sobre sus reportes directos (Gerentes Regionales para COO; todos los C-suite para CEO)

**Qué recibe del sistema:**
- Dashboard ejecutivo: KPIs top-line, follow-ups críticos vencidos, tendencias por región, alertas AI
- Weekly Reports consolidados de todas las regiones (cascade final)
- AI synthesis: resúmenes ejecutivos pre-Comité Operativo, patrones sistémicos, comparativa cross-regional
- Historial de DAs previas con tasa de cierre de findings por Gerente Regional

**Efecto en el sistema:**
- Una DA activa el mecanismo de seguimiento más exigente del sistema — cada finding tiene SLA
- Los findings de DA resurgen automáticamente en la siguiente DA al mismo Gerente Regional (feedback loop)
- Una DA puede generar items variables en los próximos checklists de tiendas afectadas (cross-module cascade)

---

#### Resumen del Flujo Bidireccional

```
         BOTTOM                                                    TOP
+------------------+    +------------------+    +------------------+    +------------------+
|   JEFE TIENDA    |    |   JEFE DE ZONA   |    |   GTE. VENTAS /  |    | GERENTE REGIONAL |
|                  |    |                  |    |   GTE. LOGÍSTICO |    |                  |
| - KPIs (KARDEX)  |--->| - Checklist      |--->| - Weekly Report  |--->| - Weekly Report  |
| - Acompaña visita|    | - Weekly Report  |    |   (Área)         |    |   (Regional)     |
| - Findings owner |    | - Follow-ups     |    | - 3-Point Review |    | - 3-Point Review |
|                  |    | - KPIs tienda    |    |   (ejecuta→JZ)   |    |   (ejecuta→Área) |
+------------------+    +------------------+    | - Follow-ups     |    | - DA sujeto      |
         ^                       ^              |   escalados      |    | - Follow-ups     |
         |                       |              +------------------+    +------------------+
         |                       |                       ^                       ^
    [Items variables         [Auto-surface              [AI prep brief]      [DA findings]
     desde JZ/AI]             findings]                 [3-Point recibido]   [3-Point recibido]
                                                                                    |
                                                                                    v
                                                                         +------------------+
                                                                         |    COO / CEO     |
                                                                         |                  |
                                                                         | - DA (ejecuta)   |
                                                                         | - Weekly Report  |
                                                                         |   (top-level)    |
                                                                         | - 3-Point Review |
                                                                         |   (ejecuta)      |
                                                                         | - Dashboard EJ.  |
                                                                         | - AI Synthesis   |
                                                                         +------------------+
```

**Regla fundamental del flujo:**
> Datos operativos suben → síntesis y perspectiva se agregan en cada capa → decisiones y findings bajan → los findings generan follow-ups → los follow-ups resurgen automáticamente hasta cerrarse.

**El sistema no termina cuando se registra un finding — termina cuando el finding tiene evidencia de cierre verificada.**

---

## 5. Module Specifications

### 5.1 Employee Database (Hub Central)

#### 5.1.1 Purpose

Hub central que conecta todos los mecanismos de control con las personas. Cada employee record acumula el historial completo de interacciones: weekly reports escritos, checklists ejecutados, 3-point reviews recibidos, DA findings asociados, follow-ups asignados.

#### 5.1.2 Data Model

```
Role {
  id: UUID
  tenant_id: UUID (FK -> Tenant)
  code: String // e.g., "CEO", "COO", "GERENTE_REGIONAL", "GTE_VENTAS", "JEFE_ZONA", "JEFE_TIENDA"
  name: String // display name, e.g., "Gerente de Ventas"
  hierarchy_level: Integer // 1=CEO, 2=C-suite, 3=Regional, 4=Area Manager, 5=Jefe de Zona, 6=Jefe Tienda
  permissions: JSON // configurable capability flags (see 3.6)
  can_execute_da: Boolean // default false, true for CEO/COO
  can_execute_3point_review: Boolean // true from Area Manager (Gte Ventas) upward
  can_execute_checklist: Boolean // true for Jefe de Zona
  can_write_weekly_report: Boolean // true for all manager levels
  da_scope: Enum [NONE, OPERATIONS_ONLY, ALL_AREAS] // NONE for most, OPS for COO, ALL for CEO
}

Employee {
  id: UUID
  tenant_id: UUID (FK -> Tenant)
  employee_code: String (unique per tenant)
  first_name: String
  last_name: String
  email: String
  role_id: UUID (FK -> Role) // configurable per tenant, not hardcoded enum
  status: Enum [ACTIVE, INACTIVE]
  reports_to: UUID (FK -> Employee) // nullable for CEO. Defines the hierarchy tree
  assigned_stores: Store[] // for roles that manage stores (Jefe de Zona)
  functional_area: String // e.g., "Ventas", "Logistica", "Admin", "IT" (nullable)
  hire_date: Date
  locale: String // e.g., "es-DO", "es-CO"
  timezone: String // e.g., "America/Santo_Domingo"
  created_at: Timestamp
  updated_at: Timestamp
}
```

#### 5.1.3 Functional Requirements

| ID | Requirement | Priority | Acceptance Criteria |
|---|---|---|---|
| `EMP-01` | CRUD de empleados con validacion de unicidad por tenant | Must | Un employee_code no se repite dentro del mismo tenant |
| `EMP-02` | Asignacion de rol jerarquico | Must | Solo se permite un nivel arriba/abajo en la cadena |
| `EMP-03` | Asignacion de tiendas a DM | Must | Una tienda solo puede estar asignada a un DM activo |
| `EMP-04` | Vista de perfil con historial acumulado | Must | El perfil muestra: weekly reports, checklists, reviews, follow-ups |
| `EMP-05` | Cascada de visibilidad por jerarquia | Must | Cada manager ve a sus reportes directos y todos los niveles inferiores de su rama. Gte. Ventas ve sus Jefes de Zona; Gerente Regional ve todo su equipo cross-funcional; COO ve todas las regiones; CEO ve todo |
| `EMP-06` | Desactivacion soft-delete | Should | Empleados inactivos conservan historial pero no aparecen en asignaciones |

#### 5.1.4 Business Rules

- `BR-EMP-01`: Un Jefe de Zona debe tener al menos 1 tienda asignada para poder ejecutar checklists
- `BR-EMP-02`: La relacion `reports_to` debe formar un arbol sin ciclos. No se impone limite de profundidad
- `BR-EMP-03`: Al desactivar un empleado, sus follow-ups abiertos deben reasignarse
- `BR-EMP-04`: Los permisos de ejecucion (DA, 3-Point, checklist, weekly) se derivan del Role configurado, no del nivel jerarquico
- `BR-EMP-05`: Un empleado puede pertenecer a un area funcional (functional_area) que determina el scope de su visibilidad dentro de la region

---

### 5.2 Weekly Reports

#### 5.2.1 Purpose

Reporte semanal de texto libre que cada nivel de la jerarquia escribe sobre su semana operativa. La informacion fluye hacia arriba (cascade up only), y cada nivel agrega su perspectiva:

- **Level 4 - Area Manager (DM)**: Escribe sobre las actividades, issues y resultados de su area (tiendas)
- **Level 3 - Boss/Director (SM)**: Agrega los reports de sus reportes directos + anade su propia perspectiva
- **Level 2 - Vice President**: Consolida reports de area + insights estrategicos para CEO
- **Level 1 - CEO**: Recibe el panorama completo semanal. Lo usa para preparar DA y 3-Point Reviews

#### 5.2.2 Data Model

```
WeeklyReport {
  id: UUID
  tenant_id: UUID
  author_id: UUID (FK -> Employee) // cualquier manager en la jerarquia
  week_number: Integer // ISO week
  year: Integer
  content: Text // texto libre, rich text basico
  status: Enum [DRAFT, SUBMITTED]
  flagged_for_followup: Boolean // marca que requiere atencion
  flagged_by: UUID (FK -> Employee) // quien lo marco
  submitted_at: Timestamp
  created_at: Timestamp
  updated_at: Timestamp
}
```

#### 5.2.3 Functional Requirements

| ID | Requirement | Priority | Acceptance Criteria |
|---|---|---|---|
| `WR-01` | Creacion de weekly report con texto libre por nivel | Must | Cada manager (DM, SM, VP) puede crear un report asociado a una semana ISO especifica |
| `WR-02` | Draft y submit workflow | Must | Report inicia como DRAFT, DM puede editar hasta hacer SUBMIT. Despues de submit es read-only |
| `WR-03` | Cascade visibility (up only) + aggregation per level | Must | Cada nivel ve los reports de sus reportes directos. SM agrega perspectiva sobre reports de DMs; VP consolida area; CEO recibe panorama completo |
| `WR-04` | Flag for follow-up | Must | Cualquier superior puede marcar un report para seguimiento. Esto genera un follow-up item automatico |
| `WR-05` | Historico por autor y semana | Must | Se puede navegar el historial de weeklys de un DM |
| `WR-06` | Un solo report por DM por semana | Must | El sistema impide duplicados por (author_id, week_number, year) |
| `WR-07` | Notificacion al SM cuando DM hace submit | Should | El SM recibe notificacion (in-app) cuando un DM de su equipo envia su weekly |
| `WR-08` | AI signal detection en contenido | Should | La AI analiza el texto buscando senales (ej: mencion de "staffing challenges" correlaciona con findings futuros) |

#### 5.2.4 Business Rules

- `BR-WR-01`: Todos los niveles de la jerarquia (DM, SM, VP) crean weekly reports. Cada nivel agrega su perspectiva sobre los reports de sus reportes directos
- `BR-WR-02`: Un report no puede editarse despues de SUBMITTED
- `BR-WR-03`: El flag_for_followup genera automaticamente un Follow-Up Item vinculado al report

---

### 5.3 Store Visit Checklist

#### 5.3.1 Purpose

Instrumento principal de control en tienda. El DM ejecuta un checklist durante su visita. Contiene items fijos definidos por HQ (template configurable por tenant) e items variables agregados por el DM o sugeridos por AI/SM feedback.

#### 5.3.2 Checklist Template Structure

Basado en el formato real observado, el template se organiza en **secciones** con **items** dentro de cada seccion:

```
ChecklistTemplate {
  id: UUID
  tenant_id: UUID
  name: String // e.g., "Checklist Jefe de Zona"
  version: Integer
  status: Enum [ACTIVE, ARCHIVED]
  sections: ChecklistSection[]
  created_at: Timestamp
  updated_at: Timestamp
}

ChecklistSection {
  id: UUID
  template_id: UUID (FK -> ChecklistTemplate)
  sort_order: Integer
  name: String // e.g., "Exterior", "Limpieza de la tienda"
  items: ChecklistItem[]
}

ChecklistItem {
  id: UUID
  section_id: UUID (FK -> ChecklistSection)
  sort_order: Integer
  description: Text // el criterio a evaluar
  guidance_note: Text // nota orientativa (opcional)
  is_required: Boolean
  item_type: Enum [FIXED, VARIABLE]
}
```

#### 5.3.3 Default Sections (per observed checklist)

El template por defecto contiene 9 secciones. Estas son **configurables por tenant**, pero el sistema provee este template base:

| # | Seccion | Items observados | Naturaleza |
|---|---|---|---|
| 1 | Exterior | Poster productos, Parqueadero, Puertas | Imagen y primera impresion |
| 2 | Limpieza de la tienda | Piso, Gondolas, Estibas, Paredes/techos, Extintor | Higiene y seguridad |
| 3 | Exhibicion de Mercancia | Frenteo/alineacion, Corte de cajas | Visual merchandising |
| 4 | Precios | Etiquetas, Flejes, Descontinuados | Pricing accuracy |
| 5 | Neveras y congeladores | Limpieza, Rotacion PEPS | Cadena de frio |
| 6 | Puesto de pago | Tecleo, Mercancia camuflada, Comprobante fiscal, Precios vs visor, Limpieza scanner | Cash control |
| 7 | Bodega | Marcado por bloque, Apilamiento, Exceso mercancia, Banos, Cocina | Almacenamiento |
| 8 | Documentos tienda | Planilla destruccion, Agotados Odoo, Merma Odoo, Pedidos, Horarios, Tareas, Vencimientos, Bloque | Compliance documental |
| 9 | Empleados | Uniforme, Atencion al cliente | Capital humano |

#### 5.3.4 Checklist Instance (Execution)

```
ChecklistInstance {
  id: UUID
  tenant_id: UUID
  template_id: UUID (FK -> ChecklistTemplate)
  store_id: UUID (FK -> Store)
  executor_id: UUID (FK -> Employee) // el DM
  accompanied_by: String // nombre de quien acompana la visita (ej: "Alexandra")
  visit_date: Date
  visit_time: Time
  status: Enum [IN_PROGRESS, COMPLETED, CANCELLED]
  score_summary: JSON // {total_items, compliant, non_compliant, na}
  started_at: Timestamp
  completed_at: Timestamp
}

ChecklistResponse {
  id: UUID
  instance_id: UUID (FK -> ChecklistInstance)
  item_id: UUID (FK -> ChecklistItem) // null if variable item
  section_name: String // denormalized for variable items
  item_description: Text // denormalized or custom for variable items
  item_type: Enum [FIXED, VARIABLE]
  result: Enum [COMPLIANT, NON_COMPLIANT, NOT_APPLICABLE]
  annotation: Text // observaciones de texto libre
  photos: Photo[] // evidencia fotografica
  generates_followup: Boolean // si el no-cumple genera follow-up
  created_at: Timestamp
}

Photo {
  id: UUID
  response_id: UUID (FK -> ChecklistResponse)
  storage_url: String
  thumbnail_url: String
  captured_at: Timestamp
}
```

#### 5.3.5 Functional Requirements

| ID | Requirement | Priority | Acceptance Criteria |
|---|---|---|---|
| `CL-01` | Iniciar checklist seleccionando tienda | Must | DM selecciona tienda de su lista asignada, sistema carga template activo |
| `CL-02` | Registrar resultado por item (Cumple/No Cumple/N/A) | Must | Interaccion tap-based. Un tap para cumple, otro para no cumple |
| `CL-03` | Agregar anotacion por item | Must | Campo de texto libre por item. Opcional para "Cumple", obligatorio workflow para "No Cumple" |
| `CL-04` | Capturar foto como evidencia | Must | Integrar camara del dispositivo. Multiples fotos por item |
| `CL-05` | Agregar items variables | Must | DM puede agregar items que no estan en el template fijo, asociados a una seccion |
| `CL-06` | Items variables sugeridos por AI | Should | AI sugiere items basados en: feedback del SM, findings previos, patrones detectados |
| `CL-07` | Completar checklist en < 10 min | Must | UI optimizada: scroll vertical continuo, taps grandes, minima escritura |
| `CL-08` | Registrar acompanante de visita | Must | Campo para nombre de la persona con quien se realizo la visita |
| `CL-09` | Score summary automatico | Must | Al completar: total items, cumple, no cumple, % cumplimiento |
| `CL-10` | Generacion automatica de follow-ups | Must | Cada "No Cumple" con anotacion puede generar un follow-up item |
| `CL-11` | Modo offline | Should | Permitir ejecutar checklist sin conexion y sincronizar al reconectar |
| `CL-12` | Historico de checklists por tienda | Must | Navegar todos los checklists ejecutados en una tienda |
| `CL-13` | Comparacion con checklist anterior | Should | Mostrar delta vs ultimo checklist de la misma tienda |

#### 5.3.6 Business Rules

- `BR-CL-01`: Solo un DM con tienda asignada puede ejecutar checklist en esa tienda
- `BR-CL-02`: No se permite mas de un checklist IN_PROGRESS por DM simultaneamente
- `BR-CL-03`: Un checklist COMPLETED no puede editarse. Se debe crear uno nuevo
- `BR-CL-04`: Items variables se persisten en la instancia pero no modifican el template
- `BR-CL-05`: Las fotos se almacenan en object storage con referencia en la DB
- `BR-CL-06`: La anotacion en un "No Cumple" es fuertemente recomendada (warning) pero no bloqueante

#### 5.3.7 UX Specification

```
+------------------------------------------+
|  STORE VISIT CHECKLIST                   |
|  CACIQUE T002 | 17-Feb-2026 | 12:30 PM  |
|  Con: Alexandra                          |
+------------------------------------------+
|                                          |
|  1. EXTERIOR                    3/3  [=] |
|  ----------------------------------------|
|  [x] Poster de productos         [note]  |
|  [ ] Parqueadero                  [note]  |
|  [ ] Puertas                      [note]  |
|                                          |
|  2. LIMPIEZA DE LA TIENDA       5/5  [=] |
|  ----------------------------------------|
|  [x] Piso de venta               [note]  |
|  [x] Gondolas                     [note]  |
|  ...                                      |
|                                          |
|  + Add variable item                      |
|                                          |
|  [    COMPLETE CHECKLIST    ]             |
+------------------------------------------+

Legend: [x] = Cumple (green tap)
        [ ] = No Cumple (red tap)
        [note] = annotation icon (blue if has content)
        [=] = section score
```

---

### 5.4 36-Point Annual Review & 3-Point Monthly Check

#### 5.4.1 Purpose

Sistema de evaluacion progresiva anual. Una biblioteca configurable de aspectos/competencias (default: 36) se revisa a lo largo del ano mediante el mecanismo de **3-Point Review mensual**: cada manager selecciona 3 aspectos por mes para evaluar con cada reporte directo.

**Formula**: 3 aspectos x 12 meses = 36 aspectos por reporte directo por ano.

Este mecanismo es uno de los 3 controles principales de GRIP (junto con Store Visits y DA) y su output alimenta directamente el Follow-Up System, el Employee Profile y la Performance Evaluation de fin de ano.

#### 5.4.2 Quien genera el 3-Point Review

Cada manager con `can_execute_3point_review = true` en su Role genera un 3-Point Review mensual **por cada reporte directo**. En contexto Ritmo, esto aplica desde Gerente de Ventas / Gerente Logistico hacia arriba:

| Reviewer (genera y pregunta) | Reviewee (recibe y responde) | Contexto |
|---|---|---|
| CEO | COO (y otros C-level si aplica) | Revision ejecutiva |
| COO | Gerentes Regionales | Revision operativa por region |
| Gerente Regional | Gte. Ventas, Gte. Logistico, otros Area Managers | Revision cross-funcional de la region |
| Gerente de Ventas | Jefes de Zona | Revision de gestion de tiendas |
| Gerente Logistico | Jefes de Recibo, Picking, Despacho | Revision de gestion de CEDI |

**Roles que NO generan 3-Point Review** (solo lo reciben como reviewees):
- Jefe de Zona
- Jefe de Tienda
- Jefes Logisticos (Recibo, Picking, Despacho)
- Staff

#### 5.4.3 Con base en que se generan: Data Sources

La seleccion de los 3 aspectos mensuales **no es arbitraria** — esta alimentada por 5 fuentes de datos del sistema. El AI Preparation Assistant cruza estas fuentes y sugiere los 3 aspectos optimos. El reviewer puede aceptar la sugerencia o elegir otros.

| Source | Codigo | Que aporta a la seleccion | Ejemplo concreto |
|---|---|---|---|
| **Weekly Reports del reporte directo** | A | Issues mencionados, proyectos con delay → se convierten en topicos de revision | "Maria menciono delays en project milestones en 2 de sus ultimos 4 weeklys" |
| **Previous 3-Point Findings (abiertos)** | B | Issues no resueltos de meses anteriores → re-check obligatorio | "Team communication fue marcado BELOW en agosto, sigue sin resolver" |
| **DA Findings relevantes a la persona** | C | Issues de auditoria directiva que competen al reviewee | "Payment terms con Supplier X: asignado a Maria, deadline vencido, sin updates en 3 semanas" |
| **Operational Data / KPIs** | D | Anomalias, tendencias negativas, bajo desempeno en metricas | "Shrinkage de Store 7 esta 0.3% arriba del promedio del distrito" |
| **Aspect Library (36 categorias)** | E | Aspectos pre-definidos no cubiertos para garantizar cobertura completa anual | "Budget forecasting, Team development, Process compliance no se han revisado — faltan 3 meses" |

```
SOURCES THAT FEED 3-POINT SELECTION
+--------------------------------------------+
| A. Weekly Reports from Direct Report       |
|    Issues mentioned, projects delayed      |     OUTPUT
|    → become check topics                   |     +---------------------------+
+--------------------------------------------+     |                           |
| B. Previous 3-Point Findings (Open)        |     |  MONTHLY 3-POINT CHECK    |
|    Unresolved issues from past months      |---->|  Per direct report         |
|    → re-check                              |     |                           |
+--------------------------------------------+     |  Example Aspects:          |
| C. DA Findings (Relevant to Person)        |     |  - Budget adherence        |
|    Area-level issues this person owns      |     |  - Team development        |
+--------------------------------------------+     |  - Process compliance      |
| D. Operational Data / KPIs                 |     |  - Project milestones      |
|    Anomalies, trends, underperformance     |     |  - Communication quality   |
+--------------------------------------------+     |  - Problem resolution      |
| E. Aspect Library (36 Categories)          |     +---------------------------+
|    Pre-defined aspects for full coverage   |              |
+--------------------------------------------+              v
                                              Findings with action items
                                              → Flow into Follow-Up System
```

#### 5.4.4 A quien se dirige

El 3-Point Review se dirige siempre a **un reporte directo especifico**. Es una conversacion 1-a-1 entre reviewer y reviewee, nunca grupal. En el Employee Database, el vinculo es `Subject = Employee` (el reviewee).

Cada reporte directo tiene su **propio track de cobertura** independiente: el sistema trackea cuales de los 36 aspectos ya fueron cubiertos para ESA persona en ESE ano.

#### 5.4.5 Efectos en cascada sobre el sistema

El 3-Point Review NO es un evento aislado. Cada review completado genera efectos en multiples modulos del sistema:

##### Efecto 1: Follow-Up System (inmediato)

Cada aspecto evaluado con rating `BELOW` o que genere un finding con action item **crea automaticamente un Follow-Up Item** en el sistema:

```
ReviewedAspect (rating = BELOW)
    |
    v
FollowUpItem {
  source_type: THREE_POINT_REVIEW
  source_id: <review_id>
  assigned_to: <reviewee_id>
  assigned_by: <reviewer_id>
  title: "3-Point: <aspect_name>"
  description: <observations del reviewer>
  deadline: <calculado segun config>
  status: OPEN
}
```

##### Efecto 2: Feedback Loop (el mas critico para GRIP)

Los Follow-Up Items generados por un 3-Point Review **resurgen automaticamente** en los proximos mecanismos de control. Este es el loop que garantiza "0 forgotten findings":

| Donde resurge | Como resurge | Cuando |
|---|---|---|
| **Proximo 3-Point Review** de esa persona | Aparece como Source B (Previous 3-Point Findings abiertos) → el reviewer lo ve al preparar el siguiente review | Siguiente mes |
| **Proximo DA** de esa area | Aparece como Source 3 del DA (3-Point Review Findings escalados) | Segun frecuencia DA |
| **Proximo Store Visit** de esa tienda | Aparece como item variable sugerido en el checklist del Jefe de Zona | Proxima visita |
| **Variable checklist suggestions** | La AI lo sugiere como punto de atencion al DM al preparar su visita | Proactivamente |

```
3-POINT REVIEW (completado)
    |
    +---> [Follow-Up Item creado]
    |         |
    |         +---> Resurge en proximo 3-Point Review (Source B)
    |         |         "Maria: Team communication sigue sin resolver"
    |         |
    |         +---> Resurge en proximo Store Visit checklist (variable)
    |         |         AI sugiere: "Verificar team communication en Store 7"
    |         |
    |         +---> Resurge en proximo DA (Source 3, si se escala)
    |         |         "3-Point finding escalado: Team communication"
    |         |
    |         +---> Se trackea hasta resolucion con evidencia
    |                   Owner + Deadline + Status visible para todos
    |
    +---> [Coverage tracking actualizado]
    |         "Maria: 27/36 aspectos cubiertos (75%)"
    |         "Gaps: Conflict resolution, Budget forecasting..."
    |         "Restan 3 meses para cubrir 9 aspectos"
    |
    +---> [Employee Profile enriquecido]
    |         Se acumula en el historial del reviewee
    |         Alimenta Performance Evaluation de fin de ano
    |
    +---> [AI Engine aprende]
    |         Correlaciona: senales en weeklys → resultados en reviews
    |         "Cuando weeklys mencionan 'staffing challenges' 2+ veces,
    |          findings de Team development aparecen 60% de las veces"
    |
    +---> [Notificacion al reviewee]
              Aspectos revisados + ratings + action items asignados
```

##### Efecto 3: Coverage Tracking (acumulativo)

El sistema mantiene un **mapa de cobertura** por cada par (reviewer, reviewee) por ano fiscal:

```
Coverage Map: Maria (reviewed by Juan) - FY2026
+----------------------------------+--------+--------+
| Aspecto                          | Status | Month  |
+----------------------------------+--------+--------+
| A01 Budget adherence             | [x]    | Jan    |
| A02 Team development             | [ ]    | -      | <- GAP
| A03 Process compliance           | [x]    | Feb    |
| A04 Project milestones           | [x]    | Jan    |
| A05 Communication quality        | [ ]    | -      | <- GAP
| A06 Problem resolution           | [x]    | Mar    |
| ...                              |        |        |
| A36 Strategic alignment          | [ ]    | -      | <- GAP
+----------------------------------+--------+--------+
Coverage: 27/36 (75%) | Remaining: 9 | Months left: 3
```

El sistema alerta al reviewer cuando:
- Quedan mas aspectos por cubrir que meses restantes en el ano
- Un aspecto no ha sido revisado en 6+ meses (Coverage Gap alert)
- El coverage pace sugiere que no se completaran los 36 al final del ano

##### Efecto 4: Performance Evaluation (fin de ano)

El brief define que el **36-Point Review History** es la primera fuente del Employee Review Report anual. Los datos acumulados durante el ano alimentan un reporte objetivo:

```
EMPLOYEE REVIEW REPORT (auto-generated)
+--------------------------------------------------+
| 36-Point coverage and results summary            |
| Follow-up performance metrics                    |
| DA ownership record                              |
| Store/area performance (if Ops)                  |
| Strengths identified                             |
| Areas for improvement                            |
| Open items requiring attention                   |
+--------------------------------------------------+
```

El beneficio clave: **conversaciones de performance basadas en hechos, no en opiniones**. El sistema captura la realidad a lo largo del ano.

##### Efecto 5: AI Learning Loop

Cada resolucion de un finding de 3-Point alimenta el AI Engine:
- El sistema aprende que patrones en weekly reports predicen que tipos de findings
- Resoluciones exitosas en un area se sugieren como solucion en areas con problemas similares
- El modelo se hace mas preciso en sus sugerencias de aspectos con cada ciclo completado

#### 5.4.6 Data Model

```
AspectLibrary {
  id: UUID
  tenant_id: UUID
  name: String // e.g., "36-Point Review FY2026"
  fiscal_year: Integer
  total_aspects: Integer // default 36, configurable
  status: Enum [ACTIVE, ARCHIVED]
  aspects: Aspect[]
  created_at: Timestamp
  updated_at: Timestamp
}

Aspect {
  id: UUID
  library_id: UUID (FK -> AspectLibrary)
  code: String // e.g., "A01", "A02"
  name: String // e.g., "Budget forecasting"
  description: Text
  category: String // agrupacion opcional (e.g., "Financial", "People", "Process")
  sort_order: Integer
  example_evidence: Text // guia de que evidencia buscar para este aspecto
}

ThreePointReview {
  id: UUID
  tenant_id: UUID
  reviewer_id: UUID (FK -> Employee) // manager que genera y ejecuta
  reviewee_id: UUID (FK -> Employee) // reporte directo evaluado
  review_month: Integer // 1-12
  review_year: Integer
  status: Enum [DRAFT, COMPLETED]

  // AI preparation context
  ai_suggested_aspects: JSON // los 3 aspectos que AI sugirio (para auditoria)
  ai_suggestion_reasoning: JSON // razonamiento de la AI por cada sugerencia

  aspects_reviewed: ReviewedAspect[]
  general_notes: Text
  completed_at: Timestamp
  created_at: Timestamp
}

ReviewedAspect {
  id: UUID
  review_id: UUID (FK -> ThreePointReview)
  aspect_id: UUID (FK -> Aspect)
  rating: Enum [EXCEEDS, MEETS, BELOW, NOT_EVALUATED]
  observations: Text // notas del reviewer sobre este aspecto

  // Evidence linkage
  evidence_references: JSON {
    weekly_report_ids: UUID[]      // weeklys que soportan la evaluacion
    checklist_ids: UUID[]          // checklists relevantes
    followup_ids: UUID[]           // follow-ups relacionados
    da_finding_ids: UUID[]         // findings de DA vinculados
    kpi_snapshots: JSON            // KPIs al momento de la evaluacion
  }

  // Follow-up generation
  generates_followup: Boolean
  followup_id: UUID (FK -> FollowUpItem) // si genero follow-up, referencia cruzada
}

// Coverage tracking (materialized / computed)
AspectCoverage {
  id: UUID
  tenant_id: UUID
  reviewee_id: UUID (FK -> Employee)
  reviewer_id: UUID (FK -> Employee)
  library_id: UUID (FK -> AspectLibrary)
  fiscal_year: Integer
  aspect_id: UUID (FK -> Aspect)
  covered: Boolean
  covered_in_month: Integer // nullable
  covered_in_review_id: UUID (FK -> ThreePointReview) // nullable
  last_rating: Enum [EXCEEDS, MEETS, BELOW, NOT_EVALUATED]
  times_reviewed_this_year: Integer // puede ser > 1 si se repite
}
```

#### 5.4.7 Functional Requirements

| ID | Requirement | Priority | Acceptance Criteria |
|---|---|---|---|
| `TP-01` | Configurar Aspect Library por tenant | Must | Admin puede CRUD aspectos. Minimo 1, sin limite maximo. Incluye nombre, descripcion, categoria y guia de evidencia |
| `TP-02` | Cada manager selecciona 3 aspectos por review mensual por reporte directo | Must | El sistema muestra los aspectos y su estado de cobertura YTD por cada par reviewer-reviewee |
| `TP-03` | Coverage tracking por reviewee | Must | Dashboard visual: cuales aspectos ya fueron cubiertos, cuales faltan, cuantos meses restan. Alerta cuando pace < required |
| `TP-04` | AI sugiere 3 aspectos basado en 5 fuentes (A-E) | Must | AI cruza las 5 fuentes y sugiere con razonamiento. Sugerencia queda registrada para auditoria |
| `TP-05` | Rating por aspecto | Must | EXCEEDS / MEETS / BELOW / NOT_EVALUATED |
| `TP-06` | Vinculacion de evidencia multi-source | Must | El reviewer puede vincular weeklys, checklists, follow-ups, DA findings y KPI snapshots como evidencia |
| `TP-07` | Generacion automatica de follow-up desde aspecto BELOW | Must | Un aspecto con rating BELOW genera follow-up automatico con source_type=THREE_POINT_REVIEW |
| `TP-08` | Follow-up resurge en proximo 3-Point (Source B) | Must | Al preparar el siguiente review, los follow-ups abiertos del reviewee aparecen automaticamente como topicos sugeridos |
| `TP-09` | Follow-up resurge en proximo Store Visit (variable item) | Must | Si el follow-up aplica a una tienda, la AI lo sugiere como item variable en el proximo checklist |
| `TP-10` | Follow-up resurge en proximo DA (Source 3) | Must | Si el follow-up se escala, aparece en las fuentes de preparacion del DA |
| `TP-11` | Vista historica de reviews por empleado | Must | Timeline de todos los 3-point reviews recibidos, con ratings y trends por aspecto |
| `TP-12` | Numero de aspectos por review configurable | Should | Default 3, configurable por tenant (2-5 range) |
| `TP-13` | Coverage gap alert | Must | Sistema alerta cuando: aspectos pendientes > meses restantes, o aspecto no revisado en 6+ meses |
| `TP-14` | Performance Evaluation data feed | Must | Toda la data de 3-Point Reviews alimenta el Employee Review Report anual |
| `TP-15` | Notificacion al reviewee al completar review | Must | Reviewee recibe notificacion con aspectos revisados, ratings y action items |
| `TP-16` | AI learning loop | Should | Cada resolucion de finding alimenta el modelo predictivo de la AI |

#### 5.4.8 Business Rules

- `BR-TP-01`: Solo roles con `can_execute_3point_review = true` pueden generar 3-Point Reviews. En Ritmo: desde Gerente de Ventas / Gerente Logistico hacia arriba
- `BR-TP-02`: Un manager solo puede revisar empleados que le reportan directamente (relacion `reports_to`)
- `BR-TP-03`: Maximo un 3-Point Review por par (reviewer, reviewee) por mes calendario
- `BR-TP-04`: Un aspecto puede repetirse en el ano si el reviewer lo considera necesario (times_reviewed_this_year > 1)
- `BR-TP-05`: Coverage se calcula como: aspectos unicos cubiertos / total aspectos en library
- `BR-TP-06`: Un 3-Point Review COMPLETED no puede editarse. Se genera uno nuevo si hay correcciones
- `BR-TP-07`: La sugerencia de AI se persiste en el review (ai_suggested_aspects) para auditoria — se puede comparar lo sugerido vs lo elegido
- `BR-TP-08`: Los follow-ups generados por 3-Point tienen source_type=THREE_POINT_REVIEW y son inmutables en su origen
- `BR-TP-09`: El feedback loop es automatico: el sistema resurge follow-ups abiertos en los proximos mecanismos de control sin intervencion manual
- `BR-TP-10`: La Performance Evaluation de fin de ano se genera a partir de datos acumulados, no requiere input manual adicional — el sistema captura la realidad a lo largo del ano

---

### 5.5 DA Module (Dienstaufsicht)

#### 5.5.1 Purpose

Auditoria directiva a nivel ejecutivo. Es el mecanismo de supervision top-down donde el CEO o COO auditan una region completa, generan findings estructurados, los envian remotamente al Gerente Regional, y luego los discuten en una reunion presencial.

A diferencia del 3-Point Review (que es 1-a-1 sobre una persona), el DA es sobre un **area/region completa** y busca identificar tanto problemas individuales como **issues sistemicos** cross-funcionales.

#### 5.5.2 Quien genera el DA

Dos roles exclusivos, con scopes distintos:

| Executor | da_scope | Areas que audita | Target |
|---|---|---|---|
| **CEO** | `ALL_AREAS` | Operations, Finance, IT, Product — todas las areas funcionales de la region | Gerente Regional |
| **COO** | `OPERATIONS_ONLY` | Stores + Logistics — solo la cadena operativa | Gerente Regional |

**El DA siempre se dirige al Gerente Regional** como target, independientemente de quien lo ejecute. La diferencia es el scope de las areas auditadas.

#### 5.5.3 Con base en que se genera: Data Sources

El DA se prepara con **6 fuentes de datos**, de nivel estrategico (a diferencia de las 5 fuentes del 3-Point que son mas operativas):

| # | Fuente | Que aporta | Nivel | Condicional |
|---|---|---|---|---|
| **1** | Weekly Reports (nivel VP/COO) | Los weeklys ya consolidados a nivel ejecutivo — no los del Jefe de Zona individual | Estrategico | Siempre |
| **2** | Previous DA Findings (Open) | Findings de DA anteriores que siguen abiertos → verificar si se resolvieron | Historico | Siempre |
| **3** | 3-Point Review Findings (Escalated) | Findings de 3-Point que fueron escalados → senal de problemas sistemicos no resueltos a nivel manager | Escalacion | Siempre |
| **4** | Store Visit Data (Ops DA) | Datos de visitas a tienda: scores, trends, findings recurrentes | Operativo | **Solo DA de Operaciones** |
| **5** | Operational KPIs / System Data | Metricas del KARDEX: ventas, inventario, merma, cash, tiempos logisticos | Cuantitativo | Siempre |
| **6** | Financial Reports (Finance DA) | Estados financieros, reconciliaciones, compliance contable | Financiero | **Solo DA de Finance** |

**Nota**: Las fuentes 1, 2, 3 y 5 son comunes a todos los DA. Las fuentes 4 y 6 son condicionales al scope del DA (Operaciones vs Finance).

```
SOURCES THAT FEED DA PREPARATION
+---------------------------------------------+
| 1. Weekly Reports (VP Level)                |
|    Consolidated strategic view              |
+---------------------------------------------+     DA COVERAGE BY ROLE
| 2. Previous DA Findings (Open)              |     +---------------------------+
|    Unresolved issues → must re-check        |     | CEO — ALL AREAS           |
+---------------------------------------------+     | Operations, Finance, IT   |
| 3. 3-Point Review Findings (Escalated)      |     +---------------------------+
|    Escalated issues → systemic signals      |     | COO — OPERATIONS ONLY     |
+---------------------------------------------+     | Stores, Logistics         |
| 4. Store Visit Data (Ops DA only)           |     +---------------------------+
|    Checklist scores, trends, patterns       |
+---------------------------------------------+            |
| 5. Operational KPIs / System Data           |            v
|    KARDEX metrics, anomalies                |     +---------------------------+
+---------------------------------------------+     | DA OUTPUT:                |
| 6. Financial Reports (Finance DA only)      |     | Findings with owner +     |
|    Financial statements, compliance         |     | deadline, systemic issues |
+---------------------------------------------+     | identified, process       |
                                                    | improvements              |
                                                    | → ALL flow to Follow-Up   |
                                                    +---------------------------+
```

#### 5.5.4 A quien se dirige

El DA **siempre se dirige al Gerente Regional**. El workflow tiene un componente remoto y uno presencial:

```
WORKFLOW DA
+----------+     +--------+     +-------------------+     +-----------+
|  DRAFT   |---->|  SENT  |---->| MEETING_SCHEDULED |---->| COMPLETED |
|          |     |        |     |                   |     |           |
| CEO/COO  |     | Envio  |     | Reunion           |     | Findings  |
| prepara  |     | remoto |     | presencial        |     | cerrados  |
| findings |     | al Gte |     | para discutir     |     | con owner |
| con AI   |     | Regional|    | hallazgos         |     | + deadline|
+----------+     +--------+     +-------------------+     +-----------+
                  (dias antes)   (fecha agendada)          (all → Follow-Up)
```

1. **DRAFT**: CEO/COO prepara el DA. La AI pre-carga las 6 fuentes relevantes para el area/region seleccionada
2. **SENT**: El DA con los findings se envia remotamente al Gerente Regional. Este recibe notificacion y acceso al documento
3. **MEETING_SCHEDULED**: Se agenda la reunion presencial. El Gerente Regional puede revisar los findings antes de la reunion
4. **COMPLETED**: Despues de la reunion, el DA se cierra. Cada finding queda formalizado con owner + deadline. Todos fluyen al Follow-Up System

#### 5.5.5 Efectos en cascada sobre el sistema

##### Efecto 1: Follow-Up System (todos los findings, obligatorio)

A diferencia del 3-Point Review (donde solo los aspectos BELOW generan follow-up) y del Checklist (donde los "No Cumple" pueden o no generar follow-up), en el DA **TODOS los findings generan Follow-Up Items obligatoriamente**. Cada finding ya viene con owner + deadline:

```
DA Finding
    |
    v
FollowUpItem {
  source_type: DA
  source_id: <da_finding_id>
  assigned_to: <owner del finding>  // puede ser Gte. Regional, Area Manager, o quien corresponda
  assigned_by: <CEO o COO>
  title: "DA [<period>]: <category> - <description>"
  description: <descripcion completa del finding>
  priority: <derivada de severity: CRITICAL→CRITICAL, HIGH→HIGH, etc.>
  deadline: <fecha asignada en el DA>
  status: OPEN
  auto_surface_in_next_review: true    // resurge en 3-Point del owner
  auto_surface_in_next_checklist: true  // resurge en Store Visit si tiene store_id
  auto_surface_in_next_da: true         // resurge en proximo DA del area
}
```

##### Efecto 2: Feedback Loop (triple resurface)

Los findings de DA resurgen automaticamente en 3 mecanismos de control:

| Donde resurge | Como resurge | Cuando |
|---|---|---|
| **Proximo DA** del area | Aparece como Source 2 (Previous DA Findings Open). El CEO/COO lo ve al preparar el siguiente DA y puede verificar si se resolvio | Segun frecuencia configurada (trimestral default) |
| **Proximo 3-Point Review** de la persona asignada | Aparece como Source C (DA Findings relevantes). El reviewer lo ve al preparar el 3-Point mensual con esa persona | Siguiente mes |
| **Proximo Store Visit** de tiendas afectadas | Aparece como item variable del checklist. La AI lo sugiere al Jefe de Zona al preparar su visita | Proxima visita a la tienda |

##### Efecto 3: Ciclo bidireccional con Store Visits

Existe una relacion circular entre DA y Store Visits que es fundamental para GRIP:

```
Store Visits ──────────→ DA (como Source 4: Store Visit Data)
     ^                         |
     |                         v
     |                    DA Findings
     |                         |
     |                         v
     +──── Follow-Up ←──── (auto_surface_in_next_checklist = true)
          (resurge como item variable en proximo checklist)
```

- Los datos de Store Visits (scores, trends, findings recurrentes) **suben** y alimentan la preparacion del DA
- Los findings del DA **bajan** y resurgen como items variables en los proximos Store Visit checklists
- Este ciclo garantiza que las observaciones ejecutivas se verifiquen a nivel de tienda

##### Efecto 4: Employee Profile del Gerente Regional

El brief en Performance Evaluation muestra "DA Findings Owned" como fuente del Employee Review Report. Se acumula en el perfil:

| Metrica | Descripcion |
|---|---|
| DA findings assigned | Total de findings asignados a esta persona via DA |
| Resolution rate | % de findings resueltos vs total asignados |
| Avg resolution time | Tiempo promedio para resolver findings de DA |
| Escalation count | Cuantos findings se escalaron por no resolverse a tiempo |
| Systemic issues owned | Issues sistemicos formalizados bajo su responsabilidad |

##### Efecto 5: Identificacion de issues sistemicos

El DA Output del brief menciona explicitamente: *"systemic issues identified, process improvements"*. Esto es una capacidad unica del DA que no existe en los otros mecanismos:

- Un **finding individual** es un problema especifico: "Payment terms con Supplier X no se cumplen en Region Norte"
- Un **issue sistemico** es un patron cross-region/cross-area: "El mismo problema de payment terms aparece en 3 regiones → es un problema de proceso, no de personas"

El DA es donde estos patrones se formalizan y se les asigna owner + deadline como process improvements.

```
DA Finding (individual)
    → "Payment terms issue en Region Norte"

DA Finding (individual)
    → "Payment terms issue en Region Sur"

DA Finding (individual)
    → "Payment terms issue en Region Este"
          |
          v
    SYSTEMIC ISSUE IDENTIFIED
    → "Payment terms enforcement process requires redesign"
    → Owner: CFO | Deadline: Q2-2026
    → Follow-Up Item con priority: CRITICAL
```

##### Efecto 6: AI Learning

Cada DA alimenta el AI Engine:
- La AI detecta patrones cross-region: "Este tipo de finding aparece en 3+ regiones → sugerir como issue sistemico"
- Resoluciones exitosas de DA en una region se sugieren como solucion en regiones con problemas similares
- La AI mejora sus sugerencias de preparation para futuros DA basandose en que findings resultaron mas impactantes

#### 5.5.6 Diferencia clave con los otros mecanismos de control

| Dimension | Store Visit Checklist | 3-Point Review | DA (Dienstaufsicht) |
|---|---|---|---|
| **Quien ejecuta** | Jefe de Zona | Desde Gte. Ventas arriba | Solo CEO y COO |
| **Frecuencia** | ~2x/semana por tienda | Mensual por reporte directo | Configurable (default trimestral) |
| **Scope** | 1 tienda | 1 persona (reporte directo) | 1 region/area completa |
| **Nivel de data** | Operativo (items de tienda) | Tactico (aspectos de persona) | Estrategico (VP-level weeklys, KPIs consolidados) |
| **Fuentes** | Template fijo + variables | 5 fuentes (A-E), enfocadas en persona | 6 fuentes (1-6), enfocadas en area |
| **Findings** | Opcionales (No Cumple puede no generar FU) | Condicionales (solo si BELOW) | **Todos obligatorios** con owner + deadline |
| **Delivery** | In-situ durante visita | Conversacion 1-a-1 directa | Envio remoto → reunion presencial |
| **Detecta sistemicos** | No | No | **Si** (cross-region, cross-area) |
| **Bidireccional** | Recibe de DA (variables) | Recibe de DA (Source C) | Recibe de Store Visits (Source 4) |

#### 5.5.7 Data Model

```
DAAudit {
  id: UUID
  tenant_id: UUID
  executor_id: UUID (FK -> Employee) // CEO o COO
  da_scope: Enum [ALL_AREAS, OPERATIONS_ONLY] // derivado del Role del executor
  region_id: UUID (FK -> Area) // region auditada
  target_regional_id: UUID (FK -> Employee) // Gerente Regional target
  audit_period: String // e.g., "Q1-2026"
  fiscal_year: Integer

  // Workflow status
  status: Enum [DRAFT, SENT, MEETING_SCHEDULED, COMPLETED]
  scheduled_meeting_date: Date // fecha de visita presencial
  meeting_location: String // lugar de la reunion

  // AI preparation context
  ai_preparation_summary: JSON // resumen auto-generado de las 6 fuentes
  sources_used: JSON {
    weekly_report_ids: UUID[]        // Source 1: weeklys VP-level
    previous_da_finding_ids: UUID[]  // Source 2: findings DA previos abiertos
    escalated_3point_ids: UUID[]     // Source 3: 3-point findings escalados
    store_visit_ids: UUID[]          // Source 4: store visits (ops DA only)
    kpi_snapshot: JSON               // Source 5: KPIs al momento de preparacion
    financial_report_refs: JSON      // Source 6: refs financieros (finance DA only)
  }

  // Timestamps
  sent_at: Timestamp
  meeting_completed_at: Timestamp
  completed_at: Timestamp
  created_at: Timestamp
  updated_at: Timestamp
}

DAFinding {
  id: UUID
  audit_id: UUID (FK -> DAAudit)

  // Classification
  category: String // area funcional: "Operations", "Finance", "Logistics", "HR", etc.
  finding_type: Enum [INDIVIDUAL, SYSTEMIC] // individual issue vs cross-region pattern
  description: Text
  severity: Enum [CRITICAL, HIGH, MEDIUM, LOW]

  // Assignment (obligatorio — todo finding tiene owner + deadline)
  assigned_to: UUID (FK -> Employee) // responsable de resolver
  deadline: Date

  // For systemic issues
  related_finding_ids: UUID[] // otros findings que forman parte del mismo patron sistemico
  process_improvement_description: Text // si es sistemico, que proceso debe mejorar

  // Lifecycle (tracked via Follow-Up System)
  followup_id: UUID (FK -> FollowUpItem) // referencia cruzada al follow-up generado
  status: Enum [OPEN, IN_PROGRESS, RESOLVED, VERIFIED]
  resolution_notes: Text
  resolution_evidence: JSON // fotos, documentos, links

  created_at: Timestamp
  updated_at: Timestamp
}
```

#### 5.5.8 Functional Requirements

| ID | Requirement | Priority | Acceptance Criteria |
|---|---|---|---|
| `DA-01` | Crear DA seleccionando region y periodo | Must | CEO/COO selecciona region y periodo. Sistema valida que el executor tiene scope para esa region |
| `DA-02` | AI pre-carga las 6 fuentes relevantes | Must | Al crear DA, la AI automaticamente agrega: weeklys VP-level, DA findings previos abiertos, 3-Point escalados, store visit data (si ops), KPIs, financial reports (si finance) |
| `DA-03` | Registrar findings estructurados | Must | Cada finding tiene: categoria, tipo (individual/sistemico), descripcion, severidad, owner, deadline. Owner y deadline son **obligatorios** |
| `DA-04` | Clasificar findings como individuales o sistemicos | Must | El executor puede marcar un finding como SYSTEMIC y vincular findings relacionados de otras regiones |
| `DA-05` | Enviar DA al Gerente Regional antes de reunion | Must | Status pasa a SENT. Gerente Regional recibe notificacion con acceso completo al documento y findings |
| `DA-06` | Programar reunion presencial | Must | Fecha y lugar de reunion quedan registrados. Gerente Regional recibe notificacion de la agenda |
| `DA-07` | Todos los findings generan follow-up automatico al completar | Must | Al status COMPLETED, CADA finding se convierte en FollowUpItem con source_type=DA. Sin excepcion |
| `DA-08` | Follow-ups de DA resurgen en proximo DA (Source 2) | Must | Al preparar un nuevo DA, el sistema muestra automaticamente findings de DA previos que siguen abiertos |
| `DA-09` | Follow-ups de DA resurgen en proximo 3-Point (Source C) | Must | Al preparar 3-Point Review de la persona asignada, los DA findings abiertos aparecen como topico sugerido |
| `DA-10` | Follow-ups de DA resurgen en proximo Store Visit (variable) | Must | Si el finding aplica a tiendas, aparece como item variable en proximos checklists |
| `DA-11` | Vista CEO: todos los DA de todas las areas | Must | Dashboard consolidado de DA por periodo, region y area |
| `DA-12` | Vista COO: DA solo de Operaciones | Must | COO ve solo los DA con da_scope=OPERATIONS_ONLY |
| `DA-13` | Vista Gerente Regional: DA recibidos | Must | Gerente Regional ve los DA que le fueron enviados, con findings y status de cada uno |
| `DA-14` | Frecuencia configurable por tenant | Must | Admin configura frecuencia (mensual, trimestral, semestral, anual). Sistema genera recordatorios cuando toca el proximo DA |
| `DA-15` | Systemic issue linking | Should | Al crear un finding SYSTEMIC, el sistema permite buscar y vincular findings similares de otros DA/regiones |
| `DA-16` | Process improvement tracking | Should | Un finding SYSTEMIC puede generar un "Process Improvement" dedicado, trackeado en Follow-Up con visibilidad cross-region |
| `DA-17` | AI detection de patrones cross-DA | Should | La AI sugiere proactivamente: "Este finding es similar al de Region X — considerar marcar como sistemico" |
| `DA-18` | Audit trail de fuentes usadas | Must | El DA registra exactamente que fuentes se usaron en la preparacion (sources_used JSON) para trazabilidad |

#### 5.5.9 Business Rules

- `BR-DA-01`: Solo roles con `can_execute_da = true` pueden crear DA. El `da_scope` del Role determina que areas pueden auditar (ALL_AREAS para CEO, OPERATIONS_ONLY para COO)
- `BR-DA-02`: Un DA debe tener al menos un finding para poder completarse
- `BR-DA-03`: El status flow es estricto: DRAFT → SENT → MEETING_SCHEDULED → COMPLETED. No se puede saltar pasos
- `BR-DA-04`: Un DA en status COMPLETED no puede editarse. Si hay correcciones, se crea un addendum o un nuevo DA
- `BR-DA-05`: **Todo finding de DA es obligatorio con owner + deadline**. El sistema no permite guardar un finding sin estos campos
- `BR-DA-06`: Los findings de DA alimentan el perfil del Gerente Regional target y del owner asignado en el Employee Database
- `BR-DA-07`: Al completar el DA, se generan automaticamente tantos FollowUpItems como findings existan. Esto es automatico, no opcional
- `BR-DA-08`: Los follow-ups generados por DA tienen los 3 flags de auto-surface activos: `auto_surface_in_next_da = true`, `auto_surface_in_next_review = true`, `auto_surface_in_next_checklist = true` (si aplica a tienda)
- `BR-DA-09`: Un finding SYSTEMIC debe vincular al menos 2 findings relacionados (de otros DA o regiones) para justificar la clasificacion
- `BR-DA-10`: La frecuencia configurada genera recordatorios automaticos al CEO/COO cuando se acerca la fecha del proximo DA pendiente
- `BR-DA-11`: El ciclo bidireccional DA ↔ Store Visits es automatico: Store Visit data alimenta DA preparation (Source 4), y DA findings resurgen como items variables en Store Visits

---

### 5.6 Follow-Up System (Core)

#### 5.6.1 Purpose

Sistema central de GRIP. **Todo finding, de cualquier fuente, converge aqui.** El follow-up system es el mecanismo que garantiza que ningun hallazgo se olvide. Es el corazon del "0 forgotten findings".

#### 5.6.2 Sources of Follow-Up Items

```
+---------------------+
| Store Visit          |---> "No Cumple" con anotacion
| Checklist            |
+---------------------+
| 3-Point Review       |---> Aspecto con rating BELOW
+---------------------+
| DA Finding           |---> Cada finding se convierte en follow-up
+---------------------+
| Weekly Report        |---> Report flagged por superior
+---------------------+
| Manual               |---> Creado directamente por cualquier manager
+---------------------+
         |
         v
+---------------------+
| FOLLOW-UP ITEM      |
| Unified lifecycle   |
+---------------------+
```

#### 5.6.3 Data Model

```
FollowUpItem {
  id: UUID
  tenant_id: UUID

  // Source traceability
  source_type: Enum [CHECKLIST, THREE_POINT_REVIEW, DA, WEEKLY_REPORT, MANUAL]
  source_id: UUID // FK to the originating record
  source_detail: Text // human-readable reference

  // Content
  title: String
  description: Text
  category: String // optional categorization

  // Assignment
  assigned_to: UUID (FK -> Employee)
  assigned_by: UUID (FK -> Employee)
  store_id: UUID (FK -> Store) // nullable, not all follow-ups are store-specific

  // Lifecycle
  status: Enum [OPEN, IN_PROGRESS, RESOLVED, VERIFIED, ESCALATED]
  priority: Enum [CRITICAL, HIGH, MEDIUM, LOW]
  deadline: Date

  // Resolution
  resolution_notes: Text
  resolution_evidence: JSON // photos, links
  resolved_at: Timestamp
  resolved_by: UUID (FK -> Employee)
  verified_at: Timestamp
  verified_by: UUID (FK -> Employee)

  // Escalation
  escalated_at: Timestamp
  escalation_reason: Text

  // Auto-surface (Feedback Loop)
  // Estos flags determinan DONDE resurge este follow-up automaticamente
  auto_surface_in_next_review: Boolean // aparece como Source B en proximo 3-Point Review del assigned_to
  auto_surface_in_next_checklist: Boolean // aparece como item variable en proximo Store Visit de la tienda
  auto_surface_in_next_da: Boolean // aparece como Source 3 en proximo DA del area (si escalado)
  surfaced_in: JSON // registro de donde ha resurgido: [{type, id, date}]

  created_at: Timestamp
  updated_at: Timestamp
}
```

#### 5.6.4 Lifecycle State Machine

```
                    +--------+
          create -> | OPEN   |
                    +--------+
                        |
                   assign/start
                        |
                        v
                  +--------------+
                  | IN_PROGRESS  |<----+
                  +--------------+     |
                    |          |       |
                resolve     escalate  reopen
                    |          |       |
                    v          v       |
              +-----------+  +----------+
              | RESOLVED  |  | ESCALATED|
              +-----------+  +----------+
                    |               |
                  verify        resolve
                    |               |
                    v               |
              +-----------+         |
              | VERIFIED  |<--------+
              +-----------+
                 (closed)
```

#### 5.6.5 Functional Requirements

| ID | Requirement | Priority | Acceptance Criteria |
|---|---|---|---|
| `FU-01` | Creacion automatica desde cualquier source | Must | Un follow-up se crea automaticamente al: marcar "No Cumple", flag weekly, DA finding, 3-point BELOW |
| `FU-02` | Creacion manual | Must | Cualquier manager puede crear un follow-up manual con titulo, descripcion, asignado, deadline |
| `FU-03` | Asignacion a responsable con deadline | Must | Cada follow-up tiene un owner y una fecha limite |
| `FU-04` | Lifecycle management (status transitions) | Must | Transiciones validas segun state machine |
| `FU-05` | Resolucion con evidencia | Must | Al resolver, el responsable debe agregar notas y opcionalmente evidencia |
| `FU-06` | Verificacion por superior | Should | Un follow-up resuelto puede ser verificado (o reabierto) por el manager que lo creo |
| `FU-07` | Escalation automatica por vencimiento | Must | Follow-ups vencidos se escalan automaticamente al nivel superior |
| `FU-08` | Overdue alerts | Must | Notificacion in-app y resumen diario de items vencidos |
| `FU-09` | Auto-surface en proximo 3-Point Review (Source B) | Must | Follow-up abierto del reviewee aparece automaticamente como topico sugerido al reviewer al preparar su proximo 3-Point Review mensual |
| `FU-09b` | Auto-surface en proximo Store Visit checklist (variable item) | Must | Si el follow-up tiene store_id, aparece como item variable sugerido por AI en el proximo checklist de esa tienda |
| `FU-09c` | Auto-surface en proximo DA (Source 3) | Must | Si el follow-up fue escalado, aparece en las fuentes de preparacion del DA del area correspondiente |
| `FU-09d` | Registro de resurgence (surfaced_in) | Must | Cada vez que un follow-up resurge en un mecanismo de control, queda registrado para trazabilidad |
| `FU-10` | Filtros: por status, por asignado, por tienda, por source, por fecha | Must | Todos los filtros combinables |
| `FU-11` | Trazabilidad completa al origen | Must | Desde un follow-up puedo navegar al checklist/review/DA/weekly que lo genero |
| `FU-12` | Close with evidence | Must | Cerrar requiere evidencia (texto minimo, foto opcional) |
| `FU-13` | Bulk actions | Should | SM puede ver todos los follow-ups de su equipo y tomar acciones en lote |
| `FU-14` | Follow-up velocity metrics | Should | Tiempo promedio de resolucion, % on-time, aging distribution |

#### 5.6.6 Business Rules

- `BR-FU-01`: Todo follow-up debe tener assigned_to y deadline al momento de creacion
- `BR-FU-02`: Solo el assigned_to o un superior jerarquico puede cambiar el status
- `BR-FU-03`: La transicion OPEN -> RESOLVED no es valida (debe pasar por IN_PROGRESS)
- `BR-FU-04`: Un follow-up VERIFIED no puede reabrirse
- `BR-FU-05`: La escalation automatica ocurre a las 24h despues del deadline
- `BR-FU-06`: El source_type y source_id son inmutables despues de la creacion
- `BR-FU-07`: El feedback loop es automatico: al crear un follow-up, los flags auto_surface_* se activan segun el contexto (store_id presente → checklist, source=3-Point → next review, escalado → DA)
- `BR-FU-08`: El campo surfaced_in se actualiza cada vez que el item resurge, creando un audit trail de cuantas veces y donde se ha revisado
- `BR-FU-09`: Un follow-up solo deja de resurgir cuando alcanza status VERIFIED (closed). Mientras este OPEN, IN_PROGRESS o ESCALATED, sigue apareciendo en los mecanismos de control

---

### 5.7 Dashboards (Role-Based)

#### 5.7.1 Purpose

Cada rol ve un dashboard diferente, ajustado a su scope y necesidades de decision. El principio central es: **cada dashboard responde UNA pregunta principal** — la que ese rol se hace al abrir el sistema cada dia.

#### 5.7.2 Design Principles

| Principio | Regla |
|---|---|
| **Una pregunta principal por rol** | Cada dashboard existe para responder la pregunta clave del rol |
| **Lo urgente arriba** | Overdue y escalations siempre en la zona superior del dashboard |
| **Drill-down universal** | Todo numero es clickeable → lleva al detalle |
| **AI summary en roles ejecutivos** | COO y CEO no deben leer 50 weeklys; la AI los resume en 3-5 bullets |
| **Forgotten Findings visible** | El KPI "Forgotten Findings = 0" esta presente en todos los dashboards desde Gte. Ventas hacia arriba |
| **Mobile-first** | Jefe de Tienda y Jefe de Zona usan smartphone primariamente. Otros roles son primarily desktop |

#### 5.7.3 Dashboard por Rol

##### 5.7.3.1 Jefe de Tienda Dashboard

**Pregunta principal**: *"Que tengo pendiente en MI tienda?"*

Rol mas operativo. No toma decisiones estrategicas. Necesita saber que resolver y que viene.

| Zona | Widget | Pregunta que responde | Visual | Source |
|---|---|---|---|---|
| Header | Store Health Score | "Como esta mi tienda hoy?" | Gauge: verde/amarillo/rojo basado en ultimo checklist | Checklist |
| Urgente | My Overdue Follow-Ups | "Que se me vencio?" | Lista con badges rojos, ordenada por antiguedad | Follow-Up System |
| Pendiente | My Open Follow-Ups | "Que tengo asignado?" | Cards por status: Open / In Progress | Follow-Up System |
| Proximo | Next Scheduled Visit | "Cuando viene mi Jefe de Zona?" | Fecha + countdown | Checklist + Calendar |
| Historico | Last Checklist Results | "Como sali en la ultima visita?" | Score con delta vs anterior + items No Cumple listados | Checklist |
| KPIs | Store KPI Snapshot | "Como van mis numeros?" | Mini-cards: Ventas, Merma, Agotados, Cash | KARDEX Integration |

##### 5.7.3.2 Jefe de Zona Dashboard (DM)

**Pregunta principal**: *"A cual tienda debo ir hoy y en que enfocarme?"*

Usuario que mas usa el sistema day-to-day. **Mobile-first obligatorio**. Necesita rapidez.

| Zona | Widget | Pregunta que responde | Visual | Source |
|---|---|---|---|---|
| Header | AI Quick Brief | "Que deberia saber ahora?" | Texto corto de AI: "Store 7 tiene 3 items vencidos. Store 12 showing shrinkage pattern" | AI Engine |
| Action | My Open Follow-Ups | "Cuantos pendientes tengo?" | Count by status + aging chart (donut) | Follow-Up System |
| Action | Overdue Items (critical) | "Que se me vencio?" | Lista roja, tap para ver detalle | Follow-Up System |
| Planning | Store Visit Map | "A cual tienda voy?" | Lista de mis ~7 tiendas con: ultimo visit date, score, items abiertos. Color-coded por urgencia | Checklist + Follow-Up |
| Planning | Weekly Report Status | "Ya envie mi weekly?" | Status: Draft/Submitted/Pending con deadline | Weekly Reports |
| KPIs | Store Comparison | "Cual tienda necesita mas atencion?" | Ranking de mis tiendas por: score, follow-ups abiertos, KPIs | Cross-module |
| Review | My 3-Point Reviews | "Que me evaluo mi jefe?" | Ultimo review recibido + aspectos pendientes de mejora | 3-Point Review |
| Trend | My Stores Trend (4 weeks) | "Estan mejorando o empeorando?" | Sparklines de score por tienda (ultimas 4 visitas) | Checklist |

##### 5.7.3.3 Jefe Logistico Dashboard (Recibo / Picking / Despacho)

**Pregunta principal**: *"Como esta MI area del CEDI y que tengo pendiente?"*

Analogo al Jefe de Zona pero en contexto CEDI. Sin checklists de tienda.

| Zona | Widget | Pregunta que responde | Visual | Source |
|---|---|---|---|---|
| Header | CEDI Area Health | "Como esta mi area?" | Gauge basado en KPIs de su area especifica | KARDEX Integration |
| Action | My Open Follow-Ups | "Que pendientes tengo?" | Count by status | Follow-Up System |
| Action | Overdue Items | "Que se me vencio?" | Lista roja | Follow-Up System |
| KPIs | Area KPI Dashboard | "Como van mis metricas?" | Recibo: tiempos de recepcion. Picking: accuracy %. Despacho: on-time % | KARDEX Integration |
| Weekly | Weekly Report Status | "Ya envie mi weekly?" | Status | Weekly Reports |
| Review | My 3-Point Reviews | "Que me evaluo mi jefe?" | Ultimo review recibido | 3-Point Review |

##### 5.7.3.4 Gerente de Ventas Dashboard (Sales Manager)

**Pregunta principal**: *"Cual de mis Jefes de Zona necesita mas atencion y por que?"*

Primer nivel de gestion. Necesita ver a su equipo completo y detectar quien necesita apoyo. Es el primer nivel que ve el KPI de Forgotten Findings.

| Zona | Widget | Pregunta que responde | Visual | Source |
|---|---|---|---|---|
| Header | Team Health Overview | "Como esta mi equipo?" | Heatmap: Jefes de Zona como filas, metricas como columnas (FU velocity, checklist freq, weekly compliance, coverage) | Cross-module |
| Header | Forgotten Findings (my scope) | "Se nos olvido algo?" | Numero grande. **Debe ser 0**. Si no es 0, drill-down inmediato | Follow-Up System |
| Critical | Team Overdue Follow-Ups | "Quien tiene mas problemas?" | Aggregated por Jefe de Zona: barras horizontales de overdue items | Follow-Up System |
| Critical | Stores Trending Down | "Que tiendas empeoran?" | Lista de tiendas con 4+ visitas decrecientes. Click → drill-down al historico | Checklist + AI |
| 3-Point | 3-Point Review Calendar | "Con quien toca review este mes?" | Lista de Jefes de Zona con: proximo review pendiente, coverage YTD% | 36-Point Review |
| 3-Point | Coverage Gaps Alert | "Quien se va a quedar sin cubrir los 36?" | Alerta: "JZ X tiene 12 aspectos pendientes y solo 3 meses" | 3-Point Review |
| Weekly | Weekly Reports Feed | "Que dicen mis Jefes de Zona?" | Feed cronologico de ultimos weeklys. Flag si alguno menciono keywords de riesgo (AI) | Weekly Reports + AI |
| KPIs | District Comparison | "Cual zona rinde mas/menos?" | Ranking de zonas por: ventas, merma, shrinkage | KARDEX Integration |
| DA | DA Findings (my team) | "Que bajo del DA a mi equipo?" | Items de DA asignados a mis Jefes de Zona con status | DA Module |
| AI | AI Insights | "Que patrones detecta la AI?" | Senales: "3 tiendas de zona Norte muestran patron de receiving process" | AI Engine |

##### 5.7.3.5 Gerente Logistico Dashboard (CEDI Manager)

**Pregunta principal**: *"Como esta mi operacion CEDI y quien de mi equipo necesita atencion?"*

Simetrico al Gte. de Ventas pero en contexto CEDI.

| Zona | Widget | Pregunta que responde | Visual | Source |
|---|---|---|---|---|
| Header | CEDI Operations Health | "Como esta el CEDI?" | 3 gauges: Recibo / Picking / Despacho | KARDEX Integration |
| Header | Forgotten Findings (my scope) | "Se nos olvido algo?" | Numero grande. **Debe ser 0** | Follow-Up System |
| Critical | Team Overdue Follow-Ups | "Quien tiene mas pendientes?" | Aggregated por Jefe Logistico | Follow-Up System |
| 3-Point | 3-Point Review Calendar | "Con quien toca review?" | Jefes de Recibo, Picking, Despacho con coverage YTD | 36-Point Review |
| 3-Point | Coverage Gaps Alert | "Quien va retrasado en cobertura?" | Alerta de pace | 3-Point Review |
| KPIs | CEDI KPI Dashboard | "Como van los numeros?" | Tiempos recibo, picking accuracy, despacho on-time, errores | KARDEX Integration |
| Weekly | Weekly Reports Feed | "Que reportan mis jefes?" | Feed de weeklys | Weekly Reports |
| DA | DA Findings (my team) | "Que bajo del DA?" | Items asignados a su equipo | DA Module |

##### 5.7.3.6 Gerente Regional Dashboard

**Pregunta principal**: *"Como esta MI region cross-funcionalmente y donde debo intervenir?"*

Dashboard mas complejo: ve **multiples areas funcionales** simultaneamente.

| Zona | Widget | Pregunta que responde | Visual | Source |
|---|---|---|---|---|
| Header | Regional Health Score | "Como esta mi region?" | Score compuesto: Ventas + CEDI + Follow-ups + Coverage. Trend arrow | Cross-module |
| Header | Forgotten Findings (my region) | "Tengo findings abandonados?" | Numero grande. **Debe ser 0** | Follow-Up System |
| Cross-func | Area Manager Heatmap | "Que area funcional necesita atencion?" | Heatmap: Gte Ventas, Gte Logistico, Gte Admin, etc. como filas. Metricas como columnas | Cross-module |
| Critical | Escalated Follow-Ups | "Que escalo a mi nivel?" | Lista de items escalados con source (checklist, 3-point, DA) | Follow-Up System |
| 3-Point | 3-Point Review Calendar | "Con quien toca review?" | Area Managers con coverage YTD | 36-Point Review |
| 3-Point | Team Coverage Overview | "Todos mis managers estan cubriendo los 36 puntos con su gente?" | Vista cascada: Regional → Area Managers → sus reportes. Coverage % por cada par | 3-Point Review |
| DA | DA Findings (my region) | "Que me dejo el ultimo DA?" | Findings por status: Open, In Progress, Resolved. Progress bar general | DA Module |
| DA | DA Preparation Alert | "Cuando viene el proximo DA?" | Countdown + readiness score (cuantos items abiertos tengo) | DA Module |
| KPIs | Regional KPI Summary | "Como van los numeros?" | Ventas, merma, shrinkage, cash, CEDI metrics. Trend vs periodo anterior | KARDEX Integration |
| Weekly | Weekly Consolidation | "Que pasa en mi region esta semana?" | Feed consolidado de weeklys de Area Managers + AI summary | Weekly Reports + AI |
| AI | Early Warnings | "Que problemas vienen?" | Alertas predictivas: store trending down, systemic patterns | AI Engine |

##### 5.7.3.7 COO Dashboard

**Pregunta principal**: *"Como estan las operaciones en todo el pais y donde hay riesgo sistemico?"*

| Zona | Widget | Pregunta que responde | Visual | Source |
|---|---|---|---|---|
| Header | Operations Pulse | "La operacion esta sana?" | 3 mega-KPIs: Forgotten Findings count (debe ser 0), Follow-up closure rate (target >90%), Store health average | Cross-module |
| Comparison | Regional Ranking | "Que region rinde mejor/peor?" | Leaderboard de Gerentes Regionales por score compuesto. Color-coded | Cross-module |
| Critical | Critical Escalations | "Que llego a mi nivel?" | Items escalados de todas las regiones con severity y aging | Follow-Up System |
| Systemic | Systemic Issues Dashboard | "Hay problemas de proceso, no de personas?" | Issues SYSTEMIC activos: cuantas regiones afecta, owner, deadline, progress | DA Module + AI |
| DA | DA Calendar & Status | "Que DA tengo pendiente y cuales estan abiertos?" | Timeline de DA por region: proximos, en proceso, completados | DA Module |
| DA | DA Resolution Rate | "Se resuelven los findings de mis DA?" | % resolucion por region, tiempo promedio, aging distribution | DA Module + Follow-Up |
| 3-Point | 3-Point Coverage Cascade | "Se estan haciendo los reviews en toda la organizacion?" | Vista cascada: regiones → coverage % promedio. Drill-down a detalle | 3-Point Review |
| KPIs | Operations KPI Trends | "Estamos mejorando o empeorando?" | Line charts: ventas, merma, shrinkage, CEDI metrics. 12 semanas trend | KARDEX Integration |
| Weekly | AI-Summarized Weeklys | "Que debo saber de esta semana?" | AI resume de los weeklys de Gerentes Regionales en 3-5 bullets | Weekly Reports + AI |
| AI | Pattern Alerts | "Que ve la AI que yo no?" | Cross-region patterns, early warnings, predictions | AI Engine |

##### 5.7.3.8 CEO Dashboard

**Pregunta principal**: *"Estan funcionando los mecanismos de control? La empresa esta bajo control?"*

El CEO no necesita detalle operativo — necesita **senales de que el sistema funciona** y alertas cuando no.

| Zona | Widget | Pregunta que responde | Visual | Source |
|---|---|---|---|---|
| Header | GRIP Health Score | "El sistema de control funciona?" | Un numero/gauge compuesto de: forgotten findings, coverage rate, follow-up velocity, DA compliance | Cross-module |
| KPI #1 | **Forgotten Findings** | "Se nos esta olvidando algo?" | **EL KPI DE GRIP**. Numero grande. Debe ser **0**. Si no es 0, drill-down inmediato a lista de items abandonados | Follow-Up System |
| KPI #2 | Follow-Up Closure Rate | "Se resuelven las cosas?" | % con trend arrow. Target: >90% | Follow-Up System |
| KPI #3 | 36-Point Coverage Rate | "Los managers estan haciendo los reviews?" | % promedio cross-company. Drill-down por nivel | 3-Point Review |
| KPI #4 | DA Compliance | "Los DA se ejecutan a tiempo?" | DA ejecutados vs esperados segun frecuencia configurada | DA Module |
| Strategic | Company KPI Overview | "Como va el negocio?" | Ventas, merma, inventario, cash — consolidado. Trend 12 meses | KARDEX Integration |
| DA | DA Portfolio | "Como van mis auditorias?" | Todos los DA del periodo: por region, por area. Status y resolution rate | DA Module |
| Systemic | Systemic Issues (company-wide) | "Hay problemas de proceso que cruzan regiones?" | Issues SYSTEMIC activos. Impact map: cuantas regiones afecta cada uno | DA Module + AI |
| AI | Executive AI Brief | "Que debo saber hoy?" | AI genera 3-5 bullets: lo mas importante basado en todas las fuentes del sistema | AI Engine |
| People | Performance Overview | "Quien rinde y quien no?" | Ranking de Gerentes Regionales y COO por score compuesto | Cross-module |
| Adoption | Control System Adoption | "La gente esta usando GRIP?" | Usage metrics: checklists/semana, weeklys submitted, reviews completados, DA on-time | Cross-module |

#### 5.7.4 Widget Cross-Reference Matrix

Vista de que widgets aparecen en cada dashboard para verificar consistencia:

| Widget Category | Jefe Tienda | Jefe Zona | Jefe Log. | Gte Ventas | Gte Log. | Gte Regional | COO | CEO |
|---|---|---|---|---|---|---|---|---|
| **Follow-Up (my items)** | x | x | x | - | - | - | - | - |
| **Follow-Up (team overview)** | - | - | - | x | x | x | x | x |
| **Forgotten Findings KPI** | - | - | - | x | x | x | x | x |
| **Overdue/Escalations** | x | x | x | x | x | x | x | x |
| **Store Visit / Checklist** | x | x | - | x | - | x | - | - |
| **Weekly Report status** | - | x | x | - | - | - | - | - |
| **Weekly Reports feed** | - | - | - | x | x | x | - | - |
| **Weekly AI summary** | - | - | - | - | - | x | x | x |
| **3-Point (received)** | - | x | x | - | - | - | - | - |
| **3-Point (calendar/execute)** | - | - | - | x | x | x | x | - |
| **3-Point (coverage cascade)** | - | - | - | x | x | x | x | x |
| **DA Findings (assigned)** | - | - | - | x | x | x | - | - |
| **DA Portfolio/Status** | - | - | - | - | - | x | x | x |
| **KPIs (store-level)** | x | x | - | x | - | - | - | - |
| **KPIs (CEDI)** | - | - | x | - | x | x | x | - |
| **KPIs (consolidated)** | - | - | - | - | - | x | x | x |
| **AI Quick Brief** | - | x | - | - | - | - | - | - |
| **AI Insights/Patterns** | - | - | - | x | - | x | x | x |
| **AI Executive Brief** | - | - | - | - | - | - | x | x |
| **Systemic Issues** | - | - | - | - | - | x | x | x |
| **Regional Ranking** | - | - | - | - | - | - | x | x |
| **System Adoption** | - | - | - | - | - | - | - | x |

#### 5.7.5 Functional Requirements

| ID | Requirement | Priority | Acceptance Criteria |
|---|---|---|---|
| `DB-01` | Dashboard role-based automatico | Must | Al login, el user ve su dashboard segun su Role configurado |
| `DB-02` | Drill-down universal | Must | Todo numero, card o barra en cualquier widget es clickeable y navega al detalle (lista, perfil, modulo) |
| `DB-03` | Filtros contextuales | Must | Filtros por periodo, tienda/CEDI, persona. Los filtros disponibles dependen del rol |
| `DB-04` | Refresh near-real-time | Must | Datos actualizados al menos cada 5 minutos. Widgets criticos (overdue, escalations) en tiempo real |
| `DB-05` | Responsive mobile-first (DM/ST) | Must | Dashboards de Jefe de Tienda y Jefe de Zona optimizados para 375px. Tap targets minimo 44x44px |
| `DB-06` | Responsive desktop-first (managers+) | Must | Dashboards de Gte. Ventas hacia arriba optimizados para 1280px+ con layout de columnas |
| `DB-07` | Export basico | Should | Exportar vista actual a PDF o CSV. Util para Gerente Regional preparando reunion |
| `DB-08` | Forgotten Findings visible en Gte. Ventas+ | Must | El widget Forgotten Findings aparece en todos los dashboards desde Gte. Ventas hacia arriba |
| `DB-09` | AI summary en COO y CEO | Must | Los dashboards de COO y CEO incluyen AI-generated summaries de weeklys y situacion general |
| `DB-10` | Widget Cross-Reference compliance | Must | Cada dashboard implementa todos los widgets listados en la matriz 5.7.4 para su rol |
| `DB-11` | Color-coding consistente | Must | Verde = on track, Amarillo = warning, Rojo = critical. Consistente en todos los dashboards |
| `DB-12` | Zero-state handling | Must | Cuando no hay datos (nuevo tenant, primer uso), los widgets muestran estado vacio informativo, no errores |

#### 5.7.6 Business Rules

- `BR-DB-01`: El dashboard se determina automaticamente por el `role_id` del Employee. No se elige manualmente
- `BR-DB-02`: Los datos visibles en cada widget respetan estrictamente la jerarquia de visibilidad (un Gte. Ventas solo ve datos de SUS Jefes de Zona, nunca de otra region)
- `BR-DB-03`: El KPI "Forgotten Findings" se calcula como: follow-ups con status OPEN y sin ninguna actualizacion en los ultimos N dias configurables (default: 7 dias)
- `BR-DB-04`: "Store Trending Down" se calcula cuando una tienda tiene scores de checklist decrecientes en 4 o mas visitas consecutivas
- `BR-DB-05`: La AI genera el Executive Brief y Weekly Summary solo con datos dentro del scope de visibilidad del usuario
- `BR-DB-06`: Los sparklines de trend usan las ultimas 4 semanas para Jefe de Zona y ultimas 12 semanas para niveles ejecutivos

---

### 5.8 AI Assistant Layer

#### 5.8.1 Purpose

Capa inteligente que lee todos los datos del sistema, aprende patrones, y asiste en cada tarea de control. No reemplaza decisiones humanas; las potencia con contexto, patrones y predicciones.

#### 5.8.2 AI Engine - Input Sources

| Source | Data Fed to AI |
|---|---|
| Weekly Reports | Texto libre (NLP) |
| Store Visit Checklists | Results, annotations, scores, trends |
| 3-Point Reviews | Ratings, coverage, observations |
| DA Findings | Categories, severities, resolutions |
| Follow-Up Items | Status, velocity, aging, escalations |
| Operational KPIs | Sales, inventory, shrinkage, cash (from KARDEX) |
| Employee Database | Roles, assignments, history |

#### 5.8.3 AI Capabilities

##### 5.8.3.1 Preparation Assistant

Ayuda a DMs y managers a prepararse para cada tarea de control con contexto relevante.

| ID | Capability | Trigger | Output |
|---|---|---|---|
| `AI-PREP-01` | Store Visit Prep | DM inicia preparacion para visita a tienda X | Checklist variable sugerido: items basados en findings abiertos, patrones, feedback del SM |
| `AI-PREP-02` | 3-Point Review Prep | SM inicia review mensual de DM Y | Sugiere 3 aspectos basados en: gaps de coverage, senales en weeklys, findings recientes |
| `AI-PREP-03` | DA Prep | CEO inicia DA de area Z | Pre-agrega contexto: findings previos, follow-ups abiertos, KPIs anomalos |
| `AI-PREP-04` | Team Meeting Prep | SM solicita resumen para reunion de equipo | Resumen de items criticos por DM, quien necesita atencion prioritaria |

##### 5.8.3.2 Pattern Detection (Cross-Company)

| ID | Pattern | Example |
|---|---|---|
| `AI-PAT-01` | Shrinkage predictor | Tiendas con 3+ findings de receiving process en checklists muestran 0.4% mas shrinkage en 60 dias |
| `AI-PAT-02` | Follow-up velocity | DMs que caen en frecuencia de visitas tienen 2x mas follow-ups abiertos 3 meses despues |
| `AI-PAT-03` | Weekly report signal | Cuando weeklys mencionan "staffing challenges" 2+ veces, DA findings siguen el 60% de las veces |
| `AI-PAT-04` | Cross-area learning | Finding de Finanzas en Area A es similar a uno resuelto por Operaciones en Area B en Q2 |

##### 5.8.3.3 Early Warning Alerts

| ID | Alert | Trigger |
|---|---|---|
| `AI-EW-01` | Performance decline | Follow-up closure rate cae >15% en 2 meses |
| `AI-EW-02` | Store trending down | Checklist scores decrecientes 4+ visitas consecutivas |
| `AI-EW-03` | Systemic issue emerging | Mismo tipo de finding aparece en 5+ tiendas de 3+ DMs |
| `AI-EW-04` | Coverage gap | Manager no ha revisado un aspecto en 6+ meses |

##### 5.8.3.4 Follow-Up Automation

| ID | Capability | Description |
|---|---|---|
| `AI-FUA-01` | Overdue summary | Resumen de items vencidos con contexto |
| `AI-FUA-02` | Reminder drafting | Borradores de recordatorios para items proximos a vencer |
| `AI-FUA-03` | Escalation suggestion | Sugiere escalaciones basadas en patrones de no-resolucion |

##### 5.8.3.5 Performance Synthesis

| ID | Capability | Description |
|---|---|---|
| `AI-PS-01` | Annual performance summary | Genera resumen de rendimiento basado en todos los datos acumulados del ano |
| `AI-PS-02` | Development recommendations | Sugiere areas de desarrollo basadas en gaps de 36-point y findings recurrentes |
| `AI-PS-03` | Review focus recommendations | Para 3-point review: sugiere en que enfocarse basado en data |

#### 5.8.4 AI Interaction Model

La interaccion con la AI es via **chat conversacional en lenguaje natural**, integrado en el contexto de cada modulo.

```
+----------------------------------------------------------+
| AI ASSISTANT                                    [context] |
|                                                          |
| User: "What should I focus on at Store 7 this week?"    |
|                                                          |
| AI: Based on the data, I recommend focusing on these    |
|     3 items for your variable checklist:                 |
|     1. Cash reconciliation - last visit found a         |
|        discrepancy, still marked as open                 |
|     2. Receiving process - Store 7's shrinkage is 0.3%  |
|        above district average                            |
|     3. Staff scheduling - similar pattern to Store 12   |
|        before their stockout issue                       |
|                                                          |
| User: "Add that to my variable checklist for today"     |
|                                                          |
| AI: Done. Your variable checklist for Store 7 now       |
|     includes: Cash reconciliation follow-up, Receiving  |
|     process review, Weekend staffing vs. restocking     |
|     schedule. Good luck with the visit.                  |
+----------------------------------------------------------+
```

#### 5.8.5 AI Functional Requirements

| ID | Requirement | Priority | Acceptance Criteria |
|---|---|---|---|
| `AI-01` | Chat conversacional por contexto | Must | AI assistant disponible en cada modulo con contexto pre-cargado |
| `AI-02` | Preparation assistant para store visit | Must | Sugiere items variables basados en datos historicos |
| `AI-03` | Preparation assistant para 3-point review | Must | Sugiere aspectos basados en coverage gaps y senales |
| `AI-04` | Pattern detection cross-company | Must | Identifica patrones que cruzan tiendas, areas y periodos |
| `AI-05` | Early warning alerts | Must | Genera alertas proactivas cuando detecta tendencias negativas |
| `AI-06` | Follow-up overdue summary | Must | Resume items vencidos con contexto |
| `AI-07` | Performance summary generation | Should | Genera borrador de resumen de performance anual |
| `AI-08` | Cross-area learning | Should | Conecta resoluciones exitosas de un area con problemas similares en otra |
| `AI-09` | Learning loop | Must | Cada resolucion alimenta el modelo; el sistema se hace mas inteligente con cada ciclo |
| `AI-10` | Natural language queries | Must | User puede preguntar en lenguaje natural sobre cualquier dato del sistema |
| `AI-11` | Action execution from chat | Should | AI puede ejecutar acciones (agregar items, crear follow-ups) desde la conversacion |

#### 5.8.6 Business Rules AI

- `BR-AI-01`: La AI nunca toma decisiones autonomas. Sugiere, el humano confirma
- `BR-AI-02`: Toda sugerencia de AI debe ser trazable a los datos que la originaron
- `BR-AI-03`: El historial de chat con AI se persiste por contexto (modulo + entidad)
- `BR-AI-04`: La AI respeta los limites de visibilidad del rol del usuario
- `BR-AI-05`: Cross-company patterns se detectan dentro del mismo tenant unicamente (data isolation)

---

#### 5.8.7 Prompt Engineering Models

> **Propósito:** Esta sección especifica el modelo de ingeniería de prompt para cada actividad de IA en GRIP. Cada prompt tiene una estructura fija (system prompt) y una sección dinámica (user prompt con contexto inyectado). Los prompts son parte de la especificación funcional — son auditables, versionables y testeables igual que cualquier otro requisito.

##### Estructura base (aplica a todos los modelos)

```
┌─────────────────────────────────────────────────────────────┐
│ SYSTEM PROMPT (estático, fijo por actividad)                │
│ Define el rol de la IA, el propósito, el idioma,            │
│ las restricciones y el formato de salida esperado.          │
├─────────────────────────────────────────────────────────────┤
│ USER PROMPT (dinámico, generado en runtime)                 │
│ Inyecta el contexto estructurado del sistema                │
│ (datos reales del tenant/usuario/entidad)                   │
│ + la instrucción específica de la tarea                     │
└─────────────────────────────────────────────────────────────┘
```

**Restricciones globales (heredadas por todos los modelos):**
- La IA responde siempre en el idioma configurado por el tenant (`tenant.locale`)
- La IA no inventa datos — si el contexto es insuficiente, lo declara explícitamente
- La IA no toma acciones autónomas — produce outputs que el usuario confirma
- Todo output incluye el campo `data_sources[]` con las fuentes que originaron la respuesta
- Si la IA no está disponible, el sistema ejecuta el **fallback mecánico** definido por actividad

---

##### AI-01 — Store Visit Preparation Brief

| Campo | Valor |
|---|---|
| **Actividad** | Preparación de visita a tienda |
| **Trigger** | JZ abre o inicia preparación para visita a tienda X |
| **Rol que consume** | Jefe de Zona |
| **Módulo** | Store Visit Checklist (5.3) |

**Context inputs inyectados:**
```
- store.id, store.name, store.code
- last_5_checklist_sessions[]: fecha, score global, items fallidos
- open_followup_items[]: descripción, aging, severity, owner
- kardex_kpis_last_30d: ventas, merma, inventario, cash
- previous_variable_items[]: items que el JZ añadió en las últimas 3 visitas
- peer_store_patterns[]: findings similares en tiendas de la misma zona (últimos 60 días)
```

**System prompt:**
```
Eres el asistente de preparación de visitas de GRIP para Jefes de Zona.
Tu misión es ayudar al JZ a llegar a la visita con el contexto necesario
para no perderse nada importante.

Reglas:
- Solo usa los datos que te son provistos. No inventes ni asumas.
- Si no hay datos suficientes para una sección, indícalo claramente.
- Sé conciso. El JZ leerá esto en 2 minutos antes de entrar a la tienda.
- Responde en [tenant.locale].
- Responde SOLO en el JSON schema indicado.
```

**User prompt (template):**
```
Datos de la tienda [store.name] ([store.code]):

ÚLTIMAS 5 VISITAS:
[last_5_checklist_sessions — serializado]

HALLAZGOS ABIERTOS:
[open_followup_items — serializado]

KPIs ÚLTIMOS 30 DÍAS (KARDEX):
[kardex_kpis_last_30d — serializado]

PATRONES EN TIENDAS PARES:
[peer_store_patterns — serializado]

Con base en estos datos, genera un briefing de preparación de visita.
```

**Output schema:**
```json
{
  "priority_areas": [
    {
      "area": "string",
      "reason": "string",
      "urgency": "HIGH | MEDIUM | LOW"
    }
  ],
  "open_findings_summary": {
    "total": "integer",
    "overdue": "integer",
    "critical": ["string"]
  },
  "kpi_alerts": [
    {
      "metric": "string",
      "value": "string",
      "benchmark": "string",
      "alert": "string"
    }
  ],
  "suggested_variable_items": [
    {
      "description": "string",
      "category": "string",
      "priority": "HIGH | MEDIUM | LOW",
      "justification": "string",
      "source_type": "FINDING | KPI | PATTERN"
    }
  ],
  "data_sources": ["string"]
}
```

**Fallback (sin IA):** Mostrar lista de open follow-ups ordenados por aging. Sin síntesis.
**Surface:** Panel "Preparación para visita" antes de iniciar el checklist.

---

##### AI-02 — 3-Point Review Preparation Brief

| Campo | Valor |
|---|---|
| **Actividad** | Preparación de revisión mensual de desempeño |
| **Trigger** | Reviewer abre formulario de 3-Point Review para reviewee Y |
| **Rol que consume** | Gte. de Ventas / Gte. Logístico / Gte. Regional / COO |
| **Módulo** | 3-Point Review (5.4) |

**Context inputs inyectados:**
```
- reviewer.id, reviewee.id, reviewee.name, reviewee.role
- aspect_history_last_6_months[]: aspecto evaluado, fecha, score, comentario
- aspects_never_evaluated[]: aspectos del catálogo nunca seleccionados para este reviewee
- aspects_with_low_scores[]: aspectos con score < umbral en últimas 3 evaluaciones
- reviewee_checklist_scores_last_60d[]: scores de visitas (si aplica al rol)
- reviewee_open_followups[]: cantidad, aging promedio, tasa de cierre
- reviewee_weekly_report_signals[]: keywords negativos detectados en últimos reportes
```

**System prompt:**
```
Eres el asistente de preparación de revisiones de desempeño de GRIP.
Tu misión es ayudar al evaluador a seleccionar los 3 aspectos más relevantes
para revisar con su reporte directo, basándote en evidencia objetiva del sistema.

Reglas:
- Sugiere exactamente 3 aspectos del catálogo disponible.
- Cada sugerencia debe estar respaldada por datos concretos.
- Prioriza aspectos con evidencia de deterioro o nunca evaluados.
- No repitas aspectos evaluados en el ciclo inmediatamente anterior.
- Responde en [tenant.locale].
- Responde SOLO en el JSON schema indicado.
```

**User prompt (template):**
```
Evaluador: [reviewer.name] ([reviewer.role])
Evaluado: [reviewee.name] ([reviewee.role])

HISTORIAL DE ASPECTOS EVALUADOS (últimos 6 meses):
[aspect_history_last_6_months — serializado]

ASPECTOS NUNCA EVALUADOS:
[aspects_never_evaluated — serializado]

ASPECTOS CON SCORES BAJOS:
[aspects_with_low_scores — serializado]

SEÑALES EN REPORTES SEMANALES:
[reviewee_weekly_report_signals — serializado]

FOLLOW-UPS DEL EVALUADO:
[reviewee_open_followups — serializado]

Sugiere los 3 aspectos más relevantes para evaluar en este ciclo.
```

**Output schema:**
```json
{
  "suggested_aspects": [
    {
      "aspect_id": "string",
      "aspect_name": "string",
      "reason": "string",
      "supporting_evidence": ["string"],
      "priority_rank": 1
    }
  ],
  "coverage_alert": {
    "months_since_full_cycle": "integer",
    "uncovered_categories": ["string"]
  },
  "data_sources": ["string"]
}
```

**Fallback (sin IA):** Mostrar aspectos nunca evaluados ordenados por fecha de última evaluación.
**Surface:** Panel "Sugerencias de aspectos" en el formulario de 3-Point Review.

---

##### AI-03 — DA Preparation Brief

| Campo | Valor |
|---|---|
| **Actividad** | Preparación de Dienstaufsicht (auditoría directiva) |
| **Trigger** | COO o CEO abre formulario de DA para Gerente Regional Z |
| **Rol que consume** | COO / CEO |
| **Módulo** | DA Module (5.5) |

**Context inputs inyectados:**
```
- auditor.id, auditor.role, auditor.da_scope
- subject.id, subject.name (Gerente Regional)
- previous_das[]: fecha, hallazgos totales, hallazgos cerrados, hallazgos abiertos, áreas auditadas
- open_da_findings[]: descripción, aging, severity, owner, ciclos sin cerrar
- regional_kpi_summary_last_quarter: KPIs consolidados de la región
- weekly_reports_last_8_weeks[]: señales detectadas por AI en reportes del regional
- escalated_followups[]: follow-ups escalados al regional en últimos 90 días
```

**System prompt:**
```
Eres el asistente de preparación de auditorías directivas (DA) de GRIP.
Tu misión es proveer al ejecutivo un briefing completo antes de iniciar una DA,
para que llegue con contexto profundo y pueda hacer preguntas precisas.

Reglas:
- Prioriza hallazgos no cerrados de DAs anteriores — son la señal más crítica.
- Identifica patrones que se repiten en múltiples ciclos.
- Señala anomalías en KPIs que merezcan exploración directa.
- Sé ejecutivo: el CEO/COO lee esto en 5 minutos.
- Responde en [tenant.locale].
- Responde SOLO en el JSON schema indicado.
```

**User prompt (template):**
```
Auditor: [auditor.name] ([auditor.role]) — Scope: [auditor.da_scope]
Sujeto: [subject.name] — Gerente Regional

DAS PREVIAS:
[previous_das — serializado]

HALLAZGOS ABIERTOS DE DAS ANTERIORES:
[open_da_findings — serializado]

KPIs REGIONALES (último trimestre):
[regional_kpi_summary_last_quarter — serializado]

SEÑALES EN REPORTES SEMANALES:
[weekly_reports_last_8_weeks — serializado]

FOLLOW-UPS ESCALADOS:
[escalated_followups — serializado]

Genera el briefing pre-DA.
```

**Output schema:**
```json
{
  "executive_summary": "string (máx 3 oraciones)",
  "unresolved_findings": [
    {
      "finding_id": "string",
      "description": "string",
      "aging_days": "integer",
      "cycles_open": "integer",
      "severity": "HIGH | MEDIUM | LOW"
    }
  ],
  "risk_areas": [
    {
      "area": "string",
      "risk_level": "HIGH | MEDIUM | LOW",
      "evidence": ["string"]
    }
  ],
  "kpi_anomalies": [
    {
      "metric": "string",
      "value": "string",
      "expected": "string",
      "delta": "string"
    }
  ],
  "recommended_focus_areas": ["string"],
  "data_sources": ["string"]
}
```

**Fallback (sin IA):** Mostrar lista de hallazgos abiertos de DAs anteriores ordenados por aging.
**Surface:** Pantalla "Briefing pre-DA" antes de abrir el formulario de DA.

---

##### AI-04 — Pattern Detection

| Campo | Valor |
|---|---|
| **Actividad** | Detección de patrones sistémicos cross-tiendas |
| **Trigger** | Job programado semanal + solicitud manual desde dashboard |
| **Rol que consume** | Gte. Ventas / Gte. Logístico / Gte. Regional / COO / CEO |
| **Módulo** | Dashboard (5.7) + Follow-Up System (5.6) |

**Context inputs inyectados:**
```
- tenant_id (scope completo del tenant)
- findings_last_90_days[]: descripción, categoría, store_id, zone_id, region_id, fecha, estado
- checklist_item_failures_last_90_days[]: ítem, frecuencia de fallo, distribución por tienda/zona
- followup_resolution_rates_by_zone[]: zona, tasa de cierre, aging promedio
- scope_filter: según el rol del usuario (región, zona, tienda)
```

**System prompt:**
```
Eres el motor de detección de patrones de GRIP.
Tu misión es identificar problemas sistémicos que no son visibles
cuando se miran las tiendas o zonas de forma individual.

Reglas:
- Un patrón requiere mínimo 3 instancias en 2 o más zonas distintas.
- Distingue entre problemas sistémicos (múltiples zonas) y problemas locales (1 zona).
- Ordena los patrones por impacto potencial (frecuencia × severidad).
- Solo reporta patrones con evidencia sólida. Si los datos son insuficientes, indica cuántos más ciclos se necesitan.
- Responde en [tenant.locale].
- Responde SOLO en el JSON schema indicado.
```

**User prompt (template):**
```
Tenant: [tenant_id] — Scope del usuario: [scope_filter]

FINDINGS ÚLTIMOS 90 DÍAS:
[findings_last_90_days — serializado]

ÍTEMS DE CHECKLIST CON MAYOR TASA DE FALLO:
[checklist_item_failures_last_90_days — serializado]

TASAS DE CIERRE POR ZONA:
[followup_resolution_rates_by_zone — serializado]

Detecta patrones sistémicos y genera alertas de patrón.
```

**Output schema:**
```json
{
  "patterns": [
    {
      "pattern_id": "string (generado)",
      "pattern_type": "SYSTEMIC | LOCAL | EMERGING",
      "title": "string",
      "description": "string",
      "affected_stores": ["store_id"],
      "affected_zones": ["zone_id"],
      "frequency": "integer",
      "severity": "HIGH | MEDIUM | LOW",
      "first_occurrence": "date",
      "latest_occurrence": "date",
      "recommended_action": "string"
    }
  ],
  "insufficient_data_note": "string | null",
  "data_sources": ["string"]
}
```

**Fallback (sin IA):** No mostrar widget. Los datos crudos están disponibles en el dashboard via filtros manuales.
**Surface:** Widget "Patrones detectados" en dashboards de Gte. Ventas y superiores.

---

##### AI-05 — Early Warning

| Campo | Valor |
|---|---|
| **Actividad** | Alerta temprana de degradación operativa |
| **Trigger** | Job programado diario que evalúa umbrales configurables |
| **Rol que consume** | Gte. Regional / COO / CEO |
| **Módulo** | Dashboard (5.7) |

**Context inputs inyectados:**
```
- checklist_scores_per_store_last_8_visits[]: store_id, scores cronológicos
- followup_closure_rate_per_jz_last_60d[]: jz_id, tasa de cierre, trend
- kardex_kpi_trends_last_12_weeks[]: store_id, ventas, merma, inventario (trend)
- three_point_scores_per_employee_last_3_cycles[]: employee_id, scores, trend
- scope_filter: región/zona del usuario
- thresholds: configurados por tenant (ej: alerta si score cae >15% en 4 visitas)
```

**System prompt:**
```
Eres el sistema de alertas tempranas de GRIP.
Tu misión es detectar señales de degradación antes de que se conviertan en crisis,
dando tiempo al manager para intervenir proactivamente.

Reglas:
- Una alerta requiere tendencia negativa consistente (mínimo 3 puntos de datos).
- Distingue entre degradación confirmada (tendencia clara) y señal débil (solo 2 puntos).
- Prioriza alertas por impacto potencial en operación.
- No generes alertas por fluctuaciones normales — usa los umbrales configurados.
- Responde en [tenant.locale].
- Responde SOLO en el JSON schema indicado.
```

**User prompt (template):**
```
Scope: [scope_filter]
Umbrales configurados: [thresholds — serializado]

SCORES DE CHECKLIST (últimas 8 visitas por tienda):
[checklist_scores_per_store_last_8_visits — serializado]

TASAS DE CIERRE POR JEFE DE ZONA (últimos 60 días):
[followup_closure_rate_per_jz_last_60d — serializado]

TENDENCIAS KPI KARDEX (últimas 12 semanas):
[kardex_kpi_trends_last_12_weeks — serializado]

Detecta degradaciones y genera alertas tempranas.
```

**Output schema:**
```json
{
  "warnings": [
    {
      "warning_id": "string (generado)",
      "entity_type": "STORE | ZONE | EMPLOYEE | REGION",
      "entity_id": "string",
      "entity_name": "string",
      "warning_type": "CHECKLIST_DECLINE | FOLLOWUP_STAGNATION | KPI_DETERIORATION | PERFORMANCE_DECLINE",
      "severity": "CRITICAL | HIGH | MEDIUM",
      "confidence": "CONFIRMED | WEAK_SIGNAL",
      "trend_data": ["string"],
      "recommended_action": "string",
      "detected_at": "timestamp"
    }
  ],
  "data_sources": ["string"]
}
```

**Fallback (sin IA):** Job mecánico que compara scores vs umbrales y genera alertas sin síntesis de texto.
**Surface:** Widget "Alertas tempranas" en dashboard + badge sobre la entidad afectada en el mapa/lista.

---

##### AI-06 — Follow-Up Automation

| Campo | Valor |
|---|---|
| **Actividad** | Resumen inteligente de follow-ups pendientes |
| **Trigger** | Job diario + usuario abre vista de Follow-Ups |
| **Rol que consume** | Todos los roles con follow-ups asignados o en su scope |
| **Módulo** | Follow-Up System (5.6) |

**Context inputs inyectados:**
```
- user.id, user.role, user.scope
- overdue_items[]: id, descripción, owner, aging_days, severity, source_type, source_module
- due_soon_items[]: id, descripción, owner, days_to_deadline, severity
- stalled_items[]: items con status IN_PROGRESS pero sin actividad en >7 días
- escalation_candidates[]: items abiertos >2 ciclos sin resolución
```

**System prompt:**
```
Eres el asistente de seguimiento de GRIP.
Tu misión es que ningún hallazgo quede olvidado.
Genera un resumen claro y accionable del estado de los follow-ups,
priorizando lo que necesita atención inmediata.

Reglas:
- Agrupa por urgencia: vencidos > próximos a vencer > estancados.
- Para items candidatos a escalar, redacta un borrador de mensaje de escalación.
- El tono es ejecutivo y directo. Sin relleno.
- Responde en [tenant.locale].
- Responde SOLO en el JSON schema indicado.
```

**User prompt (template):**
```
Usuario: [user.name] ([user.role])

ITEMS VENCIDOS ([overdue_items.length]):
[overdue_items — serializado]

ITEMS POR VENCER PRONTO ([due_soon_items.length]):
[due_soon_items — serializado]

ITEMS ESTANCADOS ([stalled_items.length]):
[stalled_items — serializado]

CANDIDATOS A ESCALACIÓN ([escalation_candidates.length]):
[escalation_candidates — serializado]

Genera el resumen de seguimiento y borradores de escalación.
```

**Output schema:**
```json
{
  "summary_headline": "string (1 oración)",
  "overdue": [
    {
      "item_id": "string",
      "description": "string",
      "owner": "string",
      "aging_days": "integer",
      "action_required": "string"
    }
  ],
  "due_soon": [
    {
      "item_id": "string",
      "description": "string",
      "days_remaining": "integer"
    }
  ],
  "escalation_drafts": [
    {
      "item_id": "string",
      "draft_message": "string",
      "escalate_to": "string (role)"
    }
  ],
  "data_sources": ["string"]
}
```

**Fallback (sin IA):** Lista ordenada por deadline ascendente. Sin síntesis ni borradores.
**Surface:** Panel "Resumen de seguimiento" en la vista de Follow-Ups de cada rol.

---

##### AI-07 — Performance Synthesis (Resumen Ejecutivo Semanal)

| Campo | Valor |
|---|---|
| **Actividad** | Síntesis ejecutiva semanal / pre-Comité Operativo |
| **Trigger** | Job semanal automático (viernes) + solicitud manual |
| **Rol que consume** | COO / CEO |
| **Módulo** | Dashboard Ejecutivo (5.7) |

**Context inputs inyectados:**
```
- weekly_reports_last_4_weeks[]: todos los reportes en el scope del ejecutivo
- da_findings_last_quarter[]: hallazgos, estados, tasas de cierre por región
- three_point_scores_last_3_months[]: scores por región/área, tendencias
- followup_kpis[]: tasa de cierre global, aging promedio, items críticos abiertos
- kardex_kpis_consolidated[]: ventas, merma, inventario — resumen por región
- period: semana/mes a sintetizar
```

**System prompt:**
```
Eres el asistente de síntesis ejecutiva de GRIP.
Tu misión es generar el resumen que el CEO/COO necesita para
tomar decisiones y presidir el Comité Operativo.

Reglas:
- Destaca los 3 temas que más merecen atención ejecutiva.
- Identifica qué está funcionando bien (no todo es negativo).
- Señala patrones que cruzan regiones.
- Propón decisiones concretas, no solo observaciones.
- Máximo 1 página de lectura (equivalente en texto).
- Responde en [tenant.locale].
- Responde SOLO en el JSON schema indicado.
```

**User prompt (template):**
```
Ejecutivo: [user.name] ([user.role])
Período: [period]

WEEKLY REPORTS RECIBIDOS ([weekly_reports_last_4_weeks.length] reportes):
[weekly_reports_last_4_weeks — serializado, versión comprimida]

HALLAZGOS DE DA (último trimestre):
[da_findings_last_quarter — serializado]

SCORES 3-POINT (últimos 3 meses):
[three_point_scores_last_3_months — serializado]

KPIs KARDEX CONSOLIDADOS:
[kardex_kpis_consolidated — serializado]

Genera la síntesis ejecutiva del período.
```

**Output schema:**
```json
{
  "executive_headline": "string (1 oración que resume el período)",
  "highlights": [
    {
      "title": "string",
      "detail": "string",
      "type": "POSITIVE | CONCERN | CRITICAL"
    }
  ],
  "cross_regional_patterns": ["string"],
  "recommended_decisions": [
    {
      "decision": "string",
      "rationale": "string",
      "urgency": "IMMEDIATE | THIS_WEEK | THIS_MONTH"
    }
  ],
  "metrics_snapshot": {
    "followup_closure_rate": "string",
    "avg_checklist_score": "string",
    "open_critical_findings": "integer"
  },
  "data_sources": ["string"]
}
```

**Fallback (sin IA):** Dashboard con métricas crudas sin síntesis de texto.
**Surface:** Sección "Síntesis ejecutiva" en dashboard CEO/COO. También disponible como vista "Pre-Comité Operativo".

---

##### AI-08 — Variable Checklist Item Suggestions

| Campo | Valor |
|---|---|
| **Actividad** | Sugerencia de ítems variables para la próxima visita |
| **Trigger** | JZ accede a la sección de ítems variables al preparar o iniciar un checklist |
| **Rol que consume** | Jefe de Zona |
| **Módulo** | Store Visit Checklist (5.3) |

> **Nota:** AI-08 es una especialización de AI-01. Mientras AI-01 genera el briefing completo de preparación, AI-08 genera exclusivamente la lista de ítems variables sugeridos para insertar directamente en el checklist.

**Context inputs inyectados:**
```
- store.id, store.name
- last_5_checklists_variable_items[]: ítem, resultado (cumplió/no cumplió), fecha
- open_followups_for_store[]: descripción, categoría, aging
- peer_store_recurring_findings[]: findings frecuentes en tiendas similares de la zona (últimos 60 días)
- kardex_anomalies_for_store[]: métricas fuera de rango respecto a benchmark de zona
- da_findings_linked_to_store[]: hallazgos de DAs que aplican a esta tienda
```

**System prompt:**
```
Eres el asistente de visitas de tienda de GRIP.
Tu misión es sugerir ítems variables concretos y accionables
para que el Jefe de Zona no pierda hallazgos importantes durante su visita.

Reglas:
- Sugiere entre 3 y 5 ítems. No más.
- Cada ítem debe ser verificable en campo (observable físicamente o medible).
- Justifica cada sugerencia con datos del sistema. Sin justificación, no hay sugerencia.
- Prioriza: (1) follow-ups abiertos, (2) anomalías KPI, (3) patrones en tiendas pares.
- Formula cada ítem como una verificación accionable, no como una pregunta abierta.
- Responde en [tenant.locale].
- Responde SOLO en el JSON schema indicado.
```

**User prompt (template):**
```
Tienda: [store.name] ([store.id])

ÍTEMS VARIABLES ÚLTIMAS 5 VISITAS:
[last_5_checklists_variable_items — serializado]

FOLLOW-UPS ABIERTOS EN ESTA TIENDA:
[open_followups_for_store — serializado]

FINDINGS FRECUENTES EN TIENDAS PARES:
[peer_store_recurring_findings — serializado]

ANOMALÍAS KPI KARDEX:
[kardex_anomalies_for_store — serializado]

HALLAZGOS DE DAS APLICABLES:
[da_findings_linked_to_store — serializado]

Sugiere entre 3 y 5 ítems variables para la visita de hoy.
```

**Output schema:**
```json
{
  "suggested_items": [
    {
      "description": "string (formulado como verificación accionable)",
      "category": "string (sección del checklist: Precios | Neveras | Bodega | etc.)",
      "priority": "HIGH | MEDIUM | LOW",
      "justification": "string",
      "source_type": "OPEN_FINDING | KPI_ANOMALY | PEER_PATTERN | DA_FINDING",
      "source_reference": "string (id del finding/KPI que lo origina)"
    }
  ],
  "data_sources": ["string"]
}
```

**Fallback (sin IA):** Listar open follow-ups de la tienda como ítems variables sugeridos (sin síntesis ni priorización).
**Surface:** Panel "Ítems sugeridos por GRIP" en la sección de ítems variables del checklist. El JZ puede aceptar, editar o rechazar cada sugerencia con 1 tap.

---

##### Resumen de modelos de prompt

| ID | Actividad | Trigger | Roles | Fallback |
|---|---|---|---|---|
| `AI-01` | Store Visit Prep Brief | Abrir preparación de visita | Jefe de Zona | Open follow-ups crudos |
| `AI-02` | 3-Point Review Prep | Abrir formulario 3-Point | Gte. Ventas / Log. / Regional / COO | Aspectos no evaluados por fecha |
| `AI-03` | DA Prep Brief | Abrir formulario DA | COO / CEO | Hallazgos abiertos crudos |
| `AI-04` | Pattern Detection | Job semanal / manual | Gte. Ventas y superiores | Sin widget (datos disponibles via filtros) |
| `AI-05` | Early Warning | Job diario | Gte. Regional / COO / CEO | Alertas mecánicas por umbral |
| `AI-06` | Follow-Up Automation | Job diario / abrir Follow-Ups | Todos los roles | Lista ordenada por deadline |
| `AI-07` | Performance Synthesis | Job semanal / manual | COO / CEO | Dashboard con métricas crudas |
| `AI-08` | Variable Checklist Items | Sección ítems variables | Jefe de Zona | Open follow-ups como ítems sugeridos |

---

## 6. Data Model

### 6.1 Entity Relationship Overview

```
Tenant (1) ----< (N) Store
Tenant (1) ----< (N) Employee
Tenant (1) ----< (N) ChecklistTemplate
Tenant (1) ----< (N) AspectLibrary

Employee (1) ----< (N) Employee              [reports_to hierarchy]
Employee (1) ----< (N) Store                  [assigned_stores for DM]

Employee (1) ----< (N) WeeklyReport           [author]
Employee (1) ----< (N) ChecklistInstance       [executor]
Employee (1) ----< (N) ThreePointReview        [reviewer]
Employee (1) ----< (N) ThreePointReview        [reviewee]
Employee (1) ----< (N) DAAudit                 [executor]
Employee (1) ----< (N) FollowUpItem            [assigned_to]

Store (1) ----< (N) ChecklistInstance
ChecklistTemplate (1) ----< (N) ChecklistInstance
ChecklistInstance (1) ----< (N) ChecklistResponse
ChecklistResponse (1) ----< (N) Photo

AspectLibrary (1) ----< (N) Aspect
ThreePointReview (1) ----< (N) ReviewedAspect
Aspect (1) ----< (N) ReviewedAspect

DAAudit (1) ----< (N) DAFinding

-- Follow-Up is polymorphic via source_type + source_id
FollowUpItem.source -> ChecklistResponse | ThreePointReview | DAAudit | WeeklyReport | null (manual)
```

### 6.2 Store Entity

```
Store {
  id: UUID
  tenant_id: UUID
  store_code: String (unique per tenant) // e.g., "T002"
  name: String // e.g., "CACIQUE"
  display_name: String // "CACIQUE T002"
  address: Text
  city: String
  region: String
  country_code: String // "DO", "CO", etc.
  status: Enum [ACTIVE, INACTIVE, TEMPORARILY_CLOSED]
  assigned_dm_id: UUID (FK -> Employee)
  area_id: UUID (FK -> Area) // nullable
  latitude: Decimal
  longitude: Decimal
  created_at: Timestamp
  updated_at: Timestamp
}

Area {
  id: UUID
  tenant_id: UUID
  name: String
  parent_area_id: UUID (FK -> Area) // for nested areas
  responsible_id: UUID (FK -> Employee) // SM or VP
}
```

### 6.3 Notification Model

```
Notification {
  id: UUID
  tenant_id: UUID
  recipient_id: UUID (FK -> Employee)
  type: Enum [
    WEEKLY_SUBMITTED,
    FOLLOWUP_ASSIGNED,
    FOLLOWUP_OVERDUE,
    FOLLOWUP_RESOLVED,
    FOLLOWUP_ESCALATED,
    DA_SENT,
    DA_MEETING_SCHEDULED,
    REVIEW_SCHEDULED,
    AI_EARLY_WARNING,
    CHECKLIST_COMPLETED
  ]
  title: String
  body: Text
  reference_type: String // entity type
  reference_id: UUID // entity id
  read: Boolean
  read_at: Timestamp
  created_at: Timestamp
}
```

---

## 7. Integration Specifications

### 7.1 KARDEX Integration

#### 7.1.1 Purpose

Alimentar GRIP con KPIs operacionales provenientes del sistema KARDEX (ERP/Inventario).

#### 7.1.2 Data Points Required

| KPI | Granularity | Frequency | Usage in GRIP |
|---|---|---|---|
| Sales (daily/weekly) | Per store | Daily sync | Dashboard, AI patterns |
| Inventory days | Per store, per category | Daily sync | Dashboard, AI early warning |
| Shrinkage / Merma | Per store | Weekly sync | Dashboard, AI pattern detection |
| Cash reconciliation | Per store | Daily sync | Dashboard, follow-up trigger |
| Out-of-stock rate | Per store, per SKU | Daily sync | Dashboard, checklist context |

#### 7.1.3 Integration Architecture

```
KARDEX System
     |
     | REST API / File-based (CSV/JSON)
     |
     v
+------------------+
| KARDEX Adapter   |  <- Adapter pattern for future ERP swaps
| - Data mapping   |
| - Validation     |
| - Error handling |
+------------------+
     |
     v
+------------------+
| KPI Data Store   |  <- Normalized, tenant-scoped
| (PostgreSQL)     |
+------------------+
```

#### 7.1.4 Integration Requirements

| ID | Requirement | Priority |
|---|---|---|
| `INT-01` | Sync diario de KPIs desde KARDEX | Must |
| `INT-02` | Adapter pattern para desacoplar del ERP especifico | Must |
| `INT-03` | Manejo de errores y reintentos con logging | Must |
| `INT-04` | Dashboard de salud de integracion para Admin | Should |
| `INT-05` | Soporte para importacion manual (CSV) como fallback | Should |

---

## 8. Non-Functional Requirements

### 8.1 Performance

| Metric | Target | Rationale |
|---|---|---|
| Page load time | < 2s (3G connection) | DMs en tienda con conexion movil |
| Checklist item tap response | < 100ms | UX tap-based requiere respuesta instantanea |
| Dashboard load | < 3s | Carga inicial con datos pre-agregados |
| API response time (p95) | < 500ms | Para operaciones CRUD estandar |
| Concurrent users | 500+ simultaneous | Para operaciones de 1000+ tiendas |

### 8.2 Scalability

| Dimension | Target |
|---|---|
| Tiendas por tenant | Hasta 5,000 |
| Empleados por tenant | Hasta 10,000 |
| Checklists por mes | Hasta 50,000 (1 diario x 1000 tiendas x 5 dias/semana aprox) |
| Follow-up items activos | Hasta 100,000 |
| Historico retencion | 5 anos minimo |

### 8.3 Availability

| Metric | Target |
|---|---|
| Uptime | 99.5% |
| Planned maintenance window | Domingo 2:00-4:00 AM local |
| RPO (Recovery Point Objective) | 1 hora |
| RTO (Recovery Time Objective) | 4 horas |

### 8.4 Security

| Requirement | Detail |
|---|---|
| Authentication | JWT-based, session management |
| Authorization | Role-based (RBAC) + hierarchical scope |
| Data encryption | TLS 1.3 in transit, AES-256 at rest |
| Tenant isolation | Schema-level in PostgreSQL |
| Audit trail | All write operations logged with user, timestamp, before/after |
| Password policy | Configurable per tenant |
| Session timeout | Configurable (default: 8 hours for mobile, 2 hours for web) |

### 8.5 Internationalization

| Requirement | Detail |
|---|---|
| Languages (MVP) | Spanish (es-DO) |
| Languages (future) | Portuguese (pt-BR), English (en-US), German (de-DE) |
| Date format | Per locale |
| Timezone | Per employee (stored in profile) |
| Currency | Per tenant configuration |
| UI framework | i18n-ready from day 1, all strings externalized |

### 8.6 Accessibility & UX

| Requirement | Detail |
|---|---|
| Responsive breakpoints | Mobile (375px+), Tablet (768px+), Desktop (1280px+) |
| Touch targets | Minimum 44x44px (WCAG) |
| Offline support | Checklist module (Phase 2) |
| Browser support | Chrome, Safari, Firefox (latest 2 versions) |
| PWA | Installable as home screen app |

---

## 9. Configuration & Multi-Tenancy

### 9.1 Tenant Configuration Model

```
TenantConfig {
  tenant_id: UUID

  // Branding
  company_name: String
  logo_url: String
  primary_color: String // hex

  // Operational Config
  checklist_template_id: UUID // active template
  aspect_library_id: UUID // active library
  aspects_per_review: Integer // default: 3
  da_frequency: Enum [MONTHLY, QUARTERLY, BIANNUAL, ANNUAL]

  // Follow-Up Config
  followup_auto_escalation_hours: Integer // default: 24
  followup_overdue_notification: Boolean // default: true

  // Integration
  kardex_api_url: String
  kardex_api_key: String (encrypted)
  kardex_sync_schedule: String // cron expression

  // Locale
  default_locale: String // "es-DO"
  default_timezone: String // "America/Santo_Domingo"
  supported_locales: String[] // ["es-DO", "es-CO"]

  // Feature Flags
  ai_assistant_enabled: Boolean
  offline_mode_enabled: Boolean
  photo_capture_enabled: Boolean
}
```

### 9.2 Multi-Tenancy Strategy

| Aspect | Strategy |
|---|---|
| **Data isolation** | Schema-per-tenant in PostgreSQL |
| **Authentication** | Shared auth service, tenant claim in JWT |
| **URL routing** | Subdomain-based: `{tenant}.grip.app` |
| **Configuration** | Per-tenant config table |
| **AI models** | Shared model, tenant-scoped data context |
| **File storage** | Tenant-prefixed paths in object storage |

### 9.3 MVP Simplification

Para el MVP con un solo tenant:
- Schema unico con `tenant_id` en todas las tablas (preparado para multi-tenancy)
- Sin subdomain routing (URL fija)
- Configuracion en tabla, no en archivos
- AI con modelo compartido

---

## 10. Phasing Strategy

### Phase 1 - MVP (Foundation + Core Control)

**Objetivo**: Sistema funcional con los mecanismos de control base y AI integrada.
**Scope organizacional MVP**: Cadena de Ventas (tiendas) + Cadena de Logistica (CEDI). Ambas ramas operativas completas.

| Module | Scope |
|---|---|
| Employee Database | CRUD, hierarchy flexible (reports_to), Role configurable, store/CEDI assignment |
| Store & CEDI Registry | CRUD de tiendas y centros de distribucion, area assignment |
| Store Visit Checklist | Template config, execution, fixed + variable items, photos, score |
| Weekly Reports | Create por nivel, submit, cascade visibility (5 niveles), flag for follow-up |
| Follow-Up System | Full lifecycle, auto-creation from checklist + weekly, escalation, overdue alerts |
| 3-Point Review | Aspect library, review execution por cada manager con sus reportes directos, coverage tracking |
| Dashboards | Jefe de Zona, Gte. Ventas, Gte. Logistico, Gerente Regional, COO (role-based) |
| AI Assistant | Store visit prep, 3-Point review prep, follow-up summary, basic natural language queries |
| KARDEX Integration | Basic sync (sales, inventory, shrinkage) + KPIs logisticos (tiempos recibo/picking/despacho) |
| Auth & Config | Login, RBAC configurable, tenant config, roles per tenant |

### Phase 2 - Extended Control & Executive Layer

| Module | Scope |
|---|---|
| DA Module | Full DA workflow (CEO all areas + COO ops only), finding management, meeting scheduling |
| Dashboards | CEO dashboard, 36-point coverage widget, DA status views |
| AI Assistant | DA prep, pattern detection cross-company, early warnings, performance synthesis |
| Offline Mode | Checklist offline execution + sync |
| CEDI Checklist | Checklist configurable para operaciones de CEDI (analogo a Store Visit Checklist) |

### Phase 3 - Intelligence & Scale

| Module | Scope |
|---|---|
| AI Advanced | Cross-company patterns, learning loop, performance synthesis |
| Multi-Tenancy | Full tenant onboarding, subdomain routing, isolated schemas |
| Advanced Dashboards | Custom widgets, export, scheduling |
| Integrations | Additional ERP adapters, email notifications, calendar integration |
| i18n | Additional languages |

---

## 11. Acceptance Criteria

### 11.1 System-Level Acceptance

| ID | Criteria | Validation |
|---|---|---|
| `AC-SYS-01` | Un Jefe de Zona puede completar un checklist de 30+ items en menos de 10 minutos | Usability test con 5 Jefes de Zona reales |
| `AC-SYS-02` | Ningun finding se pierde: todo "No Cumple" genera follow-up rastreable | Automated test: create checklist with non-compliant -> verify follow-up exists |
| `AC-SYS-03` | Un SM puede ver el estado de follow-ups de todo su equipo en un solo dashboard | SM login -> dashboard shows aggregated follow-ups |
| `AC-SYS-04` | La AI sugiere items variables relevantes basados en datos historicos | Create history -> request AI prep -> verify suggestions reference actual data |
| `AC-SYS-05` | El sistema escala automaticamente follow-ups vencidos | Create follow-up with past deadline -> verify escalation notification |
| `AC-SYS-06` | Los KPIs de KARDEX se reflejan en dashboards dentro de las 24h | Sync trigger -> verify KPI data in dashboard |
| `AC-SYS-07` | Todo el UI es funcional en smartphone (375px) | Responsive test across all modules |
| `AC-SYS-08` | La jerarquia de visibilidad es estricta: nadie ve datos fuera de su scope | Role-based access test matrix |

### 11.2 Module-Level Acceptance Summary

| Module | Key Acceptance Criteria |
|---|---|
| Employee DB | Hierarchy validated, no orphan nodes, store assignment enforced |
| Weekly Report | One per DM per week, cascade visibility correct, flag creates follow-up |
| Checklist | Template configurable, < 10 min execution, photos saved, score calculated |
| 3-Point Review | 3 aspects selected, coverage tracked, BELOW generates follow-up |
| DA | DRAFT->SENT->MEETING->COMPLETED flow, findings become follow-ups |
| Follow-Up | 0 forgotten findings: every source creates trackable item with lifecycle |
| Dashboards | Role-correct data, drill-down works, mobile-friendly |
| AI | Preparation reduces manual prep time by 50%+, suggestions are data-backed |

---

## Appendix A: Checklist Template - Default Sections Detail

Basado en el formato real "Checklist Jefe de Zona - RITMO":

### Section 1: Exterior
- Poster de productos actualizado
- Parqueadero libre de basura
- Puertas limpias y sin residuos de cinta

### Section 2: Limpieza de la tienda
- Piso de venta barrido y trapeado
- Gondolas sin polvo, sin cinta, alineadas
- Estibas en buen estado y alineadas
- Paredes y techos limpios, sin humedad, sin cintas
- Extintor limpio y con fechas revisadas

### Section 3: Exhibicion de Mercancia
- Productos frenteados y alineados
- Corte de cajas uniforme segun contenido

### Section 4: Precios
- Todos los productos con etiqueta de precio
- No voltear precio ni retirar fleje de producto agotado
- Retirar precios de productos descontinuados

### Section 5: Neveras y congeladores
- Limpieza, organizacion e iluminacion
- Rotacion PEPS correcta

### Section 6: Puesto de pago
- Escanear todos los productos (no teclear)
- Verificar mercancia camuflada
- Preguntar si necesita comprobante fiscal
- Cajeros confrontan precios con visor
- Limpieza periodica de scanner y cajones

### Section 7: Bodega
- Marcada por bloque
- Almacenamiento por rotacion, peso y tamano
- Exceso de mercancia vs dias de inventario
- Banos limpios con insumos
- Cocina organizada y limpia

### Section 8: Documentos tienda
- Planilla de destruccion
- Seguimiento de agotados actualizado
- Reporte de merma actualizado
- Planillas de pedido actualizadas e impresas
- Horarios impresos
- Distribucion de tareas actualizada e impresa
- Seguimiento de vencimientos
- Distribucion de bloque actualizado e impreso

### Section 9: Empleados
- Uniforme segun manual
- Atencion al cliente: saludo y despedida

---

## Appendix B: API Endpoints Overview (MVP)

```
# Auth
POST   /api/auth/login
POST   /api/auth/refresh
POST   /api/auth/logout

# Employees
GET    /api/employees
POST   /api/employees
GET    /api/employees/:id
PUT    /api/employees/:id
GET    /api/employees/:id/profile     # full profile with history
GET    /api/employees/:id/team        # direct reports

# Stores
GET    /api/stores
POST   /api/stores
GET    /api/stores/:id
PUT    /api/stores/:id
GET    /api/stores/:id/checklists     # checklist history
GET    /api/stores/:id/followups      # open follow-ups

# Checklist Templates
GET    /api/checklist-templates
POST   /api/checklist-templates
GET    /api/checklist-templates/:id
PUT    /api/checklist-templates/:id

# Checklist Instances
GET    /api/checklists
POST   /api/checklists                # start new checklist
GET    /api/checklists/:id
PUT    /api/checklists/:id            # update responses
POST   /api/checklists/:id/complete   # finalize
POST   /api/checklists/:id/responses  # add/update response
POST   /api/checklists/:id/photos     # upload photo

# Weekly Reports
GET    /api/weekly-reports
POST   /api/weekly-reports
GET    /api/weekly-reports/:id
PUT    /api/weekly-reports/:id
POST   /api/weekly-reports/:id/submit
POST   /api/weekly-reports/:id/flag

# Follow-Up Items
GET    /api/followups
POST   /api/followups                 # manual creation
GET    /api/followups/:id
PUT    /api/followups/:id/status      # transition status
POST   /api/followups/:id/resolve
POST   /api/followups/:id/verify
POST   /api/followups/:id/escalate
GET    /api/followups/overdue         # overdue items
GET    /api/followups/summary         # aggregated stats

# 3-Point Reviews (Phase 2)
GET    /api/reviews
POST   /api/reviews
GET    /api/reviews/:id
PUT    /api/reviews/:id
GET    /api/reviews/coverage/:employee_id

# DA (Phase 2)
GET    /api/da-audits
POST   /api/da-audits
GET    /api/da-audits/:id
PUT    /api/da-audits/:id
POST   /api/da-audits/:id/send
POST   /api/da-audits/:id/findings

# Dashboards
GET    /api/dashboards/me             # role-based dashboard data
GET    /api/dashboards/kpis           # KPI widgets

# AI Assistant
POST   /api/ai/chat                   # conversational query
POST   /api/ai/prep/store-visit       # store visit preparation
POST   /api/ai/prep/review            # 3-point review preparation
GET    /api/ai/alerts                  # early warning alerts
GET    /api/ai/patterns                # detected patterns

# Notifications
GET    /api/notifications
PUT    /api/notifications/:id/read
GET    /api/notifications/unread-count

# Admin / Config
GET    /api/config
PUT    /api/config
GET    /api/config/aspect-library
PUT    /api/config/aspect-library
```

---

*End of Document*

*Generated following SDD (Specs Driven Development) methodology: specifications first, implementation follows.*
