# 04 · Inactividad

Parte de [Documentación de n8n](<../Documentacion de n8n.md>) · Canvas: [Inactividad.canvas](Inactividad.canvas) · Fuente: `n8n-json/CLARO_MOVIL inactividad.json`

## Qué hace
Vigilante por tiempo. Cada minuto busca clientes en pasos 1–5 sin responder, envía hasta 2 recordatorios (con el menú del paso) y al 3.º aviso cierra la sesión por inactividad.

| | |
|---|---|
| **Disparador** | Cron cada 1 minuto. |
| **Llama a** | Ninguno |
| **Es llamado por** | Cron propio (también existe una llamada a `/webhook/inactividad-claro-movil` desde Opción inválida, pero este workflow no tiene nodo Webhook) |
| **Nodos** | 11 |

## Nodos
| # | Nodo | Tipo | Qué hace | Lee | Maneja / produce |
|---|---|---|---|---|---|
| 1 | **Cron 1 min** | Cron | Dispara el flujo cada minuto. | — | Cada 1 min |
| 2 | **Variables** | Set (variables) | Define tipo, cuenta, inbox, URLs, claves y sesión por defecto. | — | tipo = CLARO_MOVIL<br>id_cuenta = 1<br>inbox_id = 13<br>api_chatwoot = <clave oculta><br>url_chatwoot = https://botconnect.interactivo.com.co:8443<br>api_evolution = <clave oculta> |
| 3 | **Buscar inactivos** | MySQL | Consulta clientes en paso 1–5 con menos de 3 advertencias y con más minutos sin actividad que el umbral `(num_advertencia+1) × 1 min`. | — | Operación: SELECT sobre `claro_movil`<br>Lee: cliente, id_conversacion, num_advertencia, session_wpp, paso, TIMESTAMPDIFF(MINUTE, fecha_modificacion, NOW()) AS minuto |
| 4 | **Preparar recordatorio** | Code (JS) | Calcula el nivel de aviso (1.º, 2.º o 3.º), arma el recordatorio y el menú del paso; en el 3.º marca cierre y `paso_nuevo = 1000`. | — | Devuelve: cliente, num_advertencia_nuevo, paso_nuevo, es_cierre, texto, es_ultimo |
| 5 | **Enviar WhatsApp** | HTTP Request | Envía el recordatorio al cliente usando su sesión guardada (`session_wpp`) o la por defecto. | `Variables.url_evolution`<br>`Variables.session`<br>`Variables.api_evolution`<br>`(entrada).session_wpp`<br>`(entrada).cliente`<br>`(entrada).texto` | POST {url_evolution}/message/sendText/{session_wpp}<br>Body: number, text<br>Headers: apikey |
| 6 | **¿Último mensaje?** | IF | Si es el último texto del lote (índice final) actualiza la advertencia en MySQL. | `Preparar recordatorio.es_ultimo` | $('Preparar recordatorio').item.json.es_ultimo === true |
| 7 | **Actualizar advertencia** | MySQL | Guarda `num_advertencia` y `paso` nuevos y renueva `fecha_modificacion`. | `Preparar recordatorio.num_advertencia_nuevo`<br>`Preparar recordatorio.paso_nuevo`<br>`Preparar recordatorio.cliente`<br>`Preparar recordatorio.id_conversacion` | Operación: UPDATE sobre `claro_movil`<br>Escribe: num_advertencia, paso, fecha_modificacion |
| 8 | **Wait** | Wait | Espera 1,5 s entre mensajes del mismo cliente. | — | Espera 1.5 s |
| 9 | **Registrar en Chatwoot** | HTTP Request | Replica el recordatorio enviado en la conversación de Chatwoot. | `Variables.url_chatwoot`<br>`Preparar recordatorio.id_conversacion`<br>`Variables.api_chatwoot`<br>`Preparar recordatorio.texto` | POST {url_chatwoot}/api/v1/accounts/1/conversations/{id_conversacion}/messages<br>Body: content, message_type<br>Headers: api_access_token |
| 10 | **Es cierre?** | IF | Si es el 3.er aviso continúa a cerrar el chat en Chatwoot. | `Preparar recordatorio.es_cierre` | $('Preparar recordatorio').item.json.es_cierre === true |
| 11 | **Cerrar chat (resolved)** | HTTP Request | Cambia el estado de la conversación a `resolved` en Chatwoot (`toggle_status`). | `Variables.url_chatwoot`<br>`Preparar recordatorio.id_conversacion`<br>`Variables.api_chatwoot` | POST {url_chatwoot}/api/v1/accounts/1/conversations/{id_conversacion}/toggle_status<br>Body: status<br>Headers: api_access_token |

## Conexiones (de → a)
- Cron 1 min → Variables
- Variables → Buscar inactivos
- Buscar inactivos → Preparar recordatorio
- Preparar recordatorio → Enviar WhatsApp
- Preparar recordatorio → ¿Último mensaje?
- Enviar WhatsApp → Wait
- Actualizar advertencia → Es cierre?
- Es cierre? (sí) → Cerrar chat (resolved)
- ¿Último mensaje? (sí) → Actualizar advertencia
- Wait → Registrar en Chatwoot

## Intervención para WAHA
| Nodo | Qué cambiar |
|---|---|
| Variables | Variables: `url_evolution`/`api_evolution` → URL y clave de WAHA; sesión `AGENTE_PRUEBA_1` (ya coincide). |
| Enviar WhatsApp | Envío: URL `/message/sendText/{session}` → `/api/sendText`; header `apikey` → `X-Api-Key`; body `number` → `chatId` (`…@c.us`) + `session`. |

Volver a [Documentación de n8n](<../Documentacion de n8n.md>)
