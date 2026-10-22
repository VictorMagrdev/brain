# 03 · Opción inválida

Parte de [Documentación de n8n](<../Documentacion de n8n.md>) · Canvas: [Opcion-Invalida.canvas](Opcion-Invalida.canvas) · Fuente: `n8n-json/CLARO_MOVIL opcion-invalida.json`

## Qué hace
Maneja las respuestas no válidas del cliente. Suma un error en `num_errores`; con 1 o 2 errores avisa y reenvía el menú del paso actual; al 3.º error cierra la conversación (Chatwoot resolved, `paso = 1000`).

| | |
|---|---|
| **Disparador** | Webhook POST `/webhook/opcion-invalida-claro-movil`. |
| **Llama a** | `inactividad-claro-movil` (nodo Activar inactividad) |
| **Es llamado por** | Principal V3 (nodo Llamar opción inválida) |
| **Nodos** | 23 |

## Nodos
| # | Nodo | Tipo | Qué hace | Lee | Maneja / produce |
|---|---|---|---|---|---|
| 1 | **Webhook** | Webhook | Recibe del Principal: cliente, conversación, cuenta, tipo, paso, URLs, claves y sesión. | — | POST /webhook/opcion-invalida-claro-movil |
| 2 | **Obtener estado** | MySQL | Lee `num_errores` actual del cliente. | `Webhook.body.cliente` | Operación: SELECT sobre `claro_movil`<br>Lee: COALESCE(num_errores,0) AS num_errores |
| 3 | **Code** | Code (JS) | Suma 1 al contador `num_errores`. | — | Devuelve: num_errores |
| 4 | **Validar cierre chat** | IF | Si `num_errores ≥ 3` cierra la conversación; si no, avisa y reenvía el menú. | `(entrada).num_errores` | $json.num_errores >= 3 |
| 5 | **Cierre Chatwoot** | HTTP Request | Publica en Chatwoot el aviso de cierre automático por respuestas no identificadas. | `Webhook.body.url_chatwoot`<br>`Webhook.body.id_cuenta`<br>`Webhook.body.id_conversacion`<br>`Webhook.body.api_chatwoot` | POST {url_chatwoot}/api/v1/accounts/{id_cuenta}/conversations/{id_conversacion}/messages<br>Body: content, message_type<br>Headers: api_access_token |
| 6 | **Aviso Chatwoot** | HTTP Request | Publica en Chatwoot el aviso "No reconocimos tu respuesta". | `Webhook.body.url_chatwoot`<br>`Webhook.body.id_cuenta`<br>`Webhook.body.id_conversacion`<br>`Webhook.body.api_chatwoot` | POST {url_chatwoot}/api/v1/accounts/{id_cuenta}/conversations/{id_conversacion}/messages<br>Body: content, message_type<br>Headers: api_access_token |
| 7 | **Cierre WhatsApp** | HTTP Request | Envía el mismo aviso de cierre por WhatsApp. | `Webhook.body.url_evolution`<br>`Webhook.body.session`<br>`Webhook.body.api_evolution`<br>`Webhook.body.cliente` | POST {url_evolution}/message/sendText/{session}<br>Body: number, text<br>Headers: apikey |
| 8 | **Aviso WhatsApp** | HTTP Request | Envía por WhatsApp el aviso "No reconocimos tu respuesta". | `Webhook.body.url_evolution`<br>`Webhook.body.session`<br>`Webhook.body.api_evolution`<br>`Webhook.body.cliente` | POST {url_evolution}/message/sendText/{session}<br>Body: number, text<br>Headers: apikey |
| 9 | **¿Qué menú reenviar?** | Switch | Según `estado_actual` (1–5) elige qué menú volver a mostrar; por defecto el 2. | `Webhook.body.estado_actual` | salida 1 ⇢ 1<br>salida 2 ⇢ 2<br>salida 3 ⇢ 3<br>salida 4 ⇢ 4<br>salida 5 ⇢ 5 |
| 10 | **Cerrar chat (resolved)** | HTTP Request | Cambia el estado de la conversación a `resolved` en Chatwoot (`toggle_status`). | `Webhook.body.url_chatwoot`<br>`Webhook.body.id_cuenta`<br>`Webhook.body.id_conversacion`<br>`Webhook.body.api_chatwoot` | POST {url_chatwoot}/api/v1/accounts/{id_cuenta}/conversations/{id_conversacion}/toggle_status<br>Body: status<br>Headers: api_access_token |
| 11 | **Menú 1 Chatwoot** | HTTP Request | Reenvía en Chatwoot el mensaje de bienvenida del paso 1. | `Webhook.body.url_chatwoot`<br>`Webhook.body.id_cuenta`<br>`Webhook.body.id_conversacion`<br>`Webhook.body.api_chatwoot` | POST {url_chatwoot}/api/v1/accounts/{id_cuenta}/conversations/{id_conversacion}/messages<br>Body: content, message_type<br>Headers: api_access_token |
| 12 | **Menú 4 Chatwoot1** | HTTP Request | Reenvía en Chatwoot la lista de ofertas. | `Webhook.body.url_chatwoot`<br>`Webhook.body.id_cuenta`<br>`Webhook.body.id_conversacion`<br>`Webhook.body.api_chatwoot` | POST {url_chatwoot}/api/v1/accounts/{id_cuenta}/conversations/{id_conversacion}/messages<br>Body: content, message_type<br>Headers: api_access_token |
| 13 | **Menú 3 Chatwoot1** | HTTP Request | Reenvía en Chatwoot la pregunta SI/NO de portabilidad. | `Webhook.body.url_chatwoot`<br>`Webhook.body.id_cuenta`<br>`Webhook.body.id_conversacion`<br>`Webhook.body.api_chatwoot` | POST {url_chatwoot}/api/v1/accounts/{id_cuenta}/conversations/{id_conversacion}/messages<br>Body: content, message_type<br>Headers: api_access_token |
| 14 | **Menú 2 Chatwoot1** | HTTP Request | Reenvía en Chatwoot el menú de 5 opciones. | `Webhook.body.url_chatwoot`<br>`Webhook.body.id_cuenta`<br>`Webhook.body.id_conversacion`<br>`Webhook.body.api_chatwoot` | POST {url_chatwoot}/api/v1/accounts/{id_cuenta}/conversations/{id_conversacion}/messages<br>Body: content, message_type<br>Headers: api_access_token |
| 15 | **Actualizar estado6** | MySQL | Actualiza `num_errores` y deja `paso = 1000` (conversación cerrada). | `Code.num_errores`<br>`Webhook.body.cliente` | Operación: UPDATE sobre `claro_movil`<br>Escribe: num_errores, paso, fecha_modificacion |
| 16 | **Menú 5 Chatwoot** | HTTP Request | Reenvía en Chatwoot el texto de habeas data. | `Webhook.body.url_chatwoot`<br>`Webhook.body.id_cuenta`<br>`Webhook.body.id_conversacion`<br>`Webhook.body.api_chatwoot` | POST {url_chatwoot}/api/v1/accounts/{id_cuenta}/conversations/{id_conversacion}/messages<br>Body: content, message_type<br>Headers: api_access_token |
| 17 | **Menú 1 WhatsApp** | HTTP Request | Reenvía por WhatsApp el mensaje del paso 1 ("responde SI"). | `Webhook.body.url_evolution`<br>`Webhook.body.session`<br>`Webhook.body.api_evolution`<br>`Webhook.body.cliente` | POST {url_evolution}/message/sendText/{session}<br>Body: number, text<br>Headers: apikey |
| 18 | **Menú 4 WhatsApp1** | HTTP Request | Reenvía por WhatsApp la lista de ofertas. | `Webhook.body.url_evolution`<br>`Webhook.body.session`<br>`Webhook.body.api_evolution`<br>`Webhook.body.cliente` | POST {url_evolution}/message/sendText/{session}<br>Body: number, text<br>Headers: apikey |
| 19 | **Menú 3 WhatsApp1** | HTTP Request | Reenvía por WhatsApp la pregunta SI/NO de portabilidad. | `Webhook.body.url_evolution`<br>`Webhook.body.session`<br>`Webhook.body.api_evolution`<br>`Webhook.body.cliente` | POST {url_evolution}/message/sendText/{session}<br>Body: number, text<br>Headers: apikey |
| 20 | **Menú 2 WhatsApp1** | HTTP Request | Reenvía por WhatsApp el menú de 5 opciones. | `Webhook.body.url_evolution`<br>`Webhook.body.session`<br>`Webhook.body.api_evolution`<br>`Webhook.body.cliente` | POST {url_evolution}/message/sendText/{session}<br>Body: number, text<br>Headers: apikey |
| 21 | **Menú 5 WhatsApp** | HTTP Request | Reenvía por WhatsApp el texto de habeas data. | `Webhook.body.url_evolution`<br>`Webhook.body.session`<br>`Webhook.body.api_evolution`<br>`Webhook.body.cliente` | POST {url_evolution}/message/sendText/{session}<br>Body: number, text<br>Headers: apikey |
| 22 | **Actualizar num_errores** | MySQL | Guarda el nuevo `num_errores` y actualiza `fecha_modificacion`. | `Code.num_errores`<br>`Webhook.body.cliente` | Operación: UPDATE sobre `claro_movil`<br>Escribe: num_errores, fecha_modificacion |
| 23 | **Activar inactividad** | HTTP Request | POST al webhook `/webhook/inactividad-claro-movil` con cliente, conversación y sesión (reinicia la vigilancia por inactividad). | `Webhook.body.cliente`<br>`Webhook.body.id_conversacion`<br>`Webhook.body.session` | POST https://2031510-n8n.interactivo.com.co/webhook/inactividad-claro-movil<br>Body: cliente, id_conversacion, session<br>Headers: Content-Type |

