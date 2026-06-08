# SPECS.md — Especificaciones Técnicas del Proyecto Transferencias

## 1. Visión General del Sistema

### 1.1 Propósito
Sistema automatizado de call center para campañas de transferencias bancarias. El bot de voz IA (VAPI) realiza llamadas salientes a clientes del Banco Bolivariano, clasifica los resultados mediante GPT-5.2, aplica reglas de reintento/descarte, y reporta a Altiva CRM. Maneja objeciones en vivo con GPT-4o.

### 1.2 Alcance
- Campaña activa: `DBB_MENSUAL_C` (ID: `5783`)
- País: Ecuador (código `+593`, zona horaria `America/Guayaquil`)
- Cliente: Banco Bolivariano

---

## 2. Arquitectura del Sistema

### 2.1 Diagrama de Componentes

```
┌─────────────────────────────────────────────────────────┐
│                     n8n (Orquestador)                      │
│  ┌─────────────────────┐  ┌─────────────────────────────┐ │
│  │ Flujo A: Gestión    │  │ Flujo B: Reporte / Webhook  │ │
│  │ (Schedule 20s)      │  │ (POST /dbb-mensual-c)       │ │
│  │ ┌─────────────────┐ │  │ ┌─────────────────────────┐ │ │
│  │ │ CONFIG CAMPANA   │ │  │ │ UNIFIED ENGINE v4.3     │ │ │
│  │ │ BUSCA Y MARCA    │ │  │ │ UNIFIED ENGINE v4.2     │ │ │
│  │ │ POLITICAS v2.0   │ │  │ │ AI AGENT (GPT-4o)       │ │ │
│  │ │ SELECT PHONE v11 │ │  │ │ CLASIFICADOR (GPT-5.2)  │ │ │
│  │ └─────────────────┘ │  │ └─────────────────────────┘ │ │
│  └─────────────────────┘  └─────────────────────────────┘ │
└─────────────────────────────────────────────────────────┘
          │              │                │
          ▼              ▼                ▼
    ┌─────────┐   ┌───────────┐   ┌──────────────┐
    │  VAPI   │   │ Supabase  │   │ Altiva API   │
    │ (Llamadas)│  │ (leads_v2)│   │ (Reportes)   │
    └─────────┘   └───────────┘   └──────────────┘
```

### 2.2 Flujo de Datos

```
1. Carga de datos → leads_v2 (prioridad=8, bandera asignada, system_payload)
2. Schedule 20s → BUSCA Y MARCA → selecciona 1 lead (SKIP LOCKED)
3. POLITICAS decide si llamar → PROCESA NUMEROS → VAPI call
4. VAPI webhook → clasifica (GPT-5.2) → UNIFIED ENGINE → guarda + reporta
5. Objeciones en vivo → AI AGENT (GPT-4o) → respuesta a VAPI
```

---

## 3. Flujo A: Gestión de Llamadas (Marcado)

### 3.1 Schedule
- **Trigger:** `ScheduleTrigger` cada 20 segundos
- **Nodo:** `EJECUTA ESTO CADA 20 SEG`

### 3.2 Configuración de Campaña
- **Nodo:** `CONFIG CAMPANA` (Code)
- **Output:** `{ id_campania: '5783', fecha_inicio: '2026-06-08' }`
- **Uso:** El SQL de búsqueda usa `{{ $json.fecha_inicio }}` y `{{ $json.id_campania }}`

### 3.3 Búsqueda y Selección (SQL)
- **Nodo:** `BUSCA Y MARCA COMO EN PROCESO` (PostgreSQL)
- **Operación:** Transacción completa con CTEs anidados
- **Lógica:**
  1. `campaign_start`: fecha de inicio desde `{{ $json.fecha_inicio }}`
  2. `current_bandera`: `GREATEST(1, (CURRENT_DATE - fecha_inicio) + 1)`
  3. `permanent_discard`: motivos que no se reintentan
  4. `bandera_stats`: estadísticas de intentos en bandera actual
  5. `call_mode`: modo estricto (solo bandera actual) vs expandido (>70% con 3+ intentos)
  6. `phones_meta_norm`: normalización de metadatos de teléfonos
  7. `next`: SELECT con LIMIT 1, FOR UPDATE SKIP LOCKED
  8. `upd`: UPDATE status_flujo = 'en_progreso', picked_at = NOW()
