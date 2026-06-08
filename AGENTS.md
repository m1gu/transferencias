# AGENTS.md — Proyecto Transferencias (Banco Bolivariano)

## Resumen

Sistema de call center automatizado para transferencias bancarias del Banco Bolivariano. Usa **n8n** como orquestador, **VAPI** para llamadas de voz IA, **Supabase (PostgreSQL)** como base de datos, **Altiva API** como CRM de reportes, y **OpenAI (GPT-5.2/GPT-4o)** para clasificación y manejo de objeciones.

Dos flujos principales + un sub-flujo de autenticación. Campaña: `DBB_MENSUAL_C` (ID: `5783`).

---

## Stack Tecnológico

| Componente | Tecnología | Credencial/ID |
|---|---|---|
| Orquestador | n8n (self-hosted) | — |
| Voz IA | VAPI | Bearer `484afd71-...` |
| LLM Clasificación | OpenAI GPT-5.2 | `Transferencias Vagon 2` |
| LLM Objeciones | OpenAI GPT-4o | `Transferencias Vagon 2` |
| Base de datos | Supabase PostgreSQL | `Gea_postgress` (`Iz7EEp0WgJu3LXhn`) |
| CRM / Reportes | Altiva API | `api-hub.geainternacional.com` |
| Auth Altiva | Sub-workflow `JBab95H9VKPsLOtZ` | Bearer token |
| SIP Trunk | `vapisiptrunk.plusservices.ec` | Blind transfer |
| Zona horaria | `America/Guayaquil` | UTC-5 |
| Código país | `+593` | Ecuador |

---

## Estructura del Proyecto

```
transferencias/
├── flujo_n8n/
│   ├── DBB_GESTIONLLAMADA_MENSUAL_C_v2.json      ← Flujo de marcado (27 nodos)
│   ├── DBB_REPORTEGESTION_TRANSFERENCIA_MENSUAL_C_v2.json  ← Flujo de reporte/webhook (64 nodos)
│   ├── Respaldo/           ← Baseline original (BAU)
│   ├── Respaldo v2/        ← Versión intermedia
│   └── Respaldo v3/        ← Backup reciente
├── Docs/                   ← Documentos de análisis, reglas, audios
├── PruebasLlamadas/        ← Audios de prueba de voces
├── VocesIA/                ← Screenshot de configuración de voz
└── informe.md              ← Auditoría de optimización (6 issues)
```

---

## Arquitectura de Flujos

### Flujo A: DDB_GESTIONLLAMADA_MENSUAL_C_v2 (Marcado Automático)

**Schedule:** Cada 20 segundos. **27 nodos.**

```
Schedule Trigger → CONFIG CAMPANA → BUSCA Y MARCA → [IF: encontrado?]
  → COMO DEBE LLAMAR → POLITICAS Y REINTENTOS → [IF: puedeLlamar?]
    → PROCESA NUMEROS → [IF: isOutOfNumbers?]
      → GUARDA REGISTRO → OCURRE LLAMADA A VAPI
```

**Zonas:**
- **Configuración:** Nodo `CONFIG CAMPANA` (Code) define `id_campania` y `fecha_inicio`
- **Búsqueda:** SQL con CTEs complejos, `SKIP LOCKED`, ordenado por prioridad DESC
- **Políticas:** `pick_policy v2.0` define reglas de marcado, ventanas horarias, límites
- **Selección de teléfono:** Validación real Ecuador, normalización E.164
- **Sin números:** CODE 9 v3.5 maneja agotamiento de teléfonos

### Flujo B: DDB_REPORTEGESTION_TRANSFERENCIA_MENSUAL_C_v2 (Webhook VAPI)

**Entry:** POST `/dbb-mensual-c`. **64 nodos.** Procesa 4 tipos de eventos:

```
Webhook → [IF: type?]
  ├─ transfer-destination-request → ZONA 2: Transferencia SIP
  ├─ end-of-call-report          → ZONA 3: Clasificación post-llamada
  │   ├─ EXTRACTOR VAPI → CALCULA TIMESTAMP → CONSOLIDA CAMPOS
  │   ├─ [IF: reagendamiento?] → ZONA 4: Reagendamiento
  │   ├─ IA: CLASIFICA (GPT-5.2) → UNIFIED ENGINE v4.3
  │   └─ GUARDA BD + ENVIA ALTIVA (ZONA 7)
  ├─ status-update               → ZONA 5: Llamadas fallidas
  │   ├─ EXTRACT STATUS → [IF: buzon?] → IA: CLASIFICA → UNIFIED ENGINE v4.2
  │   └─ GUARDA BD + ENVIA ALTIVA
  └─ tool-calls                  → ZONA 6: Objeciones en vivo
      └─ AI AGENT (GPT-4o + PostgresTool) → Respuesta natural a VAPI
```

---

## Nodos Clave y su Función

### Nodos Code (motores de decisión)

