# REF · TODO_CLARO Principal V 3.0 (ya en WAHA)

Parte de [Documentación de n8n](<../Documentacion de n8n.md>) · [Comparación](<../Comparacion TODO_CLARO vs CLARO_MOVIL.md>) · Canvas: [TODO-CLARO-Principal.canvas](TODO-CLARO-Principal.canvas) · Fuente: `n8n-json/TODO_CLARO Principal V 3.0.json`

## Qué hace
Principal de la campaña TODO_CLARO, ya sobre WAHA. Recibe un mensaje **ya normalizado** (`remitente_id`, `texto`, `tipo`, `location`, `media`) desde un flujo de conexión previo, lo registra en Chatwoot (inbox 11), consulta el paso del cliente en `estado_2031509` y avanza una máquina de estados corta: saludo de oferta → opción 1 (llamada) u opción 2 (WhatsApp con habeas) → asignación de asesor o captura de celular e inserción del lead en la plataforma E2E/CTI. **No llama a WAHA directo**: envía por el webhook `envio-whatsapp-todo-claro`.

| | |
|---|---|
| **Disparador** | Webhook POST `/webhook/bot-whatsapp-todo-claro` (lo llama el flujo de conexión que recibe de WAHA). |
| **Llama a** | `envio-whatsapp-todo-claro` en n8n-general (todos los envíos a WhatsApp)<br>`opcion-invalida-todo-claro` (respuesta no válida)<br>`cambio-estado-conversacion-todo-claro` (paso desconocido)<br>Chatwoot (chatwoot-general), Nominatim y E2E/CTI (token e inserción de lead)<br>WAHA solo para descargar media (`wpp-general`) |
| **Es llamado por** | Flujo de conexión de WAHA (no incluido en las exportaciones) |
| **Nodos** | 74 |

