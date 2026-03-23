# Ingesta PDF + IA (M1) v1

## 1. Objetivo

- **Aplicación(es):** `grip-backend` y `grip-frontend`.
- **Problema o necesidad:** Agregar un canal adicional de ingesta para `jefe_zona` mediante PDF digital, manteniendo el flujo actual JSON y aplicando el mismo output esperado de IA para cabecera y hallazgos checklist.
- **Alcance:**
  - **Backend:** nuevos endpoints async para upload PDF, consulta de estado y confirmación de persistencia; extracción IA con degradación controlada.
  - **Frontend:** carga de PDF, pantalla/estado de preview y confirmación sin edición manual.
  - **Permisos:** solo `jefe_zona`.
  - **Fuera de alcance V1:** OCR de escaneados, edición manual de campos extraídos, antivirus.

## 2. Referencia a specs base

- Reglas funcionales de M1 y filosofía de control: [functional-spec.md](../functional-spec.md) §2.1 y §6.
- Contrato/arquitectura base: [technical-spec.md](../technical-spec.md) §2 (M1), §3.A (Extractor) y §3.F (resiliencia).
- Reglas R1-R6 (sin excepciones): [technical-spec.md](../technical-spec.md) §4 y [functional-spec.md](../functional-spec.md) §6.0.
- SSOT de contrato backend: [api-contract-ssot.md](../api-contract-ssot.md).

## 3. Actores y permisos

- **Rol habilitado:** `jefe_zona`.
- **Operaciones permitidas para `jefe_zona`:**
  1. subir PDF de visita,
  2. consultar estado de extracción,
  3. confirmar persistencia del preview.
- **Denegado para otros roles:** upload/confirm del canal PDF.

## 4. Flujos principales

1. `jefe_zona` selecciona PDF y envía.
2. API valida tipo de archivo, tamaño (<= 1MB) y páginas (<= 2).
3. API crea job asíncrono de extracción y retorna `job_id`.
4. IA extrae datos de cabecera + hallazgos checklist y genera enriquecimiento esperado.
5. Cliente consulta estado del job hasta `preview_ready`.
6. Usuario revisa preview (sin edición) y confirma.
7. API persiste visita/hallazgos desde el snapshot `preview_ready` y registra resultado de extracción.
8. El PDF original queda en object storage con retención de 1 año.

### Excepciones

- Si IA no completa todos los campos: **guardar parcial + advertencias**.
- Si archivo excede límites: rechazo de validación sin crear job.

## 5. Reglas de negocio

1. Canal PDF es adicional; el canal JSON actual de M1 se mantiene.
2. Solo se admiten PDFs de texto digital.
3. Límite V1: máximo 1MB y máximo 2 páginas.
4. La extracción debe mapear los mismos campos de:
   - cabecera (`store_code`, `jz_email`, `visit_date`),
   - checklist por ítem (`category`, `item`, `status`, `comment`).
5. El sistema conserva reglas R1-R6 sin excepciones.
6. Ante extracción incompleta, persistir parcial con `warnings` explícitos.
7. No hay edición manual del preview en V1; solo confirmar o descartar/reintentar.
8. Se debe guardar trazabilidad IA: `confidence`, `warnings`, campos faltantes.

## 6. Contrato API

### 6.1 Nuevo endpoint: crear job de extracción

- **POST** `/api/v1/ingest/pdf`
- **Content-Type:** `multipart/form-data`
- **Request fields (V1 implementada):**
  - `file`: PDF (único campo multipart soportado por la API en esta versión).
- **Diferido (post-V1):** validación cruzada enviando `store_code` adicional en el multipart **no está implementada**; el `store_code` del flujo proviene de la extracción IA en el preview (`result.header.store_code`). Si se reintroduce en el contrato, alinear [technical-spec.md](../technical-spec.md) §2 M1 y `app/api/ingestion.py`.
- **Response 202:**

```json
{
  "job_id": "UUID",
  "status": "queued",
  "limits": { "max_size_mb": 1, "max_pages": 2 }
}
```

### 6.2 Nuevo endpoint: consultar estado y preview

- **GET** `/api/v1/ingest/pdf/{job_id}`
- **Response 200 (ejemplo preview_ready):**

```json
{
  "job_id": "UUID",
  "status": "preview_ready",
  "result": {
    "header": {
      "store_code": "STRING",
      "jz_email": "STRING",
      "visit_date": "ISO8601_TIMESTAMP"
    },
    "raw_content": [
      { "category": "STRING", "item": "STRING", "status": true, "comment": "STRING" }
    ],
    "ai_output": [
      {
        "summary": "STRING",
        "severity_rank": "low|medium|high",
        "suggested_action": "STRING",
        "confidence": 0.0
      }
    ],
    "warnings": ["STRING"],
    "missing_fields": ["STRING"]
  }
}
```

### 6.3 Nuevo endpoint: confirmación de persistencia

- **POST** `/api/v1/ingest/pdf/{job_id}/confirm`
- **Request:**

```json
{ "confirm": true }
```

- **Response 200:**

```json
{
  "status": "confirmed",
  "visit_id": "UUID",
  "findings_created": 0,
  "warnings": ["STRING"]
}
```
- **Regla de eficiencia/costo:** `confirm` no reconsulta IA; persiste exactamente el snapshot ya calculado en `preview_ready` (determinismo preview=confirm).

### 6.4 Estados del job

- `queued`
- `processing`
- `preview_ready`
- `confirmed`
- `failed`

## 7. Cambios en UI (frontend)

- En flujo de M1, añadir opción de carga de PDF.
- Mostrar progreso/estado del job asíncrono.
- Mostrar preview extraído (solo lectura) con advertencias de extracción.
- Confirmar persistencia final desde preview.
- En confirmación exitosa, mantener patrón de UX de redirección al dashboard principal.

## 8. Criterios de aceptación

1. **Dado** un `jefe_zona` autenticado con PDF válido, **cuando** envía el archivo, **entonces** se crea un job async y retorna `job_id`.
2. **Dado** un PDF > 1MB o > 2 páginas, **cuando** se intenta subir, **entonces** el sistema rechaza la solicitud por validación.
3. **Dado** un job en `preview_ready`, **cuando** el usuario confirma, **entonces** se persisten visita y hallazgos con output IA esperado.
4. **Dado** una extracción incompleta, **cuando** la IA no puede completar todos los campos, **entonces** se guarda parcial y se exponen advertencias.
5. **Dado** un usuario distinto de `jefe_zona`, **cuando** intenta usar el canal PDF, **entonces** recibe denegación por permisos.
6. **Calidad objetivo V1:** exactitud mínima de extracción del 95% en el dataset objetivo de validación definido por negocio.
7. **Caso mínimo obligatorio:** happy path end-to-end.

## 9. Notas

- El archivo PDF original debe almacenarse en object storage con retención de 1 año.
- Mantener Gemini como proveedor IA para esta feature.
- La implementación debe conservar resiliencia IA del sistema (degradación sin bloqueo total del flujo de negocio).
