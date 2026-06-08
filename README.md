# Transferencias — Banco Bolivariano

Sistema automatizado de call center para campañas de transferencias bancarias usando **n8n + VAPI + Supabase + OpenAI + Altiva**.

## Stack

| Componente | Tecnología |
|---|---|
| Orquestador | n8n (self-hosted) |
| Voz IA | VAPI |
| Clasificación | OpenAI GPT-5.2 |
| Objeciones en vivo | OpenAI GPT-4o + PostgresTool |
| Base de datos | Supabase PostgreSQL (`leads_v2`) |
| CRM / Reportes | Altiva API |
| Región | Ecuador (`+593`, `America/Guayaquil`) |

## Flujos

| Flujo | Archivo | Trigger | Nodos |
|---|---|---|---|
| **Gestión (Marcado)** | `DBB_GESTIONLLAMADA_MENSUAL_C_v2.json` | Schedule cada 20s | 27 |
| **Reporte (Webhook)** | `DBB_REPORTEGESTION_TRANSFERENCIA_MENSUAL_C_v2.json` | POST `/dbb-mensual-c` | 64 |
| **Auth Altiva** | `JBab95H9VKPsLOtZ` (sub-workflow) | ExecuteWorkflow | — |

## Arquitectura

```
Schedule 20s → CONFIG CAMPANA → BUSCA Y MARCA → [políticas] → [selección teléfono] → VAPI call
                                                                                          ↓
Webhook VAPI ←────────────────────────────────────────────────────────────────────────────┘
     ↓
[transfer-destination-request] → SIP blind transfer
[end-of-call-report]           → GPT-5.2 clasifica → UNIFIED ENGINE v4.3 → BD + Altiva
[status-update]                → GPT-5.2 clasifica → UNIFIED ENGINE v4.2 → BD + Altiva
[tool-calls]                   → GPT-4o AI Agent → PostgresTool → respuesta a VAPI
```

## Conceptos Clave

- **Prioridad:** 8 inicial, se reduce por `deltaPrio` del catálogo `outcomes` (única fuente de verdad)
- **Bandera:** Número de día de campaña. Solo se procesan registros de la bandera actual
- **UNIFIED ENGINE:** Motor de decisión post-llamada (v4.3 para end-of-call, v4.2 para status-update)
- **Outcomes:** Catálogo con 32 entradas `(idNom, idMotivo)` → `deltaPrio`, `wait`, `maxTries`

## Documentación

- `AGENTS.md` — Guía completa para agentes de IA
- `SPECS.md` — Especificación técnica detallada
- `informe.md` — Auditoría de optimización