## Nodos
| # | Nodo | Tipo | Qué hace | Lee | Maneja / produce |
|---|---|---|---|---|---|
| 1 | **Variables imagen** | Set (variables) | **Huérfano.** Prepara base64/mimetype de la ruta antigua de Evolution; ningún nodo lo alimenta. | `Obtener estado imagen.id_conversacion`<br>`Obtener Base64 Imagen.base64`<br>`Obtener Base64 Imagen.mimetype`<br>`Obtener Base64 Imagen.caption` | id_conversacion = {{ $('Obtener estado imagen').item.json.id_conversacion }}<br>base64 = {{ $('Obtener Base64 Imagen').item.json.base64 }}<br>mimetype = {{ $('Obtener Base64 Imagen').item.json.mimetype }}<br>caption = {{ $('Obtener Base64 Imagen').item.json.caption ? $('Obtener Base64 Im<br>api_chatwoot = <clave oculta><br>url_chatwoot = https://botconnect.interactivo.com.co:8443 |
| 2 | **Obtener Base64 Imagen** | HTTP Request | **Huérfano.** Pedía el archivo a Evolution (`getBase64FromMediaMessage`); sin conexiones. | `Webhook entrada.body.data.key.id` | POST https://evolution-general.interactivo.com.co/chat/getBase64FromMediaMessage/AGENTE_WHATSAPP_TODO_CLARO_7<br>Body: message, key, id, convertToMp4<br>Headers: apikey |
| 3 | **HTTP Request9** | HTTP Request | **Huérfano.** `sendText` directo a Evolution (menú de categorías). Sin conexiones. | `Variables del menú.url_evolution`<br>`Variables globales.session`<br>`Variables del menú.api_evolution`<br>`Webhook entrada.body.data.key.remoteJid.split` | POST {url_evolution}/message/sendText/{session}<br>Body: number, text<br>Headers: apikey |
| 4 | **HTTP Request8** | HTTP Request | **Huérfano.** `sendText` directo a Evolution (solicitud registrada). Sin conexiones. | `Variables del menú.url_evolution`<br>`Variables globales.session`<br>`Variables del menú.api_evolution`<br>`Webhook entrada.body.data.key.remoteJid.split` | POST {url_evolution}/message/sendText/{session}<br>Body: number, text<br>Headers: apikey |
| 5 | **Habeas WhatsApp** | HTTP Request | **Huérfano.** `sendText` directo a Evolution (habeas). Sin conexiones. | `Variables del menú.url_evolution`<br>`Variables globales.session`<br>`Variables del menú.api_evolution`<br>`Webhook entrada.body.data.key.remoteJid.split` | POST {url_evolution}/message/sendText/{session}<br>Body: number, text<br>Headers: apikey |
| 6 | **Rechazo WhatsApp** | HTTP Request | **Huérfano.** `sendText` directo a Evolution (rechazo). Sin conexiones. | `Variables del menú.url_evolution`<br>`Variables globales.session`<br>`Variables del menú.api_evolution`<br>`Webhook entrada.body.data.key.remoteJid.split` | POST {url_evolution}/message/sendText/{session}<br>Body: number, text<br>Headers: apikey |
| 7 | **HTTP Request7** | HTTP Request | **Huérfano.** `sendText` directo a Evolution (saludo LEO). Sin conexiones. | `Variables del menú.url_evolution`<br>`Variables globales.session`<br>`Variables del menú.api_evolution`<br>`Webhook entrada.body.data.key.remoteJid` | POST {url_evolution}/message/sendText/{session}<br>Body: number, text<br>Headers: apikey |
| 8 | **Webhook entrada** | Webhook | Entrada del bot. Recibe el cuerpo **normalizado** por el flujo de conexión, no el payload crudo de WAHA. | — | POST /webhook/bot-whatsapp-todo-claro |
| 9 | **Convert to File** | convertToFile | **Huérfano.** Convierte base64 a binario; parte de la ruta antigua de imágenes. | `Variables imagen.mimetype` | — |
| 10 | **If2** | IF | Filtro anti-eco: deja pasar solo si `data.key.fromMe` es falso. Todavía lee la ruta del formato Evolution (`body.data.key.fromMe`). | `(entrada).body.data.key.fromMe` | $json.body.data.key.fromMe |
| 11 | **Switch2** | Switch | Clasifica el mensaje por `body.tipo`: location, texto, audioMessage o media. La salida de audio **no está conectada** a nada. | `(entrada).body.tipo` | salida location ⇢ location<br>salida Texto ⇢ texto<br>salida Audio ⇢ audioMessage<br>salida media ⇢ media |
| 12 | **Obtener estado imagen** | MySQL | Busca en `estado_2031509` la conversación del cliente para adjuntar la imagen. | `Webhook entrada.body.remitente_id` | Operación: SELECT sobre `estado_2031509`<br>Lee: * |
| 13 | **Normalizar ubicacion1** | Code (JS) | Toma `body.location` (latitude, longitude, live), arma el link de Google Maps y el texto de la ubicación. | — | Devuelve: esUbicacion, lat, lng, mapsUrl, enVivo, cliente, contenido |
| 14 | **Variables globales** | Set (variables) | Fija `tipo = TODO_CLARO`, una `session` y el cuerpo recibido. Ojo: `session` vale "CLARO_MOVIL" (copia de otro flujo). | `Webhook entrada.body` | tipo = TODO_CLARO<br>session = CLARO_MOVIL<br>payload = {{$('Webhook entrada').item.json.body}} |
| 15 | **Geocodificar** | HTTP Request | Nominatim reverse: obtiene departamento y municipio de lat/lng. Si falla, continúa. | `(entrada).lat`<br>`(entrada).lng` | GET https://nominatim.openstreetmap.org/reverse<br>Headers: User-Agent<br>Query: format, lat, lon, zoom, addressdetails, accept-language |
| 16 | **Log de interacción** | MySQL | Inserta el cuerpo recibido en la tabla `bot_log_v2` (tipo, cuenta, payload, cliente): registro crudo de cada interacción. | `Webhook entrada.body`<br>`(entrada).tipo`<br>`(entrada).payload.remitente_id` | Operación: ? sobre `?` |
| 17 | **HTTP Request13** | HTTP Request | Descarga la imagen desde WAHA: toma `body.media.url`, cambia el dominio de Chatwoot por `wpp-general` y usa `X-Api-Key`. | `Webhook entrada.body.media.url.replace` | GET {replace}<br>Headers: X-Api-Key |
| 18 | **Enviar Imagen Chatwoot** | HTTP Request | Sube la imagen a la conversación de Chatwoot como adjunto (`multipart/form-data`, mensaje entrante). | `Obtener estado imagen.id_conversacion` | POST https://botconnect.interactivo.com.co:8443/api/v1/accounts/1/conversations/{id_conversacion}/messages<br>Headers: api_access_token, Content-Type |
| 19 | **Formato ubicacion** | Code (JS) | Arma el texto final con lugar, dirección aproximada, coordenadas y mapa. | — | Devuelve: municipio, lat, lng, mapsUrl, cliente, enVivo, departamento, contenido |
| 20 | **Obtener estado del usuario** | MySQL | Consulta el último registro del cliente en `estado_2031509` (paso, conversación, etiqueta, cuenta). | `Webhook entrada.body.remitente_id`<br>`Variables globales.tipo` | Operación: SELECT sobre `estado_2031509`<br>Lee: * |
| 21 | **Obtener conversacion ubic1** | MySQL | Busca en `estado_2031509` la conversación del cliente (la más reciente). | `Normalizar ubicacion1.cliente` | Operación: SELECT sobre `estado_2031509`<br>Lee: id_conversacion, id_cuenta |
| 22 | **Variables del menú** | Set (variables) | Hub de configuración. Define `integracion = WAHA`, el estado, cuenta/conversación de la BD, Chatwoot (chatwoot-general), WAHA (`url_wp`, `api_wp`, `session_wp`) e `inbox_id = 11`. Se envía completo al webhook de envío. | `Variables globales.tipo`<br>`Variables globales.payload`<br>`(entrada).paso`<br>`(entrada).id_cuenta`<br>`(entrada).id_conversacion`<br>`(entrada).etiqueta_chatwoot` | integracion = WAHA<br>tipo = {{ $('Variables globales').item.json.tipo }}<br>payload = {{ $('Variables globales').item.json.payload }}<br>estado_actual = {{ $json.paso }}<br>id_cuenta = {{ $json.id_cuenta }}<br>id_conversacion = {{ $json.id_conversacion }} |
| 23 | **Ubicacion a Chatwoot1** | HTTP Request | Publica la ubicación como mensaje entrante en la conversación de Chatwoot. | `Formato ubicacion.contenido`<br>`(entrada).id_conversacion` | POST https://botconnect.interactivo.com.co:8443/api/v1/accounts/1/conversations/{id_conversacion}/messages<br>Body: content, message_type<br>Headers: Content-Type, api_access_token |
| 24 | **Enrutar por estado1** | Switch | Switch de la máquina de estados: undefined/0/1000/1 → alta de contacto y saludo; 2 → opción 1/2; -1 → validar opción 1–5; 3 → solo registrar mensaje; 4 → habeas; 6 → celular; otro → Validacion error1. | `(entrada).estado_actual` | salida undefined ⇢ undefined<br>salida 0 ⇢ 0<br>salida 1000 ⇢ 1000<br>salida 1 ⇢ 1<br>salida 2 ⇢ 2<br>salida -1 ⇢ -1 |
| 25 | **Registrar mensaje asesor** | HTTP Request | Paso 3: solo registra el mensaje del cliente como entrante en Chatwoot (el asesor ya lo atiende). | `Variables del menú.url_chatwoot`<br>`Variables del menú.id_conversacion`<br>`Variables del menú.api_chatwoot`<br>`Webhook entrada.body.texto` | POST {url_chatwoot}/api/v1/accounts/1/conversations/{id_conversacion}/messages<br>Body: content, message_type<br>Headers: api_access_token, Content-Type |
| 26 | **Validacion error1** | HTTP Request | Llama a `cambio-estado-conversacion-todo-claro` enviando todas las variables del menú (estado desconocido). | — | POST https://2031510-n8n.interactivo.com.co/webhook/cambio-estado-conversacion-todo-claro<br>Body:  |
| 27 | **Enviar Mensaje Chatwoot Cliente2** | HTTP Request | Paso -1: registra la respuesta del cliente como entrante en Chatwoot. | `Variables del menú.url_chatwoot`<br>`Variables del menú.id_conversacion`<br>`Variables del menú.api_chatwoot`<br>`Webhook entrada.body.texto` | POST {url_chatwoot}/api/v1/accounts/1/conversations/{id_conversacion}/messages<br>Body: content, message_type<br>Headers: api_access_token, Content-Type |
| 28 | **GET CONTACT** | HTTP Request | Busca el contacto en Chatwoot por `remitente_id`. | `Variables del menú.url_chatwoot`<br>`Variables del menú.api_chatwoot`<br>`(entrada).payload.remitente_id` | GET {url_chatwoot}/api/v1/accounts/1/contacts/search?q={remitente_id}<br>Headers: api_access_token, Content-Type |
| 29 | **Enviar Mensaje Chatwoot Cliente1** | HTTP Request | Paso 2: registra la respuesta del cliente como entrante en Chatwoot. | `Variables del menú.url_chatwoot`<br>`Variables del menú.id_conversacion`<br>`Variables del menú.api_chatwoot`<br>`(entrada).payload.texto` | POST {url_chatwoot}/api/v1/accounts/1/conversations/{id_conversacion}/messages<br>Body: content, message_type<br>Headers: api_access_token, Content-Type |
| 30 | **Registrar respuesta habeas** | HTTP Request | Paso 4: registra la respuesta del cliente como entrante en Chatwoot. | `Variables del menú.url_chatwoot`<br>`Variables del menú.id_conversacion`<br>`Variables del menú.api_chatwoot`<br>`Webhook entrada.body.texto` | POST {url_chatwoot}/api/v1/accounts/1/conversations/{id_conversacion}/messages<br>Body: content, message_type<br>Headers: api_access_token, Content-Type |
| 31 | **Validar opción (1-5)** | IF | Valida que la respuesta sea "1" a "5". Otra cosa → opción inválida. | `Webhook entrada.body.texto` | ['1','2','3','4','5'].includes(( $('Webhook entrada').item.json.body.texto // '').trim()) |
| 32 | **Switch** | Switch | Paso 2: texto "1" → pedir celular; "2" → habeas; otro → opción inválida. | `Webhook entrada.body.texto` | salida 1 ⇢ 1<br>salida 2 ⇢ 2 |
| 33 | **¿Responde SI o NO?** | IF | Valida que la respuesta sea exactamente "1" o "2". Otra cosa → opción inválida. | `Webhook entrada.body.texto` | ['1','2'].includes(( $('Webhook entrada').item.json.body.texto // '').trim()) |
| 34 | **Filtro Inbox 11** | Switch | Comprueba que el inbox sea el 11. Compara una constante con otra, siempre es verdadero. | `Variables del menú.inbox_id` | salida inbox11 ⇢ 11 |
| 35 | **Validacion error** | HTTP Request | Llama a `opcion-invalida-todo-claro` enviando todas las variables del menú. | — | POST https://2031510-n8n.interactivo.com.co/webhook/opcion-invalida-todo-claro<br>Body:  |
| 36 | **Si Contacto Existe** | IF | Si Chatwoot encontró contacto crea la conversación; si no, primero crea el contacto. | `(entrada).meta.count` | $json.meta.count  ==  =0 |
| 37 | **Enviar Mensaje Chatwoot Agente12** | HTTP Request | Publica en Chatwoot "Tu solicitud ha sido registrada" (como mensaje **entrante**). | `Variables del menú.url_chatwoot`<br>`Variables del menú.id_conversacion`<br>`Variables del menú.api_chatwoot` | POST {url_chatwoot}/api/v1/accounts/1/conversations/{id_conversacion}/messages<br>Body: content, message_type<br>Headers: api_access_token, Content-Type,  |
| 38 | **Habeas Chatwoot** | HTTP Request | Publica en Chatwoot el texto de habeas con respuesta numérica (1 autoriza, 2 no). | `Variables del menú.url_chatwoot`<br>`Variables del menú.id_conversacion`<br>`Variables del menú.api_chatwoot` | POST {url_chatwoot}/api/v1/accounts/1/conversations/{id_conversacion}/messages<br>Body: content, message_type<br>Headers: api_access_token, Content-Type |
| 39 | **¿Acepta habeas?** | IF | Si la respuesta es "1" acepta y continúa; "2" rechaza. | `Webhook entrada.body.texto` | ( $('Webhook entrada').item.json.body.texto// '').trim() === '1' |
| 40 | **pedir celular** | HTTP Request | Publica en Chatwoot la petición del celular para la llamada. | `Variables del menú.url_chatwoot`<br>`Variables del menú.id_conversacion`<br>`Variables del menú.api_chatwoot` | POST {url_chatwoot}/api/v1/accounts/1/conversations/{id_conversacion}/messages<br>Body: content, message_type<br>Headers: api_access_token, Content-Type |
| 41 | **Enviar Mensaje Chatwoot Agente13** | HTTP Request | Publica en Chatwoot el menú de categorías (celulares, televisores, portátiles, equipos de sonido, otros). | `Variables del menú.url_chatwoot`<br>`Variables del menú.id_conversacion`<br>`Variables del menú.api_chatwoot` | POST {url_chatwoot}/api/v1/accounts/1/conversations/{id_conversacion}/messages<br>Body: content, message_type<br>Headers: api_access_token, Content-Type,  |
| 42 | **Crear Conversación** | HTTP Request | Crea la conversación para un contacto existente. | `Variables del menú.url_chatwoot`<br>`Variables del menú.api_chatwoot`<br>`Variables del menú.inbox_id`<br>`(entrada).payload[0].id` | POST {url_chatwoot}/api/v1/accounts/1/conversations<br>Body: source_id, inbox_id, contact_id<br>Headers: api_access_token, Content-Type |
| 43 | **Crea Contacto ChatWoot** | HTTP Request | Crea el contacto con `remitente_id` como identificador y `source_id`, atributo `bot = on`. | `Variables del menú.url_chatwoot`<br>`Variables del menú.api_chatwoot`<br>`Webhook entrada.body.remitente_id`<br>`Variables del menú.inbox_id` | POST {url_chatwoot}/api/v1/accounts/1/contacts<br>Body: name, identifier, inbox_id, source_id, custom_attributes, bot<br>Headers: api_access_token, Content-Type |
| 44 | **Rechazo Chatwoot** | HTTP Request | Publica en Chatwoot el mensaje de despedida por no autorizar el tratamiento de datos. | `Variables del menú.url_chatwoot`<br>`Variables del menú.id_conversacion`<br>`Variables del menú.api_chatwoot` | POST {url_chatwoot}/api/v1/accounts/1/conversations/{id_conversacion}/messages<br>Body: content, message_type<br>Headers: api_access_token, Content-Type |
| 45 | **HTTP Request5** | HTTP Request | Envía por WhatsApp el texto de habeas por el webhook de envío. | `Variables del menú.toJsonString` | POST https://n8n-general.interactivo.com.co/webhook/envio-whatsapp-todo-claro<br>Body: variable, tipo_mensaje, mensaje |
| 46 | **HTTP Request11** | HTTP Request | Envía por WhatsApp la confirmación de solicitud registrada. | `Variables del menú.toJsonString` | POST https://n8n-general.interactivo.com.co/webhook/envio-whatsapp-todo-claro<br>Body: variable, tipo_mensaje, mensaje |
| 47 | **HTTP Request12** | HTTP Request | Envía por WhatsApp "Por favor ingrese número de celular para realizar la llamada". | `Variables del menú.toJsonString` | POST https://n8n-general.interactivo.com.co/webhook/envio-whatsapp-todo-claro<br>Body: variable, tipo_mensaje, mensaje |
| 48 | **Wait5** | Wait | Espera 3 s para que Chatwoot termine de crear la conversación. | — | Espera 3 s |
| 49 | **Crear Conversación1** | HTTP Request | Crea la conversación con el contacto recién creado. | `Variables del menú.url_chatwoot`<br>`Variables del menú.api_chatwoot`<br>`Variables del menú.inbox_id`<br>`(entrada).payload.contact.identifier`<br>`(entrada).payload.contact.id` | POST {url_chatwoot}/api/v1/accounts/1/conversations<br>Body: source_id, inbox_id, contact_id<br>Headers: api_access_token, Content-Type |
| 50 | **Guardar estado menú principal2** | MySQL | UPDATE: `telefono` = texto recibido, `paso = 3`, trazabilidad en `ori_fields`. | `Webhook entrada.body.texto`<br>`Variables del menú.id_conversacion`<br>`Variables del menú.session`<br>`Webhook entrada.body.remitente_id`<br>`Variables del menú.tipo`<br>`Variables del menú.inbox_id` | Operación: UPDATE sobre `estado_2031509`<br>Escribe: telefono, paso, fecha_modificacion, id_conversacion, num_advertencia, num_errores, session_wpp, payload, ori_fields |
| 51 | **Guardar estado habeas** | MySQL | UPDATE: `paso = 4` (espera respuesta del habeas). | `Variables del menú.id_conversacion`<br>`Variables del menú.session`<br>`Webhook entrada.body.remitente_id`<br>`Webhook entrada.body.texto`<br>`Variables del menú.tipo`<br>`Variables del menú.inbox_id` | Operación: UPDATE sobre `estado_2031509`<br>Escribe: paso, fecha_modificacion, id_conversacion, num_advertencia, num_errores, session_wpp, payload, ori_fields |
| 52 | **HTTP Request6** | HTTP Request | Envía por WhatsApp el menú de categorías. | `Variables del menú.toJsonString` | POST https://n8n-general.interactivo.com.co/webhook/envio-whatsapp-todo-claro<br>Body: variable, tipo_mensaje, mensaje |
| 53 | **HTTP Request10** | HTTP Request | Envía por WhatsApp el mensaje de rechazo. | `Variables del menú.toJsonString` | POST https://n8n-general.interactivo.com.co/webhook/envio-whatsapp-todo-claro<br>Body: variable, tipo_mensaje, mensaje |
| 54 | **Guardar estado habeas2** | MySQL | UPDATE: `paso = 6` (espera celular) y agrega la respuesta a `ori_fields`. | `Variables del menú.id_conversacion`<br>`Variables del menú.session`<br>`Webhook entrada.body.remitente_id`<br>`Webhook entrada.body.texto`<br>`Variables del menú.tipo`<br>`Variables del menú.inbox_id` | Operación: UPDATE sobre `estado_2031509`<br>Escribe: paso, fecha_modificacion, id_conversacion, num_advertencia, num_errores, session_wpp, payload, ori_fields |
| 55 | **Guardar estado menú principal** | MySQL | UPDATE: `paso = -1` (traspaso a asesor) con trazabilidad en `ori_fields`. | `Variables del menú.id_conversacion`<br>`Variables del menú.session`<br>`Webhook entrada.body.remitente_id`<br>`Webhook entrada.body.texto`<br>`Variables del menú.tipo`<br>`Variables del menú.inbox_id` | Operación: UPDATE sobre `estado_2031509`<br>Escribe: paso, fecha_modificacion, id_conversacion, num_advertencia, num_errores, session_wpp, payload, ori_fields |
| 56 | **HTTP Request2** | HTTP Request | Pide token a la plataforma E2E/CTI (`/E2E/gettoken`). Usuario y clave están escritos en el nodo. | — | POST https://e2ecti.interactivo.com.co/E2E/gettoken |
| 57 | **Enviar Mensaje Chatwoot Cliente** | HTTP Request | Registra el primer mensaje del cliente como entrante en la conversación recién creada. | `Variables del menú.url_chatwoot`<br>`Crear Conversación.id`<br>`Crear Conversación1.id`<br>`Variables del menú.api_chatwoot`<br>`Webhook entrada.body.texto` | POST {url_chatwoot}/api/v1/accounts/1/conversations/{id}/messages<br>Body: content, message_type<br>Headers: api_access_token, Content-Type |
| 58 | **Guardar estado rechazo** | MySQL | UPDATE: `paso = 1000` (conversación cerrada). | `Variables del menú.id_conversacion`<br>`Variables del menú.session`<br>`Webhook entrada.body.remitente_id`<br>`Webhook entrada.body.texto`<br>`Variables del menú.tipo`<br>`Variables del menú.inbox_id` | Operación: UPDATE sobre `estado_2031509`<br>Escribe: paso, fecha_modificacion, id_conversacion, num_advertencia, num_errores, session_wpp, payload, ori_fields |
| 59 | **obtener label** | Set (variables) | Define la etiqueta del traspaso: la original de la BD + `agente_terminales`. | `Variables del menú.etiqueta_chatwoot` | label_original = {{ $('Variables del menú').item.json.etiqueta_chatwoot }}<br>label1 = agente_terminales |
| 60 | **HTTP Request1** | HTTP Request | Inserta el lead (`insertLeadPhone`) en la CTI: teléfono (sin 57), campaña 209, GoContact, fechas con 1 hora menos. | `Variables del menú.id_conversacion`<br>`Webhook entrada.body.texto.replace`<br>`(entrada).token` | POST https://e2ecti.interactivo.com.co//E2E/insertLeadPhone<br>Body: idLeadProvider, provider, firstName, lastName, phone, phone2, phone3, phone4, canal, gender, birthdate, address, age, url, legalText, optin, cpl, c2cDate, leadDate, expirationDate, internationalDP, leadExclusivity, verticality, typeLead, utmCampaign, utmSource, utmMedium, cti, campaignId, providerService, category, priority, agentId, CustomData1, CustomData2, CustomData3, CustomData4, CustomData5, CustomData6, CustomData7, CustomData8, CustomData9, CustomData10, CustomData11, CustomData12, CustomData13, CustomData14, CustomData15, CustomData16, CustomData17, CustomData18, CustomData19, CustomData20<br>Headers: marktel-access-token |
| 61 | **obtener label3** | Set (variables) | Define la etiqueta inicial `claro-terminales`. | — | label = claro-terminales |
| 62 | **Asignar a asesor** | HTTP Request | Aplica las dos etiquetas a la conversación. | `Variables del menú.url_chatwoot`<br>`Variables del menú.id_conversacion`<br>`Variables del menú.api_chatwoot`<br>`(entrada).label_original`<br>`(entrada).label1` | POST {url_chatwoot}/api/v1/accounts/1/conversations/{id_conversacion}/labels<br>Body: labels<br>Headers: Content-Type, api_access_token |
| 63 | **Asignar a asesor3** | HTTP Request | Aplica la etiqueta a la conversación (reemplaza las etiquetas existentes). | `Variables del menú.url_chatwoot`<br>`Crear Conversación.id`<br>`Crear Conversación1.id`<br>`Variables del menú.api_chatwoot`<br>`(entrada).label` | POST {url_chatwoot}/api/v1/accounts/1/conversations/{id}/labels<br>Body: labels<br>Headers: Content-Type, api_access_token |
| 64 | **HTTP Request3** | HTTP Request | Lista los miembros del inbox 11 en Chatwoot. | `Variables del menú.url_chatwoot`<br>`Variables del menú.id_cuenta`<br>`Variables del menú.inbox_id`<br>`Variables del menú.api_chatwoot` | GET {url_chatwoot}/api/v1/accounts/{id_cuenta}/inbox_members/{inbox_id}<br>Headers: Content-Type, api_access_token |
| 65 | **HTTP Request** | HTTP Request | Consulta los miembros del inbox en Chatwoot (resultado no usado después). | `Variables del menú.url_chatwoot`<br>`Variables del menú.inbox_id`<br>`Variables del menú.api_chatwoot` | GET {url_chatwoot}/api/v1/accounts/1/inbox_members/{inbox_id}<br>Headers: Content-Type, api_access_token |
| 66 | **Code** | Code (JS) | Filtra los agentes con `availability_status = online`. | — | Devuelve: agentsFound, availableAgents |
| 67 | **Wait1** | Wait | Espera 4 s antes de responder. | — | Espera 4 s |
| 68 | **Agentes en linea** | IF | Si hay al menos un agente online continúa; si no, el flujo termina sin asignar. | `(entrada).agentsFound` | $json.agentsFound |
| 69 | **Enviar Mensaje Chatwoot Agente** | HTTP Request | Publica en Chatwoot el saludo de la oferta (opción 1 llamada, 2 WhatsApp) como saliente. | `Variables del menú.url_chatwoot`<br>`Crear Conversación.id`<br>`Crear Conversación1.id`<br>`Variables del menú.api_chatwoot` | POST {url_chatwoot}/api/v1/accounts/1/conversations/{id}/messages<br>Body: content, message_type<br>Headers: api_access_token, Content-Type,  |
| 70 | **Obtener conversaciones totales1** | HTTP Request | Trae las conversaciones abiertas asignadas del inbox con etiqueta `asignado-asesor`. | `Variables del menú.url_chatwoot`<br>`Variables del menú.id_cuenta`<br>`Variables del menú.inbox_id`<br>`Variables del menú.api_chatwoot` | GET {url_chatwoot}/api/v1/accounts/{id_cuenta}/conversations?assignee_type=assigned&status=open&inbox_id={inbox_id}&labels[]=asignado-asesor&page=1<br>Headers: Content-Type, api_access_token |
| 71 | **HTTP Request4** | HTTP Request | Envía por WhatsApp el saludo de la oferta usando el webhook de envío (`tipo_mensaje: texto`). | `Variables del menú.toJsonString` | POST https://n8n-general.interactivo.com.co/webhook/envio-whatsapp-todo-claro<br>Body: variable, tipo_mensaje, mensaje |
| 72 | **Code1** | Code (JS) | Elige el agente online con menos conversaciones asignadas. | `Code.availableAgents` | Devuelve: agentId |
| 73 | **Guardar estado menú principal1** | MySQL | INSERT/UPSERT en `estado_2031509`: `paso = 2`, conversación nueva, etiqueta, sesión, payload y `ori_fields` con la primera respuesta. | `Webhook entrada.body.remitente_id`<br>`Variables del menú.tipo`<br>`Variables del menú.inbox_id`<br>`Crear Conversación.id`<br>`Crear Conversación1.id`<br>`Variables globales.session`<br>… | Operación: INSERT sobre `estado_2031509`<br>Escribe: cliente,paso,tipo,id_cuenta,id_inbox,id_conversacion,num_advertencia,num_errores,fecha_modificacion,session_wpp,payload,ori_fields,etiqueta_chatwoot |
| 74 | **Asignar agente1** | HTTP Request | Asigna la conversación al agente elegido. | `Variables del menú.url_chatwoot`<br>`Variables del menú.id_cuenta`<br>`Variables del menú.id_conversacion`<br>`Variables del menú.api_chatwoot`<br>`(entrada).agentId` | POST {url_chatwoot}/api/v1/accounts/{id_cuenta}/conversations/{id_conversacion}/assignments<br>Body: assignee_id<br>Headers: Content-Type, api_access_token |

## Conexiones (de → a)
- Variables imagen → Convert to File
- Obtener estado imagen → HTTP Request13
- Validar opción (1-5) (sí) → Enviar Mensaje Chatwoot Agente12
- Validar opción (1-5) (no) → Validacion error
- Obtener conversacion ubic1 → Ubicacion a Chatwoot1
- Normalizar ubicacion1 → Geocodificar
- Formato ubicacion → Obtener conversacion ubic1
- Geocodificar → Formato ubicacion
- Enviar Mensaje Chatwoot Cliente2 → Validar opción (1-5)
- If2 (sí) → Switch2
- Enviar Mensaje Chatwoot Agente13 → HTTP Request6
- Guardar estado menú principal → obtener label
- HTTP Request3 → Code
- Obtener conversaciones totales1 → Code1
- obtener label → Asignar a asesor
- Asignar a asesor → HTTP Request3
- Code1 → Asignar agente1
- Agentes en linea (sí) → Obtener conversaciones totales1
- Code → Agentes en linea
- Switch (salida 0) → pedir celular
- Switch (salida 1) → Habeas Chatwoot
- Switch (salida 2) → Validacion error
- HTTP Request2 → HTTP Request1
- Wait5 → Enviar Mensaje Chatwoot Cliente
- Crear Conversación1 → Enviar Mensaje Chatwoot Cliente
- Si Contacto Existe (sí) → Crear Conversación
- Si Contacto Existe (no) → Crea Contacto ChatWoot
- GET CONTACT → Filtro Inbox 11
- Crear Conversación → Wait5
- Crea Contacto ChatWoot → Crear Conversación1
- Enviar Mensaje Chatwoot Cliente1 → Switch
- Switch2 (salida 0) → Normalizar ubicacion1
- Switch2 (salida 1) → Variables globales
- Switch2 (salida 3) → Obtener estado imagen
- Enviar Mensaje Chatwoot Cliente → obtener label3
- Asignar a asesor3 → HTTP Request
- obtener label3 → Asignar a asesor3
- Guardar estado menú principal2 → HTTP Request2
- Enviar Mensaje Chatwoot Agente12 → HTTP Request11
- HTTP Request → Wait1
- Habeas Chatwoot → HTTP Request5
- Registrar respuesta habeas → ¿Responde SI o NO?
- ¿Responde SI o NO? (sí) → ¿Acepta habeas?
- ¿Responde SI o NO? (no) → Validacion error
- ¿Acepta habeas? (sí) → Enviar Mensaje Chatwoot Agente13
- ¿Acepta habeas? (no) → Rechazo Chatwoot
- Rechazo Chatwoot → HTTP Request10
- Enrutar por estado1 (salida 0) → GET CONTACT
- Enrutar por estado1 (salida 1) → GET CONTACT
- Enrutar por estado1 (salida 2) → GET CONTACT
- Enrutar por estado1 (salida 3) → GET CONTACT
- Enrutar por estado1 (salida 4) → Enviar Mensaje Chatwoot Cliente1
- Enrutar por estado1 (salida 5) → Enviar Mensaje Chatwoot Cliente2
- Enrutar por estado1 (salida 6) → Registrar mensaje asesor
- Enrutar por estado1 (salida 7) → Registrar respuesta habeas
- Enrutar por estado1 (salida 8) → Enviar Mensaje Chatwoot Agente12
- Enrutar por estado1 (salida 9) → Validacion error1
- Enviar Mensaje Chatwoot Agente → HTTP Request4
- Wait1 → Enviar Mensaje Chatwoot Agente
- Variables del menú → Enrutar por estado1
- Filtro Inbox 11 (salida 0) → Si Contacto Existe
- Obtener estado del usuario → Variables del menú
- Variables globales → Log de interacción
- Log de interacción → Obtener estado del usuario
- Webhook entrada → If2
- HTTP Request4 → Guardar estado menú principal1
- HTTP Request5 → Guardar estado habeas
- HTTP Request6 → Guardar estado menú principal
- HTTP Request10 → Guardar estado rechazo
- HTTP Request11 → Guardar estado menú principal2
- pedir celular → HTTP Request12
- HTTP Request12 → Guardar estado habeas2
- HTTP Request13 → Enviar Imagen Chatwoot

## Observaciones (restos de Evolution, credenciales, detalles)
| Nodo | Observación |
|---|---|
| Registrar mensaje asesor | Credencial escrita en el nodo. |
| Variables imagen | Credencial escrita en el nodo. |
| Obtener Base64 Imagen | Resto de Evolution (llamada directa a Evolution).<br>Lee la ruta del formato Evolution (`body.data.key…`) aunque el cuerpo ya viene normalizado.<br>Credencial escrita en el nodo. |
| Enviar Mensaje Chatwoot Cliente2 | Credencial escrita en el nodo. |
| Guardar estado menú principal | Lee `Variables del menú.session`, que ese nodo no define (define `session_wp`). |
| If2 | Lee la ruta del formato Evolution (`body.data.key…`) aunque el cuerpo ya viene normalizado. |
| HTTP Request9 | Resto de Evolution (llamada directa a Evolution).<br>Lee la ruta del formato Evolution (`body.data.key…`) aunque el cuerpo ya viene normalizado. |
| Enviar Mensaje Chatwoot Agente13 | Credencial escrita en el nodo. |
| HTTP Request3 | Credencial escrita en el nodo. |
| Asignar agente1 | Credencial escrita en el nodo. |
| Obtener conversaciones totales1 | Credencial escrita en el nodo. |
| Asignar a asesor | Credencial escrita en el nodo. |
| HTTP Request2 | Credencial escrita en el nodo. |
| Enviar Mensaje Chatwoot Cliente | Credencial escrita en el nodo. |
| Crear Conversación1 | Credencial escrita en el nodo. |
| GET CONTACT | Credencial escrita en el nodo. |
| Crear Conversación | Credencial escrita en el nodo. |
| Crea Contacto ChatWoot | Credencial escrita en el nodo. |
| Enviar Mensaje Chatwoot Cliente1 | Credencial escrita en el nodo. |
| HTTP Request | Credencial escrita en el nodo. |
| Asignar a asesor3 | Credencial escrita en el nodo. |
| Guardar estado menú principal2 | Lee `Variables del menú.session`, que ese nodo no define (define `session_wp`). |
| HTTP Request8 | Resto de Evolution (llamada directa a Evolution).<br>Lee la ruta del formato Evolution (`body.data.key…`) aunque el cuerpo ya viene normalizado. |
| Enviar Mensaje Chatwoot Agente12 | Credencial escrita en el nodo. |
| Habeas Chatwoot | Credencial escrita en el nodo. |
| Habeas WhatsApp | Resto de Evolution (llamada directa a Evolution).<br>Lee la ruta del formato Evolution (`body.data.key…`) aunque el cuerpo ya viene normalizado. |
| Guardar estado habeas | Lee `Variables del menú.session`, que ese nodo no define (define `session_wp`). |
| Registrar respuesta habeas | Credencial escrita en el nodo. |
| Rechazo Chatwoot | Credencial escrita en el nodo. |
| Rechazo WhatsApp | Resto de Evolution (llamada directa a Evolution).<br>Lee la ruta del formato Evolution (`body.data.key…`) aunque el cuerpo ya viene normalizado. |
| Guardar estado rechazo | Lee `Variables del menú.session`, que ese nodo no define (define `session_wp`). |
| HTTP Request7 | Resto de Evolution (llamada directa a Evolution).<br>Lee la ruta del formato Evolution (`body.data.key…`) aunque el cuerpo ya viene normalizado. |
| Enviar Mensaje Chatwoot Agente | Credencial escrita en el nodo. |
| Variables del menú | Credencial escrita en el nodo. |
| Guardar estado habeas2 | Lee `Variables del menú.session`, que ese nodo no define (define `session_wp`). |
| pedir celular | Credencial escrita en el nodo. |
| HTTP Request13 | Credencial escrita en el nodo. |

Volver a [Documentación de n8n](<../Documentacion de n8n.md>)
