# 📘 Especificación Funcional Maestra: Proyecto GRIP (Fase 1)

**Estado:** Final y Definitiva (Ultra-Completa)  
**Metodología:** Spec-Driven Development (SDD)  
**Concepto Central:** *Trust Through Control* (Confianza a través del Control)  
**Alineación:** Revisión R1–R6 / SDD 2025-03-21 · Detalle técnico: [technical-spec.md §4](technical-spec.md) · Audit: [backend_audit_report.md](backend_audit_report.md)

---

## 1. Modelo de Gobernanza y Jerarquía
GRIP establece una cadena de mando digital donde la información fluye sin filtros y la supervisión es multinivel. El sistema protege la integridad de los datos evitando que los niveles intermedios "maquillen" los reportes.

| Nivel | Actor | Responsabilidad Clave | Control y Asistencia de IA |
| :--- | :--- | :--- | :--- |
| **5** | **CEO** | **Dienstaufsicht (DA)** sobre Regionales. | Análisis de causa raíz sistémica y redacción de DA. Acceso al Generador de Resumen Semanal por IA. |
| **4** | **COO** | Análisis de Weekly Regional + 3 Preguntas. | Detección de sesgos y contradicciones en reportes. Acceso al Generador de Resumen Semanal por IA. |
| **3** | **Gerente Regional** | Estrategia de Zona y Sustentación de DA. | Consolidación de riesgos zonales e históricos. Acceso al **Generador de Resumen Semanal por IA**, que consolida múltiples hallazgos de una tienda en Puntos Focales y Resumen Ejecutivo. |
| **2** | **Gerente de Ventas** | Weekly Zonal y Focal Points. | Resumen ejecutivo de hallazgos críticos. |
| **1** | **Jefe de Zona (JZ)** | Ejecución de Checklist e Ingestión. | Extracción y clasificación técnica de datos de campo. |

---

## 2. Definición Funcional de Módulos y Lógica de Inteligencia Artificial

Esta sección describe la función de cada módulo y detalla exactamente cómo la IA asiste al criterio ejecutivo en cada punto de contacto.

### 2.1. Módulo 1: Ingestión y Clasificación (The Data Factory / The Extractor)
* **Función:** Procesar el Checklist (PDF/Excel) cargado por el Jefe de Zona.
* **Input IA:** Texto crudo extraído de las celdas "Anotaciones" del PDF/Excel y el estado (Cumple/No Cumple).
* **Interfaz de Usuario (Formulario de Ingesta - JZ):**
    * **Datos de cabecera obligatorios:**
        * `Código de tienda` (selector o campo de texto validado contra el catálogo de tiendas).
        * `Email del JZ` (correo corporativo del Jefe de Zona).
        * `Fecha de visita` (selector de fecha; debe corresponder a la visita en curso).
    * **Datos del hallazgo (Checklist) por cada ítem evaluado:**
        * `Categoría` (ej.: Limpieza, Precios, Seguridad).
        * `Ítem evaluado` (texto del punto de control del checklist).
        * `Estado` (selector binario **Cumple / No Cumple**).
        * `Comentarios` (campo de texto libre donde el JZ describe el hallazgo).
* **Regla de UX del Formulario de Ingesta:**
    * Al **enviar el formulario exitosamente**, el usuario JZ **debe ser redirigido automáticamente al Dashboard Principal** (pantalla de resumen generado por la IA) para visualizar inmediatamente el resultado de la ingesta (hallazgos clasificados y resúmenes).
* **Lógica de Procesamiento IA:**
    * **Normalización:** Convierte lenguaje informal en categorías estandarizadas (ej: mapea "piso manchado" a la categoría "Limpieza").
    * **Detección de Entidades:** Identifica objetos específicos afectados (ej: "Maizena", "Fachada", "Cajas").
    * **Análisis de Gravedad:** Clasifica el hallazgo en *Baja, Media o Alta* severidad basado en palabras clave de riesgo operativo (seguridad, dinero, imagen legal).
* **Output IA:** Un objeto estructurado con: `Categoría`, `Severidad`, `Resumen Ejecutivo del Hallazgo` y `Accionable Sugerido`.

### 2.2. Módulo 2: Motor de Reincidencias y Follow-Up (The Guardian / The Matcher)
* **Función:** Asegurar el cierre de tareas operativas requiriendo siempre evidencia obligatoria.
* **Interfaz de Usuario (Follow-up):** El usuario podrá interactuar con los hallazgos desde el Dashboard para cambiar su estado mediante un botón "Resolver" por cada hallazgo. Al seleccionar "Resolver", se abre un modal para actualizar el estado (in_progress, verified) y adjuntar evidencia si se marca como verified.
* **Capacidades del Dashboard JZ:** Visualización del historial completo de hallazgos, búsqueda en tiempo real, filtros combinados y códigos de color por estado.
* **Regla de negocio visual:** Añadir evidencia es OBLIGATORIO para marcar un hallazgo como resuelto ('verified'), pero es OPCIONAL si el usuario solo quiere actualizar el estado a 'in_progress'.
* **Input IA:** Hallazgo actual vs. Histórico de hallazgos de la misma tienda (`punto_control`).
* **Lógica de Procesamiento IA:**
    * **Semantic Matching:** Compara si el problema actual es el mismo que el anterior, aunque se haya escrito diferente (ej: "vidrios sucios" es semánticamente similar a "pegotes en la fachada").
    * **Cómputo de Persistencia:** Incrementa el contador de reincidencia si no hay evidencia de cierre válida en el sistema y el problema reaparece.