| Nodo | Archivo | Función |
|---|---|---|
| `REINTENTOS + REGLAS MARCADO` | Reporte | **UNIFIED ENGINE v4.3** (end-of-call-report). Clasifica, repara, aplica reglas de prioridad/reintento. |
| `REINTENTOS + REGLAS MARCADO1` | Reporte | **UNIFIED ENGINE v4.2** (status-update). Igual lógica pero para llamadas no atendidas. |
| `COMO DEBE LLAMAR EL BOT (POLITICAS)` | Gestión | `pick_policy v2.0`. Define catálogo `outcomes` con `deltaPrio`, ventanas, límites. |
| `POLITICAS Y REINTENTOS (RESPALDO)` | Gestión | CODE 8 v2.17. Lógica de marcado: valida horario, límites, resetea cursor. |
| `PROCESA LOS NUMEROS Y FORMATEA` | Gestión | SELECT PHONE v11. Valida teléfonos Ecuador real, normaliza E.164. |
| `CUANDO NO SE ENCONTRO NUMERO VALIDO` | Gestión | CODE 9 v3.5. Maneja agotamiento de teléfonos. |
| `CONFIG CAMPANA` | Gestión | Configuración de campaña: `id_campania`, `fecha_inicio`. |

### Nodos PostgreSQL

Todos operan sobre `public.leads_v2`. Conexión: `Gea_postgress`.

| Nodo | Operación |
|---|---|
| `BUSCA Y MARCA COMO EN PROCESO` | SELECT + UPDATE con `SKIP LOCKED` |
| `GUARDA REGISTRO Y CAMPAÑA` | UPDATE (30+ columnas) |
| `GUARDAR REPORTE` | UPDATE (win_stamp, telefono, phones_allowed) |
| `ACTUALIZA LEADS_V2: RESULTADO CLASIFICADO` | UPDATE (clasificación post-llamada) |
| `ACTUALIZA LEADS_V2: STATUS-UPDATE` | UPDATE (varios branches de status) |

### Nodos API

| Nodo | Endpoint |
|---|---|
| `OCURRE LLAMADA A VAPI` | `POST https://api.vapi.ai/call` |
| `ENVIA REPORTE A API ALTIVA (*)` | `POST .../ec/v1/campania/ia-call-summary` |

---

## Catálogo de Outcomes (Fuente de Verdad)

Definido en `COMO DEBE LLAMAR EL BOT (POLITICAS)` → `system_payload.outcomes`.
Viaja en cada registro de `leads_v2`. Es la **única fuente de verdad** para `deltaPrio`.

### Estructura de cada outcome:
```js
{ idNom, idMotivo, key, wait, maxTries, deltaPrio, aliases }
```

### Nomenclaturas:

| idNom | Nombre | Significado |
|---|---|---|
| 1 | ACEPTACION | Cliente aceptó transferencia |
| 2 | NO ACEPTACION | Rechazo explícito (cierra molesto, no confía, etc.) |
| 3 | PENDIENTE DE ACEPTACION | Necesita pensar, solo preguntó |
| 4 | CONTACTO FALLIDO JUSTIFICADO | Descarte permanente (cuenta cerrada, fallecido, etc.) |
| 5 | CONTACTO FALLIDO NO JUSTIFICADO | Reintentar (no se encuentra, llamar después, etc.) |
| 6 | NO CONTACTO JUSTIFICADO | Descarte permanente (línea fuera de servicio, no existe) |
| 7 | NO CONTACTO NO JUSTIFICADO | Buzón, no contesta, ocupado, fax |

### deltaPrio por outcome:
- **0:** Aceptación, rechazos, pendientes, contactos fallidos permanentes
- **-1:** Cuenta corporativa, equivocado, IVR empresa, no se encuentra, llamar después, ocupado, fuera de servicio, no existe, buzón, fax
- **-4:** NO CONTESTA (motivo 42) — la mayor penalización

---

## Sistema de Prioridad

1. **Valor inicial:** `prioridad = 8` (asignado en carga de datos).
2. **Catálogo outcomes** en `system_payload.outcomes` define `deltaPrio` por `(idNom, idMotivo)`.
3. **UNIFIED ENGINE** (v4.3 y v4.2) calcula: `nueva_prioridad = inputPrioridad + deltaPrio`.
4. **TODAS las ramas** usan `inputPrioridad + deltaPrio` sin excepción (corregido 2026-06-08).
5. **Rango:** `priorityMin: -5`, `priorityMax: 0` (definido en políticas).
6. **Orden de selección:** `ORDER BY prioridad DESC` (mayor prioridad primero).

---

## Sistema de Banderas

