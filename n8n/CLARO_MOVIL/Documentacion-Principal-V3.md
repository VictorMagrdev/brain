# 01 · Principal V3

Parte de [Documentación de n8n](<../Documentacion de n8n.md>) · Canvas: [Principal-V3.canvas](Principal-V3.canvas) · Fuente: `n8n-json/CLARO_MOVIL Principal V3 (Evolution).json`

## Qué hace
Cerebro del bot. Recibe cada mensaje que entra por WhatsApp, lo registra en Chatwoot, consulta en qué paso del menú está el cliente (tabla `claro_movil`), decide la respuesta, la envía por WhatsApp y Chatwoot, guarda el nuevo paso y, si aplica, asigna un asesor.

| | |
|---|---|
| **Disparador** | Webhook POST `/webhook/bot-claro-movil` (hoy lo llama Evolution). |
| **Llama a** | `opcion-invalida-claro-movil` (respuesta no válida)<br>`cambio-estado-conversacion-claro-movil` (nodo Validacion error1, cuando el paso no es 0–5) |
| **Es llamado por** | Evolution (webhook de mensajes entrantes) |
| **Nodos** | 61 |

## Nodos
| # | Nodo | Tipo | Qué hace | Lee | Maneja / produce |
|---|---|---|---|---|---|
| 1 | **Webhook entrada** | Webhook | Punto de entrada del bot. Recibe el POST con el mensaje de WhatsApp. Hoy el cuerpo trae el formato de Evolution (`body.data…`). | — | POST /webhook/bot-claro-movil |
| 2 | **Es texto entrante?** | IF | Filtro anti-eco: deja pasar solo mensajes que NO son propios (`fromMe` distinto de true). Evita que el bot se responda a sí mismo. | `Webhook entrada.body.data.key.fromMe` | $('Webhook entrada').item.json.body?.data?.key?.fromMe !== true && !$('Webhook entrada').i |
| 3 | **Es ubicacion?** | IF | Detecta si el mensaje es una ubicación (estática, en vivo o dentro de `deviceSentMessage`). Si sí va al camino de ubicación; si no, al de texto. | `Webhook entrada.body.data.message.locationMessage`<br>`Webhook entrada.body.data.message.liveLocationMessage`<br>`Webhook entrada.body.data.message.deviceSentMessage.message.locationMessage`<br>`Webhook entrada.body.data.message.deviceSentMessage.message.liveLocationMessage` | !!($('Webhook entrada').item.json.body?.data?.message?.locationMessage // $('Webhook entra |
| 4 | **Variables globales** | Set (variables) | Fija `tipo` = CLARO_MOVIL, la `session` de WhatsApp y guarda el payload completo recibido. | `Webhook entrada.body` | tipo = CLARO_MOVIL<br>session = AGENTE_PRUEBA_1<br>payload = {{ $('Webhook entrada').item.json.body }} |
| 5 | **Variables ubicacion** | Set (variables) | Define las variables del camino de ubicación (URLs, claves, sesión, inbox) porque este camino no pasa por `Variables del menú`. | — | tipo = CLARO_MOVIL<br>id_cuenta = 1<br>api_chatwoot = <clave oculta><br>url_chatwoot = https://botconnect.interactivo.com.co:8443<br>api_evolution = <clave oculta><br>url_evolution = https://evolution-general.interactivo.com.co |
| 6 | **Obtener estado** | MySQL | Consulta en MySQL el último registro del cliente (por teléfono y tipo) para conocer su `paso`, conversación, inbox y etiqueta. | `Webhook entrada.body.data.key.remoteJid.split`<br>`Variables globales.tipo` | Operación: SELECT sobre `claro_movil`<br>Lee: * |
| 7 | **Normalizar ubicacion** | Code (JS) | Extrae latitud, longitud, nombre y dirección; valida que sean números, estén en rango y no sean 0,0; arma el link de Google Maps y detecta si es en vivo. | `Webhook entrada.body` | Devuelve: esUbicacion, valida, lat, lng, enVivo, cliente, mapsUrl |
| 8 | **Variables del menú** | Set (variables) | Hub de configuración del flujo: junta estado actual (`paso`), ids de Chatwoot, URLs, claves, sesión, inbox y URL de n8n. Casi todos los nodos leen de aquí. | `(entrada).paso`<br>`(entrada).id_conversacion`<br>`(entrada).etiqueta_chatwoot` | tipo = CLARO_MOVIL<br>estado_actual = {{ $json.paso // '0' }}<br>id_cuenta = 1<br>id_conversacion = {{ $json.id_conversacion // '' }}<br>etiqueta_chatwoot = {{ $json.etiqueta_chatwoot // '' }}<br>api_chatwoot = <clave oculta> |
| 9 | **Coordenadas validas?** | IF | Bifurca: coordenadas válidas → geocodificar; inválidas → avisar al cliente. | `(entrada).valida` | $json.valida === true |
| 10 | **Registrar entrante Chatwoot1** | HTTP Request | Escribe en Chatwoot el mensaje que mandó el cliente (texto o aviso de ubicación) como `incoming`, en la conversación existente. | `Variables del menú.url_chatwoot`<br>`Variables del menú.id_conversacion`<br>`Variables del menú.api_chatwoot`<br>`Webhook entrada.body.data.message.conversation`<br>`Webhook entrada.body.data.message.extendedTextMessage.text` | POST {url_chatwoot}/api/v1/accounts/1/conversations/{id_conversacion}/messages<br>Body: content, message_type<br>Headers: api_access_token |
| 11 | **Geocodificar** | HTTP Request | Consulta Nominatim (OpenStreetMap, reverse) para obtener departamento y municipio a partir de lat/lng. | `(entrada).lat`<br>`(entrada).lng` | GET https://nominatim.openstreetmap.org/reverse<br>Headers: User-Agent<br>Query: format, lat, lon, zoom, addressdetails, accept-language |
| 12 | **Error ubicacion** | HTTP Request | Envía por WhatsApp el aviso "No pudimos procesar la ubicación" cuando las coordenadas no son válidas. | `Variables ubicacion.url_evolution`<br>`Variables ubicacion.session`<br>`Variables ubicacion.api_evolution`<br>`Normalizar ubicacion.cliente` | POST {url_evolution}/message/sendText/{session}<br>Body: number, text<br>Headers: apikey |
| 13 | **Enrutar por estado** | Switch | Switch principal de la máquina de estados: según `estado_actual` manda a bienvenida (0/1000), validación de SI (1), menú 1–5 (2), SI/NO portabilidad (3), planes 1–4 (4), habeas (5) o a Validacion error1 (cualquier otro, p. ej. -1). | `Variables del menú.estado_actual` | salida 0 ⇢ 0 / 1000<br>salida 1 ⇢ 1<br>salida 2 ⇢ 2<br>salida 3 ⇢ 3<br>salida 4 ⇢ 4<br>salida 5 ⇢ 5 |
| 14 | **Formato ubicacion** | Code (JS) | Arma el texto legible de la ubicación (tipo, lugar, dirección aproximada, coordenadas y link al mapa) sin saltos ni comillas para no romper el JSON. | — | Devuelve: municipio, lat, lng, mapsUrl, cliente, enVivo, departamento, contenido |
| 15 | **Validar plan (1-4)** | IF | Paso 4: acepta "1" a "4". Otra cosa → opción inválida. | `Webhook entrada.body.data.message.conversation` | ['1','2','3','4'].includes(($('Webhook entrada').item.json.body.data.message.conversation  |
| 16 | **¿SI o NO?** | IF | Paso 3: valida SI/NO. Otra cosa → opción inválida. | `Webhook entrada.body.data.message.conversation` | /(^/\s)(si/sí/s/no/n)(\s/$/\.)/i.test(($('Webhook entrada').item.json.body.data.message.co |
| 17 | **Validar opción (1-5)** | IF | Paso 2: acepta solo "1" a "5". Otra cosa → opción inválida. | `Webhook entrada.body.data.message.conversation` | ['1','2','3','4','5'].includes(($('Webhook entrada').item.json.body.data.message.conversat |
| 18 | **¿Dijo SI?** | IF | Paso 1: valida con regex que la respuesta sea SI/S/sí. Si no, llama a opción inválida. | `Webhook entrada.body.data.message.conversation` | /(^/\s)(si/sí/s)(\s/$/\.)/i.test(($('Webhook entrada').item.json.body.data.message.convers |
| 19 | **GET CONTACT** | HTTP Request | Busca el contacto en Chatwoot por número de teléfono. | `Variables del menú.url_chatwoot`<br>`Webhook entrada.body.data.key.remoteJid.split`<br>`Variables del menú.api_chatwoot` | GET {url_chatwoot}/api/v1/accounts/1/contacts/search?q={split}<br>Headers: api_access_token |
| 20 | **Validacion error1** | HTTP Request | Camino de "paso desconocido" (p. ej. -1, conversación con asesor): POST al workflow de cambio de estado enviando todas las variables. | — | POST https://2031510-n8n.interactivo.com.co/webhook/cambio-estado-conversacion-claro-movil<br>Body:  |
| 21 | **Obtener conversacion ubic** | MySQL | Busca en MySQL el `id_conversacion` y `id_cuenta` del cliente para poder escribir en su conversación de Chatwoot. | `Normalizar ubicacion.cliente` | Operación: SELECT sobre `claro_movil`<br>Lee: id_conversacion, id_cuenta |
| 22 | **¿Habeas válido?** | IF | Paso 5: valida que la respuesta sea SI/NO. Otra cosa → opción inválida. | `Webhook entrada.body.data.message.conversation` | /(^/\s)(si/sí/s/no/n)(\s/$/\.)/i.test(($('Webhook entrada').item.json.body.data.message.co |
| 23 | **Plan → asesor (handoff)** | Set (variables) | Mensaje de solicitud registrada y traspaso a asesor; `paso = -1`, etiqueta `claro-terminales`, `handoff = true`. | — | texto = Tu solicitud ha sido registrada correctamente.\n\nPróximamente, uno de<br>paso_nuevo = -1<br>etiqueta = claro-terminales<br>handoff = true |
| 24 | **Migración (handoff)** | Set (variables) | Mensaje de traspaso a asesor de migración; `paso = -1`, etiqueta `claro-migracion`, `handoff = true`. | — | texto = Gracias por la información.\n\nA continuación, serás atendido por un a<br>paso_nuevo = -1<br>etiqueta = claro-migracion<br>handoff = true |
| 25 | **Switch Opción** | Switch | Enruta por la opción elegida: 1 deuda, 2 y 3 portabilidad (habeas), 4 ofertas, 5 *611; otro → opción inválida. | `Webhook entrada.body.data.message.conversation` | salida 1 ⇢ 1<br>salida 2 ⇢ 2<br>salida 3 ⇢ 3<br>salida 4 ⇢ 4<br>salida 5 ⇢ 5 |
| 26 | **Menu** | Set (variables) | Prepara el menú de 5 opciones (texto) y pasa a `paso = 2`. | — | texto = Para continuar, responde con el número de la opción que corresponda:\n<br>paso_nuevo = 2<br>etiqueta = <br>handoff = false |
| 27 | **Filtro Inbox** | Switch | Verifica que el inbox sea el 13 (CLARO_MOVIL); si no, el flujo se detiene. | `Variables del menú.inbox_id` | salida inbox13 ⇢ 13 |
| 28 | **Ubicacion a Chatwoot** | HTTP Request | Publica la ubicación formateada como mensaje entrante en la conversación de Chatwoot. | `Variables ubicacion.url_chatwoot`<br>`Variables ubicacion.id_cuenta`<br>`Variables ubicacion.api_chatwoot`<br>`Formato ubicacion.contenido`<br>`(entrada).id_conversacion` | POST {url_chatwoot}/api/v1/accounts/{id_cuenta}/conversations/{id_conversacion}/messages<br>Body: content, message_type<br>Headers: Content-Type, api_access_token |
| 29 | **¿Aceptó habeas?** | IF | Si aceptó (SI) continúa a portabilidad; si dijo NO, va a Rechazo habeas. | `Webhook entrada.body.data.message.conversation` | /(^/\s)(si/sí/s)(\s/$/\.)/i.test(($('Webhook entrada').item.json.body.data.message.convers |
| 30 | **Llamar opción inválida** | HTTP Request | POST al workflow Opción inválida enviando teléfono, conversación, paso actual, URLs, claves y sesión. | `Variables del menú.url_n8n`<br>`Webhook entrada.body.data.key.remoteJid.split`<br>`Variables del menú.id_conversacion`<br>`Variables del menú.estado_actual`<br>`Variables del menú.url_chatwoot`<br>`Variables del menú.api_chatwoot`<br>… | POST {url_n8n}/webhook/opcion-invalida-claro-movil<br>Body: cliente, id_conversacion, id_cuenta, tipo, estado_actual, url_chatwoot, api_chatwoot, url_evolution, api_evolution, session<br>Headers: Content-Type |
| 31 | **Opción 5 (*611)** | Set (variables) | Indica comunicarse con la línea *611 y cierra (`paso 1000`). | — | texto = De acuerdo. 😊\n\nPara este caso, te invitamos a comunicarte con la lí<br>paso_nuevo = 1000<br>etiqueta = <br>handoff = false |
| 32 | **Opción 4 (ofertas)** | Set (variables) | Muestra los 3 planes y la opción 4 de asesor; `paso = 4`, etiqueta `claro-terminales`. | — | texto = 🎉 ¡Tenemos ofertas para ti!\n\nElige la opción que más te interese es<br>paso_nuevo = 4<br>etiqueta = claro-terminales<br>handoff = false |
| 33 | **Opción 2-3 (portab)** | Set (variables) | Pregunta si el número a portar recibe llamadas (SI/NO); `paso = 3`, etiqueta `claro-portabilidad`. | — | texto = Para continuar, confírmanos si el número que deseas portar recibe llam<br>paso_nuevo = 3<br>etiqueta = claro-portabilidad<br>handoff = false |
| 34 | **Opción 1 (deuda)** | Set (variables) | Respuesta: no se puede continuar por deuda pendiente. Cierra el ciclo (`paso 1000`). | — | texto = Entendemos tu interés.\n\nEn este momento no es posible continuar con <br>paso_nuevo = 1000<br>etiqueta = <br>handoff = false |
| 35 | **Si Contacto Existe** | IF | Bifurca: si Chatwoot encontró contacto (`meta.count ≠ 0`) busca sus conversaciones; si no, lo crea. | `(entrada).meta.count` | $json.meta.count  ==  0 |
| 36 | **Trazabilidad ubicacion** | MySQL | Guarda un evento `ubicacion_recibida` (lat, lng, en_vivo, municipio, departamento, fecha) en la columna JSON `ori_fields`. | `Formato ubicacion.lat`<br>`Formato ubicacion.lng`<br>`Formato ubicacion.enVivo`<br>`Formato ubicacion.municipio`<br>`Formato ubicacion.departamento`<br>`Formato ubicacion.cliente` | Operación: UPDATE sobre `claro_movil`<br>Escribe: ori_fields |
| 37 | **Habeas 2-3** | Set (variables) | Pide autorización de tratamiento de datos (habeas data) y pasa a `paso = 5`; etiqueta `claro-portabilidad`. | — | — |
| 38 | **Rechazo habeas** | Set (variables) | Agradece y cierra (`paso 1000`) porque el cliente no autorizó el tratamiento de datos. | — | — |
| 39 | **Consolidar** | Code (JS) | Nodo de unión: junta `texto`, `paso_nuevo`, `etiqueta`, `handoff`, `phone` y `conversation_id` que producen todas las ramas del menú. | `Webhook entrada.body`<br>`Variables del menú.id_conversacion`<br>`(entrada).texto`<br>`(entrada).paso_nuevo`<br>`(entrada).etiqueta`<br>`(entrada).handoff` | Devuelve: texto, paso_nuevo, etiqueta, handoff, phone, conversation_id |
| 40 | **Crea Contacto** | HTTP Request | Crea el contacto en Chatwoot con teléfono, identificador, `source_id`, inbox 13 y atributo `bot = on`. | `Variables del menú.url_chatwoot`<br>`Variables del menú.api_chatwoot`<br>`Webhook entrada.body.data.key.remoteJid.split`<br>`Variables del menú.inbox_id` | POST {url_chatwoot}/api/v1/accounts/1/contacts<br>Body: name, phone_number, identifier, inbox_id, source_id, custom_attributes, bot<br>Headers: api_access_token |
| 41 | **Conversaciones del contacto** | HTTP Request | Lista las conversaciones del contacto en Chatwoot para ver si ya tiene una abierta. | `Variables del menú.url_chatwoot`<br>`GET CONTACT.payload[0].id`<br>`Variables del menú.api_chatwoot` | GET {url_chatwoot}/api/v1/accounts/1/contacts/{payload}/conversations<br>Headers: api_access_token |
| 42 | **Guardar estado** | MySQL | UPSERT en `claro_movil` con el nuevo `paso`, etiqueta y sesión; reinicia advertencias y errores. | `Consolidar.phone`<br>`Variables del menú.inbox_id`<br>`Consolidar.conversation_id`<br>`Consolidar.paso_nuevo`<br>`Variables del menú.session`<br>`Consolidar.etiqueta` | Operación: INSERT sobre `claro_movil`<br>Escribe: cliente,tipo,id_cuenta,id_inbox,id_conversacion,paso,num_advertencia,num_errores,session_wpp,etiqueta_chatwoot,fecha_modificacion |
| 43 | **Crear Conversación1** | HTTP Request | Crea la conversación en Chatwoot usando el contacto recién creado. | `Variables del menú.url_chatwoot`<br>`Variables del menú.api_chatwoot`<br>`Webhook entrada.body.data.key.remoteJid.split`<br>`Variables del menú.inbox_id`<br>`(entrada).payload.contact.id` | POST {url_chatwoot}/api/v1/accounts/1/conversations<br>Body: source_id, inbox_id, contact_id<br>Headers: api_access_token |
| 44 | **¿Reutilizar conversación?** | Code (JS) | Decide reutilizar una conversación `open/pending` del inbox 13. Si el estado previo es 1000 (cerrada) nunca reutiliza. | `Obtener estado.paso`<br>`Variables del menú.inbox_id` | Devuelve: reutiliza, id |
| 45 | **Enviar WhatsApp** | HTTP Request | Envía por WhatsApp el texto armado por la rama del menú. | `Variables del menú.url_evolution`<br>`Variables del menú.session`<br>`Variables del menú.api_evolution`<br>`Consolidar.phone`<br>`Consolidar.texto` | POST {url_evolution}/message/sendText/{session}<br>Body: number, text<br>Headers: apikey |
| 46 | **¿Ya tiene abierta?** | IF | Bifurca: con conversación abierta continúa con ella; si no, crea una nueva. | `(entrada).reutiliza` | $json.reutiliza === true |
| 47 | **Crear Conversación** | HTTP Request | Crea una conversación nueva para el contacto existente que no tenía una abierta. | `Variables del menú.url_chatwoot`<br>`Variables del menú.api_chatwoot`<br>`Webhook entrada.body.data.key.remoteJid.split`<br>`Variables del menú.inbox_id`<br>`GET CONTACT.payload[0].id` | POST {url_chatwoot}/api/v1/accounts/1/conversations<br>Body: source_id, inbox_id, contact_id<br>Headers: api_access_token |
| 48 | **Enviar Chatwoot** | HTTP Request | Replica ese mismo texto en la conversación de Chatwoot como `outgoing` con `source_id = bot`. | `Variables del menú.url_chatwoot`<br>`Consolidar.conversation_id`<br>`Variables del menú.api_chatwoot`<br>`Consolidar.texto` | POST {url_chatwoot}/api/v1/accounts/1/conversations/{conversation_id}/messages<br>Body: content, message_type, source_id<br>Headers: api_access_token |
| 49 | **¿Handoff?** | IF | Si la rama marcó `handoff = true` pasa a asignar asesor; si no, termina. | `Consolidar.handoff` | $('Consolidar').item.json.handoff === true |
| 50 | **Wait** | Wait | Espera 3 s para que Chatwoot termine de crear la conversación. | — | Espera 3 s |
| 51 | **Inbox members** | HTTP Request | Lista los agentes del inbox 13 en Chatwoot. | `Variables del menú.url_chatwoot`<br>`Variables del menú.inbox_id`<br>`Variables del menú.api_chatwoot` | GET {url_chatwoot}/api/v1/accounts/1/inbox_members/{inbox_id}<br>Headers: api_access_token |
| 52 | **Ctx inicial** | Code (JS) | Toma el id de conversación (falla con error si Chatwoot no lo devuelve) y el teléfono; es el contexto del primer mensaje. | `Webhook entrada.body.data.key.remoteJid.split`<br>`(entrada).id` | Devuelve: conversation_id, phone |
| 53 | **Agentes online** | Code (JS) | Filtra los agentes con `availability_status = online`. | — | Devuelve: agentsFound, availableAgents |
| 54 | **Guardar estado inicial** | MySQL | UPSERT en `claro_movil`: deja `paso = 1`, reinicia advertencias y errores, y guarda conversación, inbox y sesión. | `Variables del menú.inbox_id`<br>`Variables del menú.session`<br>`(entrada).phone`<br>`(entrada).conversation_id` | Operación: INSERT sobre `claro_movil`<br>Escribe: cliente,tipo,id_cuenta,id_inbox,id_conversacion,paso,num_advertencia,num_errores,session_wpp,fecha_modificacion |
| 55 | **Registrar entrante Chatwoot (inicial)** | HTTP Request | Registra en Chatwoot el primer mensaje del cliente como `incoming`. | `Variables del menú.url_chatwoot`<br>`Ctx inicial.conversation_id`<br>`Variables del menú.api_chatwoot`<br>`Webhook entrada.body.data.message.conversation` | POST {url_chatwoot}/api/v1/accounts/1/conversations/{conversation_id}/messages<br>Body: content, message_type<br>Headers: api_access_token |
| 56 | **Conversaciones totales** | HTTP Request | Trae las conversaciones abiertas asignadas del inbox con etiqueta `asignado-asesor` para medir la carga. | `Variables del menú.url_chatwoot`<br>`Variables del menú.inbox_id`<br>`Variables del menú.api_chatwoot` | GET {url_chatwoot}/api/v1/accounts/1/conversations?assignee_type=assigned&status=open&inbox_id={inbox_id}&labels[]=asignado-asesor&page=1<br>Headers: api_access_token |
| 57 | **Menos cargado** | Code (JS) | Elige al agente online con menos conversaciones. Si no hay agentes online lanza error. | `Agentes online.availableAgents` | Devuelve: agentId, id |
| 58 | **Bienvenida WhatsApp** | HTTP Request | Envía por WhatsApp el saludo de bienvenida de Claro pidiendo responder SI. | `Variables del menú.url_evolution`<br>`Variables del menú.session`<br>`Variables del menú.api_evolution`<br>`Ctx inicial.phone` | POST {url_evolution}/message/sendText/{session}<br>Body: number, text, message_type<br>Headers: apikey |
| 59 | **Bienvenida Chatwoot** | HTTP Request | Deja el mismo saludo de bienvenida en la conversación de Chatwoot como `outgoing`. | `Variables del menú.url_chatwoot`<br>`Ctx inicial.conversation_id`<br>`Variables del menú.api_chatwoot` | POST {url_chatwoot}/api/v1/accounts/1/conversations/{conversation_id}/messages<br>Body: content, message_type<br>Headers: api_access_token |
| 60 | **Asignar agente** | HTTP Request | Asigna la conversación al agente elegido en Chatwoot. | `Variables del menú.url_chatwoot`<br>`Consolidar.conversation_id`<br>`Variables del menú.api_chatwoot`<br>`(entrada).agentId` | POST {url_chatwoot}/api/v1/accounts/1/conversations/{conversation_id}/assignments<br>Body: assignee_id<br>Headers: api_access_token |
| 61 | **Etiqueta asesor** | HTTP Request | Agrega a la conversación la etiqueta definida en la rama (`claro-terminales`, `claro-migracion`…). | `Variables del menú.url_chatwoot`<br>`Consolidar.conversation_id`<br>`Variables del menú.api_chatwoot`<br>`Consolidar.etiqueta` | POST {url_chatwoot}/api/v1/accounts/1/conversations/{conversation_id}/labels<br>Body: labels<br>Headers: api_access_token |

## Conexiones (de → a)
- Registrar entrante Chatwoot (inicial) → Bienvenida WhatsApp
- Asignar agente → Etiqueta asesor
- Menos cargado → Asignar agente
- Conversaciones totales → Menos cargado
- Agentes online → Conversaciones totales
- Inbox members → Agentes online
- ¿Handoff? (sí) → Inbox members
- Enviar WhatsApp → Enviar Chatwoot
- Guardar estado → Enviar WhatsApp
- Consolidar → Guardar estado
- Plan → asesor (handoff) → Consolidar
- Validar plan (1-4) (sí) → Plan → asesor (handoff)
- Validar plan (1-4) (no) → Llamar opción inválida
- Migración (handoff) → Consolidar
- ¿SI o NO? (sí) → Migración (handoff)
- ¿SI o NO? (no) → Llamar opción inválida
- Opción 5 (*611) → Consolidar
- Opción 4 (ofertas) → Consolidar
- Opción 2-3 (portab) → Consolidar
- Opción 1 (deuda) → Consolidar
- Switch Opción (salida 0) → Opción 1 (deuda)
- Switch Opción (salida 1) → Habeas 2-3
- Switch Opción (salida 2) → Habeas 2-3
- Switch Opción (salida 3) → Opción 4 (ofertas)
- Switch Opción (salida 4) → Opción 5 (*611)
- Switch Opción (salida 5) → Llamar opción inválida
- Validar opción (1-5) (sí) → Switch Opción
- Validar opción (1-5) (no) → Llamar opción inválida
- Menu → Consolidar
- ¿Dijo SI? (sí) → Menu
- ¿Dijo SI? (no) → Llamar opción inválida
- Bienvenida WhatsApp → Bienvenida Chatwoot
- Guardar estado inicial → Registrar entrante Chatwoot (inicial)
- Ctx inicial → Guardar estado inicial
- Crear Conversación1 → Ctx inicial
- Crea Contacto → Crear Conversación1
- Crear Conversación → Wait
- Si Contacto Existe (sí) → Conversaciones del contacto
- Si Contacto Existe (no) → Crea Contacto
- GET CONTACT → Filtro Inbox
- Enrutar por estado (salida 0) → GET CONTACT
- Enrutar por estado (salida 1) → ¿Dijo SI?
- Enrutar por estado (salida 2) → Validar opción (1-5)
- Enrutar por estado (salida 3) → ¿SI o NO?
- Enrutar por estado (salida 4) → Validar plan (1-4)
- Enrutar por estado (salida 5) → ¿Habeas válido?
- Enrutar por estado (salida 6) → Validacion error1
- Variables del menú → Registrar entrante Chatwoot1
- Obtener estado → Variables del menú
- Variables globales → Obtener estado
- Es texto entrante? (sí) → Es ubicacion?
- Webhook entrada → Es texto entrante?
- Filtro Inbox (salida 0) → Si Contacto Existe
- Wait → Ctx inicial
- Registrar entrante Chatwoot1 → Enrutar por estado
- Es ubicacion? (sí) → Variables ubicacion
- Es ubicacion? (no) → Variables globales
- Normalizar ubicacion → Coordenadas validas?
- Coordenadas validas? (sí) → Geocodificar
- Coordenadas validas? (no) → Error ubicacion
- Geocodificar → Formato ubicacion
- Formato ubicacion → Obtener conversacion ubic
- Obtener conversacion ubic → Ubicacion a Chatwoot
- Ubicacion a Chatwoot → Trazabilidad ubicacion
- Variables ubicacion → Normalizar ubicacion
- Conversaciones del contacto → ¿Reutilizar conversación?
- ¿Reutilizar conversación? → ¿Ya tiene abierta?
- ¿Ya tiene abierta? (sí) → Ctx inicial
- ¿Ya tiene abierta? (no) → Crear Conversación
- Enviar Chatwoot → ¿Handoff?
- Habeas 2-3 → Consolidar
- ¿Habeas válido? (sí) → ¿Aceptó habeas?
- ¿Habeas válido? (no) → Llamar opción inválida
- ¿Aceptó habeas? (sí) → Opción 2-3 (portab)
- ¿Aceptó habeas? (no) → Rechazo habeas
- Rechazo habeas → Consolidar

## Intervención para WAHA
| Nodo | Qué cambiar |
|---|---|
| Registrar entrante Chatwoot (inicial) | Texto: `body.data.message.conversation` → `body.payload.body`. |
| Llamar opción inválida | Teléfono: `body.data.key.remoteJid` → `body.payload.from`.<br>Variables: renombrar `url_evolution`/`api_evolution` por las de WAHA (se pasan al sub-flujo). |
| Enviar WhatsApp | Envío: URL `/message/sendText/{session}` → `/api/sendText`; header `apikey` → `X-Api-Key`; body `number` → `chatId` (`…@c.us`) + `session`. |
| Consolidar | Teléfono: `body.data.key.remoteJid` → `body.payload.from`. |
| Validar plan (1-4) | Texto: `body.data.message.conversation` → `body.payload.body`. |
| ¿SI o NO? | Texto: `body.data.message.conversation` → `body.payload.body`. |
| Switch Opción | Texto: `body.data.message.conversation` → `body.payload.body`. |
| Validar opción (1-5) | Texto: `body.data.message.conversation` → `body.payload.body`. |
| ¿Dijo SI? | Texto: `body.data.message.conversation` → `body.payload.body`. |
| Bienvenida WhatsApp | Envío: URL `/message/sendText/{session}` → `/api/sendText`; header `apikey` → `X-Api-Key`; body `number` → `chatId` (`…@c.us`) + `session`. |
| Ctx inicial | Teléfono: `body.data.key.remoteJid` → `body.payload.from`. |
| Crear Conversación1 | Teléfono: `body.data.key.remoteJid` → `body.payload.from`. |
| Crea Contacto | Teléfono: `body.data.key.remoteJid` → `body.payload.from`. |
| Crear Conversación | Teléfono: `body.data.key.remoteJid` → `body.payload.from`. |
| GET CONTACT | Teléfono: `body.data.key.remoteJid` → `body.payload.from`. |
| Variables del menú | Variables: `url_evolution`/`api_evolution` → URL y clave de WAHA; sesión `AGENTE_PRUEBA_1` (ya coincide).<br>Dominio n8n `2031510-n8n…` vs `n8n-general…` (donde apunta WAHA): confirmar cuál es el vigente. |
| Obtener estado | Teléfono: `body.data.key.remoteJid` → `body.payload.from`. |
| Es texto entrante? | Propio: `data.key.fromMe` → `payload.fromMe` (+ filtrar `event = message`). |
| Webhook entrada | Ruta: `bot-claro-movil` → `recepcion-whatsapp-claro-movil` (donde WAHA ya apunta). |
| Validacion error1 | Dominio n8n `2031510-n8n…` vs `n8n-general…` (donde apunta WAHA): confirmar cuál es el vigente. |
| Registrar entrante Chatwoot1 | Texto: `body.data.message.conversation` → `body.payload.body`. |
| Es ubicacion? | Ubicación: `locationMessage` / `degreesLatitude/Longitude` → `payload.location` (`latitude/longitude`). Validar con payload real. |
| Normalizar ubicacion | Teléfono: `body.data.key.remoteJid` → `body.payload.from`.<br>Ubicación: `locationMessage` / `degreesLatitude/Longitude` → `payload.location` (`latitude/longitude`). Validar con payload real. |
| Error ubicacion | Envío: URL `/message/sendText/{session}` → `/api/sendText`; header `apikey` → `X-Api-Key`; body `number` → `chatId` (`…@c.us`) + `session`. |
| Variables ubicacion | Variables: `url_evolution`/`api_evolution` → URL y clave de WAHA; sesión `AGENTE_PRUEBA_1` (ya coincide). |
| ¿Habeas válido? | Texto: `body.data.message.conversation` → `body.payload.body`. |
| ¿Aceptó habeas? | Texto: `body.data.message.conversation` → `body.payload.body`. |

Volver a [Documentación de n8n](<../Documentacion de n8n.md>)