* **Output IA:** Flag de `Reincidencia: TRUE/FALSE` y alerta visual de "Problema Sistémico" enviada a la gerencia si el contador es >= 2.

### 2.3. Módulo 3: Weekly Hub - Focal Points (The Decision Center / The Summarizer)
* **Función:** Interfaz de visualización de visitas para Jefes de Zona, Gerentes de Ventas y Regionales. Los roles superiores (Regional, COO, CEO) tienen acceso a un **Generador de Resumen Semanal** impulsado por IA que consolida múltiples hallazgos de una tienda en Puntos Focales (3 bullets) y un Resumen Ejecutivo breve.
* **Input IA:** Todos los hallazgos de la zona, estados del Follow-up y notas semanales del Gerente de Ventas.
* **Lógica de Procesamiento IA:**
    * **Síntesis Ejecutiva:** Condensa entre 40 y 50 hallazgos en los 3 puntos más críticos para presentar al Gerente Regional/CEO.
    * **Detección de Sesgos:** Compara el reporte escrito por el Gerente con la data cruda del sistema y resalta discrepancias (ej: "El Gerente reporta una mejora general, pero la IA detecta 5 reincidencias críticas nuevas en su zona").
* **Output IA:** **Focal Points** (3 bullets de texto claro con instrucciones) y un **Executive Brief** automático para guiar el inicio de la reunión Weekly.

### 2.4. Módulo 4: Ask GRIP - Memoria Organizacional (RAG) (The Analyst)
* **Función:** Asistente conversacional impulsado por IA que permite a los gerentes hacer preguntas en lenguaje natural sobre el rendimiento y los hallazgos de las tiendas. Base de conocimientos sobre la historia operativa y las decisiones previas de la empresa.
* **Interfaz de Usuario (Ask GRIP / Chat):** Selector opcional de tienda (`store_code`). Historial de mensajes con burbujas diferenciadas (usuario / IA). Input de texto con botón de enviar. Indicador de carga mientras la IA procesa.
* **Input IA:** Consultas en lenguaje natural de la Alta Gerencia (ej: "¿Por qué fallamos recurrentemente en la Zona Norte?", "¿Qué hallazgos críticos tiene CACIQUE T002?").
* **Lógica de Procesamiento IA:** Búsqueda vectorial sobre el histórico de actas Weekly, hallazgos verificados y DAs anteriores para encontrar patrones. Opcionalmente filtrado por tienda (`store_code`).
* **Output IA:** Respuestas contextuales basadas en evidencia dura (ej: "La Zona Norte falla en 'Seguridad' debido a la alta rotación reportada en el feedback de Q3, no por falta de presupuesto").

### 2.5. Módulo 5: DA - Dienstaufsicht (El Acto de Gobierno / Deep Analyst)
* **Función:** Intervención estructural obligatoria y documentada del CEO ante fallas del Gerente Regional.
* **Input IA:** Expediente completo de incumplimientos de la tienda/zona y las respuestas históricas del Regional a las "3 Preguntas" del COO.
* **Lógica de Procesamiento IA:**
    * **Análisis de Causa Raíz (5 Whys):** Sugiere al CEO posibles causas estructurales analizando contradicciones o negligencias pasadas (ej: "El problema no es la limpieza, es la falta de insumos reportada por el Jefe de Tienda hace 2 meses y omitida por el Regional").
    * **Predictor de Impacto:** Estima el riesgo de negocio si el DA no se resuelve (riesgo de multa, pérdida de inventario).
* **Output IA:** Borrador inicial del **Documento DA** con las secciones de "Hechos Probados", "Análisis Sugerido" y un "Plan de Acción Estructural".

---

## 3. Flujo de Comunicación Estratégica (The Feedback Loop)

El sistema fuerza la interacción entre jerarquías mediante este ciclo continuo:

1. **Top-Down (COO/CEO):** El COO revisa el resumen del Weekly Regional y le aplica el **Esquema de 3 Preguntas** (*¿Qué pasó?, ¿Por qué pasó?, ¿Qué cambiamos?*). El CEO monitorea la integridad de toda la cadena y emite DAs si detecta complacencia.
2. **Middle-Out (GV/GR):** El Gerente de Ventas traduce la presión de arriba definiendo **Focal Points** específicos para la próxima visita a tienda basándose en la data del Weekly Hub.
3. **Bottom-Up (JZ):** Antes de iniciar un nuevo checklist, el Jefe de Zona visualiza obligatoriamente en su pantalla de **Pre-Visita** los Focal Points de su jefe y las reincidencias históricas pendientes de esa tienda.