- **Orden de selección:** `ORDER BY prioridad DESC, bandera_match, phone_tries ASC, reintentar_iso NULLS FIRST, picked_at NULLS FIRST, created_at`
- **Estados elegibles:** `pendiente`, `reintentar`
- **Límites:** `intentos_llamada < 14`, `reintentar_iso <= NOW()`, descarte diario < 7

### 3.4 Políticas de Marcado
- **Nodo:** `COMO DEBE LLAMAR EL BOT (POLITICAS)` (Code)
- **Versión:** `pick_policy v2.0`
- **Configuración:**
  - `TZ`: `America/Guayaquil`
  - `CC`: `+593`
  - `dayMax`: 7 (intentos por día)
  - `maxLifetimeTries`: 14 (intentos de vida)
  - `priorityMin`: -5, `priorityMax`: 0
  - `resetContactTriesOnWindowChange`: true
- **Ventanas horarias:**
  - `WD_0800_1759`: L-V, 08:00-17:59, max 1 intento, wait 30min
  - `WE_0930_1230`: Sábado, 09:30-12:30, max 1 intento, wait 30min
- **Orden de teléfonos por ventana:** `ph_celular, ph_celular2, ph_residencial, ph_residencial2, ph_comercial, ph_telefono4`
- **Catálogo outcomes:** Ver sección 8

### 3.5 Validación de Llamada
- **Nodo:** `POLITICAS Y REINTENTOS (RESPALDO)` (Code)
- **Versión:** CODE 8 v2.17
- **Validaciones:**
  1. Inyecta política DEFAULT si `system_payload` está vacío
  2. Valida horario vs ventanas (`windows`)
  3. Valida límite por ventana (max = 1)
  4. Restaura pairing de DB (`win_*`, `telefono_index`)
  5. Reset del cursor SOLO al cambiar de ventana
  6. Respeta `maxLifetimeTries` (14)

### 3.6 Selección de Teléfono
- **Nodo:** `PROCESA LOS NUMEROS Y FORMATEA` (Code)
- **Versión:** SELECT PHONE v11
- **Validación Ecuador real:**
  - Celular: `/^09\d{8}$/` o `/^5939\d{8}$/`
  - Fijo: códigos de área `02, 03, 04, 05, 06, 07`
- **Normalización E.164:** `+593` + dígitos

### 3.7 Sin Números Disponibles
- **Nodo:** `CUANDO NO SE ENCONTRO NUMERO VALIDO` (Code)
- **Versión:** CODE 9 v3.5
- **Casos:**
  - Números existen pero ventana no los permite → `PENDIENTE` + próxima ventana
  - Sin dígitos en ningún campo → `AGOTADO` + `DISCARDED`
  - Dígitos pero todos inválidos → `AGOTADO` + `DISCARDED`

### 3.8 Llamada VAPI
- **Nodo:** `OCURRE LLAMADA A VAPI` (HTTP Request)
- **Endpoint:** `POST https://api.vapi.ai/call`
- **Auth:** `Bearer 484afd71-2703-4852-8580-9629620dde82`
- **Assistant ID:** `3fcd093b-b8c2-4d85-aa50-e1da35ea6111`
- **Phone Number ID:** `451334d1-9ee7-4659-aa0b-491450adcbe5`
- **Variable values enviadas:** id_campania, id_registro, cedula, nuic, nombre_cliente, segmento, ciudad, email, teléfonos, win_*, prioridad, consumo, intentos_llamada, id_altiva, sip=9331

---

## 4. Flujo B: Reporte y Webhook (Cierre de Llamada)

### 4.1 Entrada
- **Nodo:** `Webhook`
- **Ruta:** `POST /dbb-mensual-c`
- **Eventos procesados:**
  - `transfer-destination-request` → ZONA 2
  - `end-of-call-report` → ZONA 3
  - `status-update` → ZONA 5
  - `tool-calls` → ZONA 6