- **`bandera`**: Número de día de la campaña (1, 2, 3, ...).
- **`campaign_start`**: Fecha de inicio, configurada en nodo `CONFIG CAMPANA` → `fecha_inicio`.
- **`current_bandera`**: `(CURRENT_DATE - fecha_inicio) + 1`. Mínimo 1.
- **Selección:** Solo registros con `bandera = current_bandera` (modo estricto) o `<= current_bandera` (modo expandido).
- **Expansión:** Si >70% de registros en bandera actual tienen 3+ intentos, se expande a banderas anteriores.

---

## Tabla leads_v2 (columnas relevantes)

| Columna | Tipo | Descripción |
|---|---|---|
| `id_registro` | int | PK, ID del lead |
| `id_campania` | varchar | ID de campaña (ej: '5783') |
| `nombre_cliente` | varchar | Nombre del cliente |
| `status_flujo` | varchar | pendiente/reintentar/en_progreso/finalizado_ok/rechazado/agotado |
| `prioridad` | int | Prioridad de marcado (range: -5 a 0, default: 8 en carga) |
| `intentos_llamada` | int | Intentos totales de llamada |
| `bandera` | varchar | Día de la campaña |
| `reintentar_iso` | timestamptz | Fecha/hora para próximo reintento |
| `last_try_ymd` | date | Último día que se intentó llamar |
| `nomenclatura_ultima` | int | Última nomenclatura clasificada (1-7) |
| `motivo_ultimo` | int | Último motivo clasificado |
| `win_stamp` | varchar | ID de ventana horaria actual |
| `win_tries` | int | Intentos en ventana actual |
| `win_id` | varchar | ID de ventana |
| `telefono_actual` | varchar | Teléfono usado en última llamada |
| `telefono_field` | varchar | Campo del teléfono (ph_celular, etc.) |
| `telefono_index` | int | Posición del teléfono en la lista |
| `ph_celular/ph_celular2/ph_residencial/ph_residencial2/ph_comercial/ph_telefono4` | varchar | Teléfonos |
| `system_payload` | jsonb | Políticas, outcomes, phones_meta |
| `sentimiento` | varchar | Sentimiento detectado |
| `resumen_llamada` | text | Resumen de la llamada |
| `transcripcion` | text | Transcripción completa |
| `duracion_segundos` | int | Duración de llamada |
| `id_altiva` | varchar | ID en Altiva |

---

## Convenciones del Proyecto

1. **Nombres de nodos en n8n:** UPPERCASE descriptivo (ej: `BUSCA Y MARCA COMO EN PROCESO`).
2. **Nombres de campos:** snake_case (ej: `id_campania`, `status_flujo`).
3. **Código JavaScript:** Sin punto y coma al final de statements, funciones con llaves en nueva línea.
4. **Edición de flujos:** Siempre modificar la versión `_v2` activa. Los respaldos son solo referencia.
5. **Credenciales:** Nunca exponer en código. Usar nombres de credencial n8n.
6. **Encoding:** Evitar caracteres especiales (ñ, acentos) en nombres de nodos. Usar CAMPANA sin ñ.

---

## Cómo Modificar los Flujos

### Agregar un nodo
1. Crear objeto nodo con `id` (GUID), `name`, `type`, `position [x, y]`, `parameters`.
2. Agregar al array `nodes`.
3. Actualizar `connections`: agregar entrada del nodo anterior → nuevo nodo, y nuevo nodo → nodo siguiente.
4. La estructura de `main` en connections es: `[[{node, type, index}]]` (array 2D).

### Modificar SQL en nodo PostgreSQL
1. El campo `query` soporta expresiones n8n `{{ }}`.
2. Para parametrizar: `'{{ $json.campo }}'`.
3. Las expresiones se evalúan antes de enviar a PostgreSQL.

### Modificar UNIFIED ENGINE
1. Ubicar el nodo Code (`REINTENTOS + REGLAS MARCADO` o `REINTENTOS + REGLAS MARCADO1`).
2. Extraer `jsCode` completo (puede tener ~15k chars).
3. La función clave es `buildFlowState(canonical, original)`.
4. **Regla de prioridad:** siempre usar `inputPrioridad + deltaPrio` donde `deltaPrio = num(matchedOutcome?.deltaPrio, 0)`.

---

## Solución de Problemas Comunes

### Registro con bandera incorrecta siendo seleccionado
- Verificar `fecha_inicio` en nodo `CONFIG CAMPANA`.
- `current_bandera = (CURRENT_DATE - fecha_inicio) + 1`.
- Si `fecha_inicio` está mal (por MIN(created_at) antiguo), la bandera actual será mayor de lo esperado.

### Prioridad no se reduce correctamente
- Verificar que `system_payload.outcomes` existe en el registro.
- Verificar que `deltaPrio` del outcome coincide con el catálogo.
- El engine debe usar `inputPrioridad + deltaPrio` en TODAS las ramas.

### Conexiones rotas al editar JSON
- Las conexiones usan arrays anidados: `main: [[{node, type, index}]]`.
- Un solo output: `[[{...}]]`. Dos outputs: `[[{...}], [{...}]]`.