---

## 4. Ingeniería de Prompts (Reglas de Comportamiento IA)

Para asegurar que los modelos de lenguaje (LLMs) actúen como un "peer" corporativo implacable y no como un asistente virtual genérico, se establecen estas reglas de tono innegociables:

| Componente IA | Meta del Prompt (Goal) | Tono y Restricción de Personalidad |
| :--- | :--- | :--- |
| **Ingestión (M1)** | Extraer datos sin interpretar; solo categorizar y asignar severidad técnica. | Técnico, neutro, conciso. Prohibido usar adjetivos innecesarios. |
| **Weekly Hub (M3)** | Resumir para un ejecutivo ocupado. Ir directo al grano y resaltar el riesgo. | Ejecutivo, directo. Enfocado en impacto financiero y operativo. |
| **Memoria RAG (M4)** | Conectar puntos ciegos entre diferentes tiendas y fechas. Buscar patrones. | Analítico, consultor. Basado estrictamente en la evidencia recuperada. |
| **DA (M5)** | Cuestionar la operación e identificar por qué fallaron los controles previos. | Crítico, inquisitivo, autoritario. Perspectiva de Alta Dirección. |

---

## 5. Anexos de Contrato de Datos Funcionales

Para que la experiencia de usuario (UX) coincida con la estructura de datos, el flujo debe soportar la siguiente visualización mínima:

* **Header de Visita:** `Código de Tienda`, `Fecha`, `Nombre JZ`, `Nombre GV`, `Nombre GR`.
* **Cuerpo (Checklist):** Porcentaje de cumplimiento por categoría (0-100%).
* **Action Items (Hallazgos):** Texto de observación, URL de la evidencia fotográfica, Estado (Abierto/Cerrado), Flag de Reincidencia.
* **Feedback Estratégico:** Cajas de texto para Focal Points y para las respuestas a las 3 Preguntas.

---

## 6. Reglas Funcionales Innegociables (Hard Rules)

Para preservar la filosofía de *Trust Through Control*, el desarrollo debe respetar rigurosamente estas limitantes operativas.

### 6.0 Índice R1–R6 (identificadores estables / SDD)

Los IDs **R1–R6** son la referencia canónica entre producto, spec técnica y código. La **redacción normativa y los endpoints** viven en [technical-spec.md §4](technical-spec.md); aquí se resume el *por qué* funcional.

| ID | Título (resumen) | Intención de negocio |
| :--- | :--- | :--- |
| **R1** | Bloqueo de cierre sin evidencia | No se puede dar por cerrado un hallazgo sin prueba documentada. |
| **R2** | Persistencia severidad en `reopened` | Reabrir eleva la severidad para no minimizar el riesgo. |
| **R3** | Visibilidad directa CEO | El CEO no queda limitado por filtros de región al consolidar (anti-sesgo). |
| **R4** | Bloqueo por DA activo | Con DA activo en tienda, no se permiten cierres estándar de hallazgos que eludan el gobierno. |
| **R5** | Escalamiento de firma (MFA) | El hash de firma del CEO debe exigir MFA cuando esté integrado. |
| **R6** | Memoria post-DA (RAG) | Al cerrar el DA, el plan/resolución entra en la memoria consultable (RAG). |

**Mantenimiento SDD:** cualquier cambio en R1–R6 se actualiza en [technical-spec.md §4](technical-spec.md) y en esta tabla en el mismo cambio (o PR).

### 6.1 Narrativa (hard rules en lenguaje de producto)

1. **Transparencia Radical (alineado con R3):** El CEO debe poder hacer un "Drill-down" ininterrumpido desde el reporte macro de una región hasta ver la foto original cargada por el Jefe de Zona. Ningún filtro de nivel medio puede ocultar data a nivel L5. *Implementación backend actual del bypass CEO:* resumen semanal (`GET /api/v1/weekly/summary`); la lista de hallazgos puede añadir el mismo criterio en evoluciones futuras.
2. **Bloqueo por Evidencia (R1):** La interfaz de usuario no permitirá dar por "Verificado/Cerrado" un hallazgo operativo si el usuario no ha adjuntado un archivo o foto como prueba de resolución.
3. **Exclusividad del DA:** Solo los usuarios con rol de `CEO` verán el botón y tendrán permiso para crear, emitir y cerrar el flujo de una intervención `Dienstaufsicht` (coherente con Módulo 5 técnico).
4. **Inmutabilidad de Reincidencias:** Si el motor de IA detecta un problema persistente, el Jefe de Zona o Gerente de Ventas no pueden "desmarcar" la etiqueta de reincidencia manualmente; deben resolver el problema de raíz y sustentarlo.