### 4.2 ZONA 2: Transferencia SIP
- **Condición:** `$json.body.message.type === 'transfer-destination-request'`
- **Modo:** Blind transfer
- **SIP Trunk:** `vapisiptrunk.plusservices.ec`
- **Extensión:** 9331
- **Respuesta:** JSON con `destination` SIP URI + headers (caller ID, etc.)

### 4.3 ZONA 3: Clasificación Post-Llamada (end-of-call-report)
1. `EXTRAE DATOS ESTRUCTURADOS DEL REPORTE VAPI` → extrae structuredOutputs, variables, summary, sentiment, transcript, etc.
2. `CALCULA TIMESTAMP REAGENDAMIENTO` → normaliza fechas de reagendamiento a ISO 8601
3. `CONSOLIDA TODOS LOS CAMPOS` → 32+ asignaciones de campos
4. `SI TIENE REAGENDAMIENTO VALIDO` → IF → ZONA 4
5. `IA: CLASIFICA MOTIVO DE LLAMADA` → GPT-5.2 clasifica (id_nomenclatura, id_motivo)
6. `REINTENTOS + REGLAS MARCADO` → **UNIFIED ENGINE v4.3** (ver sección 6.1)
7. `ACTUALIZA LEADS_V2: RESULTADO CLASIFICADO` → PostgreSQL UPDATE
8. `ENVIA REPORTE A API ALTIVA` → POST a Altiva

### 4.4 ZONA 4: Reagendamiento
- **Condición:** Cliente pide ser llamado en otro momento
- **Procesamiento:** Fecha normalizada por IA → guardada en `reintentar_iso`
- **Status:** `pendiente` (con fecha futura)
- **Guardado:** `ACTUALIZA LEADS_V2: REAGENDAMIENTO`
- **Reporte:** `ENVIA REPORTE A API ALTIVA (reagendamiento programado)`

### 4.5 ZONA 5: Status Update (Llamadas Fallidas)
- **Condición:** `$json.body.message.type === 'status-update'`
- **Sub-tipos:** `no-answer`, `voicemail`, `busy`, `customer-ended-call`
- **Flujo:**
  1. `EXTRAE DATOS DE STATUS UPDATE VAPI`
  2. `¿ES BUZON DE VOZ?` → IF → `FUERZA CLASIFICACION BUZON DE VOZ` o `IA: CLASIFICA`
  3. `REINTENTOS + REGLAS MARCADO1` → **UNIFIED ENGINE v4.2** (ver sección 6.2)
  4. `MAPEA CAMPOS STATUS-UPDATE` → `ACTUALIZA LEADS_V2: STATUS-UPDATE`
  5. `ENVIA REPORTE A API ALTIVA`

### 4.6 ZONA 6: AI Agent - Objeciones en Vivo
- **Condición:** `$json.body.message.type === 'tool-calls'`
- **Modelo:** GPT-4o (`Transferencias Vagon 2`)
- **Herramienta:** PostgresTool → `callcenter_objeciones_asistencia`
- **Query SQL:** `SELECT * FROM callcenter_objeciones_asistencia WHERE UPPER(TRIM(categoria)) ILIKE '%' || UPPER(TRIM($1)) || '%' ORDER BY RANDOM()`
- **Memoria:** Buffer window de 20 mensajes
- **Flujo:** Extrae prompt + contexto → AI Agent clasifica objeción → busca respuesta en BD → devuelve a VAPI

### 4.7 ZONA 7: Persistencia y Reportes
- **Actualización BD:** PostgreSQL UPDATE sobre `leads_v2` (18+ columnas)
- **Reporte Altiva:** `POST https://api-hub.geainternacional.com/ec/v1/campania/ia-call-summary`
- **Auth:** Bearer token del sub-workflow `JBab95H9VKPsLOtZ`
- **Campos enviados:** id_cuenta, id_registro, resumen_ejecutivo, sentimiento, transcripcion, duracion_segundos, telefono_contacto, id_nomenclatura, id_motivo, url_grabacion, observacion, fecha_hora

