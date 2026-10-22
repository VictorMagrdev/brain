# 05 · Cambio de estado de conversación

Parte de [Documentación de n8n](<../Documentacion de n8n.md>) · Canvas: [Cambio-Estado-Conversacion.canvas](Cambio-Estado-Conversacion.canvas) · Fuente: `n8n-json/CLARO_MOVIL cambio_estado_conversacion.json`

## Qué hace
Sincroniza el cierre. Cuando una conversación del inbox 13 pasa a `resolved` en Chatwoot, pone `paso = 1000` en la tabla para que el próximo mensaje del cliente empiece de cero.

| | |
|---|---|
| **Disparador** | Webhook POST `/webhook/cambio-estado-conversacion-claro-movil` (Chatwoot, evento de cambio de estado; también lo llama Principal). |
| **Llama a** | Ninguno |
| **Es llamado por** | Chatwoot<br>Principal V3 (nodo Validacion error1) |
| **Nodos** | 5 |

## Nodos
| # | Nodo | Tipo | Qué hace | Lee | Maneja / produce |
|---|---|---|---|---|---|
| 1 | **Webhook** | Webhook | Recibe de Chatwoot el cambio de estado de la conversación. | — | POST /webhook/cambio-estado-conversacion-claro-movil |
| 2 | **Switch** | Switch | Filtra por `status`: solo continúa si es `resolved`. | `Webhook.body.status` | salida resolved ⇢ resolved |
| 3 | **inbox** | IF | Filtra que el inbox sea el 13. | `Webhook.body.inbox_id`<br>`Webhook.body.messages`<br>`Webhook.body.contact_inbox.inbox_id` | Number($('Webhook').item.json.body.inbox_id ?? $('Webhook').item.json.body.messages?.[0]?. |
| 4 | **Cerrar conversación** | MySQL | Pone `paso = 1000` al último registro del cliente (por `source_id`). | `(entrada).body.contact_inbox.source_id` | Operación: UPDATE sobre `claro_movil`<br>Escribe: paso, fecha_modificacion |
| 5 | **Ignorar** | NoOp | Descarta el evento (inbox distinto de 13). | — | — |

## Conexiones (de → a)
- Webhook → Switch
- Switch (salida 0) → inbox
- inbox (sí) → Cerrar conversación
- inbox (no) → Ignorar

## Intervención para WAHA
| Nodo | Qué cambiar |
|---|---|
| — | Ningún nodo requiere cambio. |

Volver a [Documentación de n8n](<../Documentacion de n8n.md>)