## Conexiones (de → a)
- Webhook → Obtener estado
- Obtener estado → Code
- Code → Validar cierre chat
- Validar cierre chat (sí) → Cierre Chatwoot
- Validar cierre chat (no) → Aviso Chatwoot
- Actualizar num_errores → Activar inactividad
- ¿Qué menú reenviar? (salida 0) → Menú 1 Chatwoot
- ¿Qué menú reenviar? (salida 1) → Menú 2 Chatwoot1
- ¿Qué menú reenviar? (salida 2) → Menú 3 Chatwoot1
- ¿Qué menú reenviar? (salida 3) → Menú 4 Chatwoot1
- ¿Qué menú reenviar? (salida 4) → Menú 5 Chatwoot
- ¿Qué menú reenviar? (salida 5) → Menú 2 Chatwoot1
- Menú 1 Chatwoot → Menú 1 WhatsApp
- Menú 1 WhatsApp → Actualizar num_errores
- Menú 4 Chatwoot1 → Menú 4 WhatsApp1
- Menú 3 Chatwoot1 → Menú 3 WhatsApp1
- Menú 2 Chatwoot1 → Menú 2 WhatsApp1
- Menú 2 WhatsApp1 → Actualizar num_errores
- Menú 4 WhatsApp1 → Actualizar num_errores
- Menú 3 WhatsApp1 → Actualizar num_errores
- Cerrar chat (resolved) → Actualizar estado6
- Cierre WhatsApp → Cerrar chat (resolved)
- Cierre Chatwoot → Cierre WhatsApp
- Aviso WhatsApp → ¿Qué menú reenviar?
- Aviso Chatwoot → Aviso WhatsApp
- Menú 5 Chatwoot → Menú 5 WhatsApp
- Menú 5 WhatsApp → Actualizar num_errores