---

## 5. Esquema de Base de Datos

### 5.1 Tabla: `public.leads_v2`

| Columna | Tipo | Restricciones | Descripción |
|---|---|---|---|
| `id_registro` | int | PK | ID único del lead |
| `id_campania` | varchar | NOT NULL | ID de campaña (ej: '5783') |
| `cedula` | varchar | | Cédula de identidad |
| `nuic` | varchar | | Número único de identificación |
| `nombre_cliente` | varchar | | Nombre completo |
| `segmento` | varchar | | Segmento de cliente (AHO, etc.) |
| `ciudad` | varchar | | Ciudad |
| `email` | varchar | | Email |
| `phones_raw` | jsonb | | Teléfonos en formato raw |
| `ph_celular` | varchar | | Teléfono celular 1 |
| `ph_celular2` | varchar | | Teléfono celular 2 |
| `ph_residencial` | varchar | | Teléfono residencial 1 |
| `ph_residencial2` | varchar | | Teléfono residencial 2 |
| `ph_comercial` | varchar | | Teléfono comercial |
| `ph_telefono4` | varchar | | Teléfono adicional |
| `status_flujo` | varchar | | Estado: pendiente, reintentar, en_progreso, finalizado_ok, rechazado, agotado |
| `reintentar_iso` | timestamptz | | Fecha/hora próximo reintento |
| `prioridad` | int | DEFAULT 8 | Prioridad de marcado (range: -5 a 0) |
| `intentos_llamada` | int | DEFAULT 0 | Total intentos de llamada |
| `last_try_ymd` | date | | Último día de intento |
| `picked_at` | timestamptz | | Timestamp de selección |
| `win_stamp` | varchar | | ID de ventana horaria actual |
| `win_tries` | int | DEFAULT 0 | Intentos en ventana actual |
| `win_id` | varchar | | ID de ventana |
| `nomenclatura_ultima` | int | | Última nomenclatura (1-7) |
| `motivo_ultimo` | int | | Último motivo clasificado |
| `resumen_llamada` | text | | Resumen de la última llamada |
| `sentimiento` | varchar | | Sentimiento detectado |
| `transcripcion` | text | | Transcripción completa |
| `duracion_segundos` | int | | Duración de llamada en segundos |
| `observaciones_internas` | text | | Notas internas |
| `system_payload` | jsonb | | Políticas, outcomes, phones_meta |
| `telefono_actual` | varchar | | Teléfono usado en última llamada |
| `telefono_field` | varchar | | Campo del teléfono usado |
| `telefono_index` | int | | Posición del teléfono en la lista |
| `phones_allowed` | varchar | | Teléfonos permitidos en ventana actual |
| `created_at` | timestamptz | | Fecha de creación |
| `updated_at` | timestamptz | | Fecha de última actualización |
| `last_outcome_at` | timestamptz | | Fecha del último outcome |
| `bandera` | varchar | | Día de la campaña (1, 2, 3, ...) |
| `consumo` | varchar | | Información de consumo |
| `horareagenda` | varchar | | Hora de reagendamiento |
| `uso tarjeta` | boolean | | ¿Usa tarjeta? |
| `id_altiva` | varchar | | ID en Altiva CRM |
| `vapi_call_id` | varchar | | ID de llamada en VAPI |
| `vapi_intentos` | int | DEFAULT 0 | Intentos via VAPI |
| `es_agendamiento` | boolean | | ¿Es reagendamiento? |
| `id_agendamiento` | int | | ID de reagendamiento |
| `agendado_para` | timestamptz | | Fecha de reagendamiento |

### 5.2 Tabla: `public.callcenter_objeciones_asistencia`

| Columna | Tipo | Descripción |
|---|---|---|
| `categoria` | varchar | Categoría de objeción |
| (otros) | — | Columnas de respuesta para el AI Agent |

---

## 6. UNIFIED ENGINE (Motor de Decisión)

