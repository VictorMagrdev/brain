# 02 · Webhook Chatwoot (relay)

Parte de [Documentación de n8n](<../Documentacion de n8n.md>) · Canvas: [Relay-Chatwoot.canvas](Relay-Chatwoot.canvas) · Fuente: `n8n-json/CLARO_MOVIL Webhook_chatwoot (relay, sin imagenes).json`

## Qué hace
Puente asesor → cliente. Cuando un asesor escribe en Chatwoot, reenvía ese texto al WhatsApp del cliente, pero solo si la conversación está en `paso = -1` (atendida por humano).

| | |
|---|---|
| **Disparador** | Webhook POST `/webhook/chatwoot-claro-movil` (lo llama Chatwoot en `message_created`). |
| **Llama a** | Ninguno |
| **Es llamado por** | Chatwoot (webhook del inbox 13) |
| **Nodos** | 7 |

## Nodos
| # | Nodo | Tipo | Qué hace | Lee | Maneja / produce |
|---|---|---|---|---|---|
| 1 | **Webhook** | Webhook | Recibe el evento de Chatwoot `message_created`. | — | POST /webhook/chatwoot-claro-movil |
| 2 | **If2** | IF | Filtro: solo mensajes `message_created`, salientes, públicos, de un agente (`sender.type = user`) y que no sean del bot. | `(entrada).body.message_type`<br>`(entrada).body.private`<br>`(entrada).body.sender.type`<br>`(entrada).body.source_id`<br>`(entrada).body.event` | $json.body.message_type === 'outgoing' && $json.body.private !== true && $json.body.sender |
| 3 | **inbox** | IF | Verifica que el mensaje pertenezca al inbox 13. | `(entrada).body.conversation.messages[0].inbox_id` | $json.body.conversation.messages[0].inbox_id  ==  13 |
| 4 | **Obtener estado** | MySQL | Consulta en MySQL el `paso` del cliente (por `source_id` o identificador del contacto). | `Webhook.body.conversation.contact_inbox.source_id`<br>`Webhook.body.conversation.meta.sender.identifier` | Operación: SELECT sobre `claro_movil`<br>Lee: * |
| 5 | **Variables** | Set (variables) | Variables del flujo: estado actual (`paso`), claves, URLs, sesión e inbox. | `(entrada).paso` | tipo = CLARO_MOVIL<br>estado_actual = {{ $json.paso }}<br>api_chatwoot = <clave oculta><br>url_chatwoot = https://botconnect.interactivo.com.co:8443<br>api_evolution = <clave oculta><br>url_evolution = https://evolution-general.interactivo.com.co |
| 6 | **If1** | IF | Deja pasar solo si el cliente está en `paso = -1` (atendido por humano). | `Obtener estado.paso` | String($('Obtener estado').item.json.paso).trim()  ==  -1 |
| 7 | **Enviar Texto WhatsApp** | HTTP Request | Envía por WhatsApp el texto que escribió el asesor en Chatwoot (limpiando saltos y comillas). | `Variables.url_evolution`<br>`Variables.session`<br>`Variables.api_evolution`<br>`Webhook.body.conversation.contact_inbox.source_id` | POST {url_evolution}/message/sendText/{session}<br>Body: number, text, message_type<br>Headers: apikey |

## Conexiones (de → a)
- Webhook → If2
- inbox (sí) → Obtener estado
- Obtener estado → Variables
- Variables → If1
- If1 (sí) → Enviar Texto WhatsApp
- If2 (sí) → inbox

## Intervención para WAHA
| Nodo | Qué cambiar |
|---|---|
| Variables | Variables: `url_evolution`/`api_evolution` → URL y clave de WAHA; sesión `AGENTE_PRUEBA_1` (ya coincide). |
| Enviar Texto WhatsApp | Envío: URL `/message/sendText/{session}` → `/api/sendText`; header `apikey` → `X-Api-Key`; body `number` → `chatId` (`…@c.us`) + `session`. |

Volver a [Documentación de n8n](<../Documentacion de n8n.md>)