## Intervención para WAHA
| Nodo | Qué cambiar |
|---|---|
| Menú 1 WhatsApp | Envío: URL `/message/sendText/{session}` → `/api/sendText`; header `apikey` → `X-Api-Key`; body `number` → `chatId` (`…@c.us`) + `session`. |
| Activar inactividad | Dominio n8n `2031510-n8n…` vs `n8n-general…` (donde apunta WAHA): confirmar cuál es el vigente. |
| Menú 4 WhatsApp1 | Envío: URL `/message/sendText/{session}` → `/api/sendText`; header `apikey` → `X-Api-Key`; body `number` → `chatId` (`…@c.us`) + `session`. |
| Menú 3 WhatsApp1 | Envío: URL `/message/sendText/{session}` → `/api/sendText`; header `apikey` → `X-Api-Key`; body `number` → `chatId` (`…@c.us`) + `session`. |
| Menú 2 WhatsApp1 | Envío: URL `/message/sendText/{session}` → `/api/sendText`; header `apikey` → `X-Api-Key`; body `number` → `chatId` (`…@c.us`) + `session`. |
| Cierre WhatsApp | Envío: URL `/message/sendText/{session}` → `/api/sendText`; header `apikey` → `X-Api-Key`; body `number` → `chatId` (`…@c.us`) + `session`. |
| Aviso WhatsApp | Envío: URL `/message/sendText/{session}` → `/api/sendText`; header `apikey` → `X-Api-Key`; body `number` → `chatId` (`…@c.us`) + `session`. |
| Menú 5 WhatsApp | Envío: URL `/message/sendText/{session}` → `/api/sendText`; header `apikey` → `X-Api-Key`; body `number` → `chatId` (`…@c.us`) + `session`. |

Volver a [Documentación de n8n](<../Documentacion de n8n.md>)