### 6.1 UNIFIED ENGINE v4.3 (end-of-call-report)
- **Nodo:** `REINTENTOS + REGLAS MARCADO`
- **Ubicación:** `DBB_REPORTEGESTION_TRANSFERENCIA_MENSUAL_C_v2.json`
- **Función:** `buildFlowState(canonical, original)`
- **Features:**
  - Anti-hallucination (KB detection + repair)
  - Safe JSON extraction (tolerante a markdown)
  - Auto repair de `id_motivo`/`id_nomenclatura`
  - Strict classification validation
  - Flow state machine
  - Retry engine
  - Attempt counter fix (shouldCountAttempt)
- **Cálculo de prioridad:**
  ```js
  const inputPrioridad = num(original?.prioridad, 8)
  const outcomes = original?.system_payload?.outcomes || []
  const matchedOutcome = outcomes.find(o => o.idNom === nom && o.idMotivo === motivo)
  const deltaPrio = num(matchedOutcome?.deltaPrio, 0)
  const nuevaPrioridad = inputPrioridad + deltaPrio  // aplicado en TODAS las ramas
  ```
- **Estados de salida:**
  - `finalizado_ok`: nom 1 (aceptación)
  - `rechazado`: nom 2 (no aceptación), nom 3 (equivocado)
  - `reintentar`: nom 4, nom 5, nom 7→41 (voicemail), nom 7→43 (busy)
  - `agotado`: nom 7→44 (fax), nom 7 default, max retries alcanzados
  - `agotado_hoy`: max retries (early return)
  - `pendiente`: fallback

### 6.2 UNIFIED ENGINE v4.2 (status-update)
- **Nodo:** `REINTENTOS + REGLAS MARCADO1`
- **Ubicación:** `DBB_REPORTEGESTION_TRANSFERENCIA_MENSUAL_C_v2.json`
- **Función:** `buildFlowState(canonical, original)`
- **Diferencias vs v4.3:**
  - Sin contador de intentos (attempt counter fix)
  - Procesa `endedReason` en vez de resumen de llamada
  - Aplica políticas de reintento para no-contactos
  - Max retries se chequea DESPUÉS de las ramas de nomenclatura
  - nom 7 es un solo caso plano (sin sub-ramas por motivo)

---

## 7. Sistema de Clasificación

### 7.1 Nomenclaturas

| idNom | Nombre | Significado | Acción |
|---|---|---|---|
| 1 | ACEPTACION | Cliente aceptó ser transferido | finalizado_ok |
| 2 | NO ACEPTACION | Rechazo explícito | rechazado |
| 3 | PENDIENTE DE ACEPTACION | Duda, necesita pensar, solo preguntó | rechazado |
| 4 | CONTACTO FALLIDO JUSTIFICADO | Descarte permanente | reintentar / deltaPrio |
| 5 | CONTACTO FALLIDO NO JUSTIFICADO | Reintentar pronto | reintentar / deltaPrio |
| 6 | NO CONTACTO JUSTIFICADO | Descarte permanente | reintentar / deltaPrio |
| 7 | NO CONTACTO NO JUSTIFICADO | Buzón, no contesta, ocupado, fax | reintentar o agotado |

### 7.2 Motivos por Nomenclatura

**Nom 1 (ACEPTACION):**
- `null`: Aceptación de transferencia

**Nom 2 (NO ACEPTACION):**
- `2`: Cliente cierra molesto
- `5`: No confía en este tipo de llamadas
- `14`: Titular cerrará la cuenta
- `1129`: Cliente grosero durante la llamada
- `1695`: Cuenta cancelada
- `2029`: Cliente no da apertura al inicio
- `2520`: No desea ser transferido
- `2620`: No da apertura

**Nom 3 (PENDIENTE DE ACEPTACION):**
- `20`: Necesita pensar
- `2683`: Pendiente, solo preguntó sobre transferencia

**Nom 4 (CONTACTO FALLIDO JUSTIFICADO):**
- `22`: Cambio de lugar de trabajo
- `23`: Cuenta bancaria cerrada
- `24`: Cuenta corporativa o compartida
- `25`: Equivocado
- `26`: No reside en el país
- `27`: No vive ahí
- `28`: Titular con discapacidad
- `29`: Titular fallecido
- `30`: Titular no habla español
- `1005`: Imposible contactarse con el cliente
- `1043`: Cliente solicita no ser llamado
- `1130`: IVR de empresa

**Nom 5 (CONTACTO FALLIDO NO JUSTIFICADO):**
- `31`: Titular está fuera del país
- `32`: Titular no se encuentra
- `33`: Llamar a otro teléfono
- `34`: No se transfirió la llamada al titular
- `35`: Se cortó la llamada
- `38`: Volver a llamar
- `3359`: Ocupado

**Nom 6 (NO CONTACTO JUSTIFICADO):**
- `39`: Línea telefónica fuera de servicio o suspendida
- `40`: Línea telefónica no existe

**Nom 7 (NO CONTACTO NO JUSTIFICADO):**
- `41`: Buzón de voz
- `42`: No contesta
- `43`: Suena ocupado
- `44`: Tono de fax

---

## 8. Catálogo de Outcomes (deltaPrio)

Fuente de verdad definida en `COMO DEBE LLAMAR EL BOT (POLITICAS)` → `system_payload.outcomes`.

| idNom | idMotivo | deltaPrio | Descripción |
|---|---|---|---|
| 1 | 0 | 0 | ACEPTACION |
| 2 | 2 | 0 | CLIENTE CIERRA MOLESTO |
| 2 | 5 | 0 | NO CONFIA |
| 2 | 14 | 0 | TITULAR CERRARA LA CUENTA |
| 2 | 1129 | 0 | CLIENTE GROSERO |
| 2 | 1695 | 0 | CUENTA CANCELADA |
| 2 | 2029 | 0 | NO DA APERTURA INICIO |
| 2 | 2520 | 0 | NO DESEA SER TRANSFERIDO |
| 2 | 2620 | 0 | NO DA APERTURA |
| 3 | 20 | 0 | NECESITA PENSAR |
| 3 | 2683 | 0 | PENDIENTE SOLO PREGUNTA |
| 4 | 22 | 0 | CAMBIO DE TRABAJO |
| 4 | 23 | 0 | CUENTA CERRADA |
| 4 | 24 | -1 | CUENTA CORPORATIVA |
| 4 | 25 | -1 | EQUIVOCADO |
| 4 | 26 | 0 | NO RESIDE EN EL PAIS |
| 4 | 27 | 0 | NO VIVE AHI |
| 4 | 28 | 0 | TITULAR CON DISCAPACIDAD |
| 4 | 29 | 0 | TITULAR FALLECIDO |
| 4 | 30 | 0 | NO HABLA ESPANOL |
| 4 | 1005 | 0 | IMPOSIBLE CONTACTAR |
| 4 | 1043 | 0 | SOLICITA NO SER LLAMADO |
| 4 | 1130 | -1 | IVR DE EMPRESA |
| 5 | 31 | -1 | FUERA DEL PAIS |
| 5 | 32 | -1 | NO SE ENCUENTRA |
| 5 | 33 | -1 | OTRO TELEFONO |
| 5 | 34 | -1 | NO SE TRANSFIRIO |
| 5 | 35 | -1 | SE CORTO LA LLAMADA |
| 5 | 38 | -1 | VOLVER A LLAMAR |
| 5 | 3359 | -1 | OCUPADO |
| 6 | 39 | -1 | LINEA FUERA DE SERVICIO |
| 6 | 40 | -1 | LINEA NO EXISTE |
| 7 | 41 | -1 | BUZON DE VOZ |
| 7 | 42 | **-4** | NO CONTESTA |
| 7 | 43 | -1 | SUENA OCUPADO |
| 7 | 44 | -1 | TONO DE FAX |

**Reglas:**
- `deltaPrio = 0`: Sin cambio de prioridad (aceptaciones, rechazos, pendientes, contactos fallidos permanentes)
- `deltaPrio = -1`: Penalización leve (cuenta corporativa, equivocado, IVR, no se encuentra, llamar después, ocupado, fuera de servicio, no existe, buzón, fax)
- `deltaPrio = -4`: Penalización máxima (NO CONTESTA — motivo 42)

---

## 9. Sistema de Banderas (Días de Campaña)

### 9.1 Concepto
- `bandera`: Número de día de la campaña (1, 2, 3, ...)
- Asignado durante la carga de datos
- Controla qué lotes de registros se procesan cada día

### 9.2 Cálculo
```sql
current_bandera = GREATEST(1, (CURRENT_DATE - fecha_inicio) + 1)
```
Donde `fecha_inicio` viene de `CONFIG CAMPANA` (antes se usaba `MIN(created_at)`).

### 9.3 Modos de Selección

**Modo Estricto** (`strict_mode = true`):
- Solo registros con `bandera = current_bandera`
- Se activa cuando hay registros en la bandera actual y <70% tienen 3+ intentos

**Modo Expandido** (`strict_mode = false`):
- Registros con `bandera <= current_bandera`
- Se activa cuando:
  - No hay registros en la bandera actual (total = 0)
  - O >=70% de registros en bandera actual tienen 3+ intentos

### 9.4 Ejemplo
Si `fecha_inicio = '2026-06-08'`:
- 8/6/2026: `bandera = 1`
- 9/6/2026: `bandera = 2`
- 10/6/2026: `bandera = 3`

---

## 10. Integraciones API

### 10.1 VAPI
- **Base URL:** `https://api.vapi.ai`
- **Auth:** Bearer token `484afd71-2703-4852-8580-9629620dde82`
- **Endpoints:**
  - `POST /call` — Iniciar llamada saliente
  - `POST /dbb-mensual-c` (webhook) — Recibir eventos

### 10.2 Altiva CRM
- **Base URL:** `https://api-hub.geainternacional.com`
- **Endpoint:** `POST /ec/v1/campania/ia-call-summary`
- **Auth:** Bearer token via sub-workflow `JBab95H9VKPsLOtZ`
- **Payload:**
  ```json
  {
    "id_cuenta": "5783",
    "id_registro": "99",
    "resumen_ejecutivo": "...",
    "sentimiento": "positive|neutral|negative",
    "transcripcion": "...",
    "duracion_segundos": 120,
    "telefono_contacto": "0995390486",
    "id_nomenclatura": 1,
    "id_motivo": null,
    "url_grabacion": "...",
    "observacion": "...",
    "fecha_hora": "2026-06-08 14:42:00"
  }
  ```

### 10.3 OpenAI
- **Credencial:** `Transferencias Vagon 2`
- **Modelos:**
  - GPT-5.2: Clasificación de motivos post-llamada
  - GPT-4o: AI Agent para manejo de objeciones en vivo (temperature: 0.3)

---

## 11. Configuración de Entorno

### 11.1 Credenciales n8n

| Nombre | Tipo | ID |
|---|---|---|
| `Gea_postgress` | PostgreSQL | `Iz7EEp0WgJu3LXhn` |
| `Transferencias Vagon 2` | OpenAI API | `QzoOK3GYXjiOR7BY` |

### 11.2 Sub-Workflow
- **ID:** `JBab95H9VKPsLOtZ`
- **Función:** Generar Bearer token para Altiva API
- **Llamado desde:** 5 nodos `ExecuteWorkflow` en el flujo de reporte

---

## 12. Registro de Cambios (Changelog)

| Fecha | Cambio | Archivo |
|---|---|---|
| 2026-06-08 | Corrección: `inputPrioridad + deltaPrio` en TODAS las ramas de v4.3 y v4.2 | Reporte |
| 2026-06-08 | Eliminado hardcode `prioridad: -2` y `prioridad: 0` en v4.3 | Reporte |
| 2026-06-08 | Agregado nodo `CONFIG CAMPANA` con `fecha_inicio` e `id_campania` | Gestión |
| 2026-06-08 | SQL `campaign_start` cambiado de `MIN(created_at)` a `{{ $json.fecha_inicio }}` | Gestión |
| 2026-06-08 | Parametrizado `id_campania` en SQL con `{{ $json.id_campania }}` | Gestión |
