# Datos recibidos, equivalencias WAHA y dónde tocar

Tres cosas en un solo documento: (1) qué datos me dieron y en qué parte de los flujos se usan, (2) qué hay hoy en los flujos de CLARO_MOVIL y cuál es su equivalente en WAHA, sacado del flujo ya migrado [TODO_CLARO Principal V 3.0](TODO_CLARO/Documentacion-TODO-CLARO-Principal.md), y (3) en qué flujo y nodo está cada cosa, para saber qué hacer. Los flujos se cambian en n8n por Vic; aquí solo se documenta. Contexto: [Contexto de la tarea](../Projects/Contexto-Migracion-WAHA-CLARO_MOVIL.md) · [Comparación](Comparacion%20TODO_CLARO%20vs%20CLARO_MOVIL.md) · [Índice](Documentacion%20de%20n8n.md).

Abreviaturas de flujo: **01** Principal V3 · **02** Relay Chatwoot · **03** Opción inválida · **04** Inactividad · **05** Cambio de estado. Los nombres de nodo son los del JSON (búscalos en el canvas del flujo, carpeta [CLARO_MOVIL](CLARO_MOVIL/)).

## 1. Lo que me dieron y para qué sirve en los flujos
| Dato recibido | Valor | Dónde se usa / para qué |
|---|---|---|
| Línea WhatsApp | 3014048151 | No aparece escrita en ningún nodo. Es el número de la sesión de WAHA; sirve para saber a qué número escribir en las pruebas |
| Campaña | CLARO_MOVIL | `tipo = "CLARO_MOVIL"` en los nodos Set (01 Variables del menú / Variables globales / Variables ubicacion, 02 Variables, 04 Variables) y en la columna `tipo` de `claro_movil`. Sufijo de los paths (`…-claro-movil`) |
| Propietario | ICC | Solo informativo (a quién pertenece la línea). No entra a los flujos |
| Tipo | No verificada | Solo informativo del inventario. No entra a los flujos |
| Estado | En migración Waha | Solo informativo; indica que esta es la línea a migrar |
| Responsable | Sergio Moreno (desarrollo) | Con quién coordinar el corte. No entra a los flujos |
| Servidor | `botconnect.interactivo.com.co:8446/manager`, carpeta CLARO_MOVIL | Manager de la instancia Evolution actual: aquí se desconecta/respalda la instancia al hacer el corte. Ojo: los flujos llaman a `evolution-general.interactivo.com.co`, otro host |
| Agente Evolution | AGENTE_PRUEBA_1 | `session` de Evolution: 01 (Variables del menú, Variables ubicacion, Variables globales), 02 (Variables), 04 (Variables). También se guarda en `claro_movil.session_wpp` |
| Agente WAHA | AGENTE_PRUEBA_1 | Pasa a ser `session_wp` dentro de `Variables del menú` (nombre que usa el modelo). Es la sesión por la que WAHA envía |
| Chatwoot | Inbox 13 - CLARO_MOVIL | `inbox_id = 13`: 01 Variables del menú, 02 Variables y su IF `inbox`, 04 Variables, 05 IF `inbox`, 01 `Filtro Inbox` |
| URL Chatwoot | `https://chatwoot-general.interactivo.com.co/` | `url_chatwoot`. Hoy los flujos usan `botconnect…:8443` en 01 (Variables del menú, Variables ubicacion), 02 Variables, 04 Variables; pasa a `chatwoot-general` |
| Webhook de Sergio | `n8n-general…/webhook/recepcion-whatsapp-claro-movil` | Es la **recepción** (WAHA → n8n). No es un nodo del Principal: este flujo recibe de ahí el cuerpo ya normalizado. Hay que ver a qué path reenvía |
| Accesos (Excel) | usuarios y claves | `api_wp` (clave de WAHA) en `Variables del menú`; claves de Chatwoot (`api_chatwoot`) ya existen en los nodos Set. No se copian a la bóveda |
| (del flujo modelo) URL de WAHA | `https://wpp-general.interactivo.com.co` | `url_wp` en `Variables del menú`; también para descargar media |
| (del flujo modelo) Envío | `https://n8n-general.interactivo.com.co/webhook/envio-whatsapp-<campaña>` | Destino de todos los envíos; para CLARO_MOVIL: `envio-whatsapp-claro-movil` |
| (del flujo modelo) Entrada del Principal | `bot-whatsapp-<campaña>` | Path del Webhook entrada del 01: `bot-whatsapp-claro-movil` |

## 2. Equivalencias: CLARO_MOVIL (Evolution) → WAHA (según TODO_CLARO)
Las dos columnas de la derecha salen de los nodos del flujo modelo.

| # | Qué | Hoy en CLARO_MOVIL (Evolution) | Equivalente en WAHA (flujo modelo) | Nodo del modelo donde se ve |
|---|---|---|---|---|
| 1 | Path de entrada | `bot-claro-movil` | `bot-whatsapp-claro-movil` (patrón `bot-whatsapp-todo-claro`) | Webhook entrada |
| 2 | Forma del cuerpo | Payload crudo `body.data…` | Cuerpo normalizado: `remitente_id`, `texto`, `tipo`, `location`, `media` | Webhook entrada → todos los demás |
| 3 | Teléfono | `body.data.key.remoteJid.split('@')[0]` | `body.remitente_id` | Obtener estado del usuario, GET CONTACT, Crea Contacto ChatWoot |
| 4 | Texto del cliente | `body.data.message.conversation` / `extendedTextMessage.text` | `body.texto` | Switch, ¿Responde SI o NO?, ¿Acepta habeas?, Enviar Mensaje Chatwoot Cliente1 |
| 5 | Tipo de mensaje | Se deduce de la forma (`locationMessage`…) | `body.tipo`: `location`, `texto`, `audioMessage`, `media` | Switch2 |
| 6 | Ubicación | `locationMessage` / `liveLocationMessage`; `degreesLatitude/Longitude` | `body.location.latitude/longitude/live` | Switch2 (salida location) → Normalizar ubicacion1 |
| 7 | Imagen | No existe (cuenta como error) | Descarga `body.media.url` cambiando el dominio por `wpp-general` con `X-Api-Key`, y la sube a Chatwoot como adjunto | Switch2 (salida media) → Obtener estado imagen → HTTP Request13 → Enviar Imagen Chatwoot |
| 8 | Audio | Cuenta como error | Salida `Audio` del Switch sin conectar (se ignora) | Switch2 |
| 9 | Mensaje propio (`fromMe`) | `data.key.fromMe` en Es texto entrante? | `If2` sigue leyendo `body.data.key.fromMe` | If2 |
| 10 | Envío a WhatsApp | `POST {url_evolution}/message/sendText/{session}`, header `apikey`, body `{number, text}` | `POST n8n-general/webhook/envio-whatsapp-todo-claro`, body `{variable, tipo_mensaje:"texto", mensaje}` | HTTP Request4, 5, 6, 10, 11, 12 |
| 11 | Variables de WhatsApp | `url_evolution`, `api_evolution`, `session` | `integracion:"WAHA"`, `url_wp`, `api_wp`, `session_wp` | Variables del menú |
| 12 | Variables de Chatwoot | `url_chatwoot` = `botconnect…:8443`, `inbox_id` 13 | `url_chatwoot` = `chatwoot-general…`, `inbox_id` de la campaña (11 en el modelo) | Variables del menú |
| 13 | Datos que viajan al envío | — | `variable` = `Variables del menú.toJsonString()`: `integracion, tipo, payload, estado_actual, id_cuenta, id_conversacion, etiqueta_chatwoot, api_chatwoot, url_chatwoot, api_wp, url_wp, session_wp, inbox_id` | HTTP Request4 |
| 14 | Llamada a Opción inválida | Body armado a mano: `cliente, id_conversacion, id_cuenta, tipo, estado_actual, url_chatwoot, api_chatwoot, url_evolution, api_evolution, session` | Body = `Variables del menú` completo | Validacion error |
| 15 | Llamada a Cambio de estado | Mismo `Variables del menú` | Mismo `Variables del menú` | Validacion error1 |
| 16 | Cliente en los sub-flujos | `body.cliente` | `body.payload.remitente_id` (deducido de lo que envía Validacion error) | — |
| 17 | Sesión guardada en BD | `session_wpp = Variables del menú.session` | `session_wpp` debería salir de `session_wp` (el modelo lo deja mal: lee `session`, que no existe) | Guardar estado menú principal… |
| 18 | Log de entrada | No existe | Insert en `bot_log_v2` con el cuerpo recibido | Log de interacción (opcional) |

## 3. Dónde está cada cosa en CLARO_MOVIL y qué hacer
Cada fila de la sección 2 mapeada a los nodos exactos de los flujos de CLARO_MOVIL.

| # | Flujo | Nodos | Qué hacer |
|---|---|---|---|
| 1 | 01 | Webhook entrada | Cambiar el path a `bot-whatsapp-claro-movil` (y que la recepción reenvíe ahí) |
| 3 | 01 (9 nodos) | Obtener estado, GET CONTACT, Crea Contacto, Crear Conversación, Crear Conversación1, Ctx inicial, Consolidar, Llamar opción inválida, Normalizar ubicacion | `remoteJid.split('@')[0]` → `body.remitente_id` |
| 4 | 01 (9 nodos) | Registrar entrante Chatwoot (inicial), Registrar entrante Chatwoot1, ¿Dijo SI?, ¿SI o NO?, Validar opción (1-5), Switch Opción, Validar plan (1-4), ¿Habeas válido?, ¿Aceptó habeas? | `data.message.conversation` / `extendedTextMessage.text` → `body.texto` |
| 5, 8 | 01 | Es ubicacion? (y los nodos del camino de texto) | Reemplazar por un Switch por `body.tipo`: location / texto / audioMessage / media |
| 6 | 01 (2 nodos) | Es ubicacion?, Normalizar ubicacion | Leer `body.location.latitude/longitude/live` |
| 7 | 01 | (nuevo, a continuación del Switch por tipo) | Agregar la rama media: Obtener estado → GET `body.media.url` con `X-Api-Key` → adjunto en Chatwoot |
| 9 | 01 | Es texto entrante? | Decidir si queda el filtro `fromMe` (en el modelo sigue en formato Evolution) |
| 10 | 01 (3) | Enviar WhatsApp, Bienvenida WhatsApp, Error ubicacion | Cada uno → POST a `envio-whatsapp-claro-movil` con `{variable, tipo_mensaje, mensaje}` |
| 10 | 02 (1) | Enviar Texto WhatsApp | Igual; `variable` se arma en este flujo (no viene del Principal) |
| 10 | 03 (7) | Menú 1 WhatsApp, Menú 2 WhatsApp1, Menú 3 WhatsApp1, Menú 4 WhatsApp1, Menú 5 WhatsApp, Cierre WhatsApp, Aviso WhatsApp | Igual |
| 10 | 04 (1) | Enviar WhatsApp | Igual; la sesión sale de `session_wpp` de la tabla o de `session_wp` |
| 11, 12 | 01 (3) | Variables del menú, Variables ubicacion, Variables globales | Quitar `url_evolution/api_evolution/session`; poner `integracion, url_wp, api_wp, session_wp, payload`, `url_chatwoot` nuevo |
| 11, 12 | 02 (1) | Variables | Igual |
| 11, 12 | 04 (1) | Variables | Igual |
| 14 | 01 | Llamar opción inválida | Body = `Variables del menú` |
| 15 | 01 | Validacion error1 | Body = `Variables del menú` (ya lo hace; solo cambia el contenido de ese nodo) |
| 16 | 03 (20 nodos leen `body.…`) | Obtener estado, ¿Qué menú reenviar?, los 5 Menú N Chatwoot, los 5 Menú N WhatsApp, Actualizar num_errores, Actualizar estado6, Cerrar chat (resolved), Cierre Chatwoot/WhatsApp, Aviso Chatwoot/WhatsApp, Activar inactividad | `body.cliente` → `body.payload.remitente_id`; `body.session` → `body.session_wp`; quitar `url_evolution/api_evolution` |
| 17 | 01 (2) | Guardar estado inicial, Guardar estado | `session_wpp` ← `session_wp` |
| — | 05 | Todo el flujo | Sin cambios de proveedor; recibe `Variables del menú` desde 01 |
| — | 01, 03 | Variables del menú (`url_n8n`), Validacion error1, Llamar opción inválida, Activar inactividad | Mantener o ajustar los dominios de n8n (`2031510-n8n` para sub-flujos, `n8n-general` para envío) |

## 4. Resumen por flujo (orden sugerido de trabajo)
| Orden | Flujo | Cambios |
|---|---|---|
| 1 | 01 Principal V3 | Path, lectura de `remitente_id`/`texto`/`tipo`/`location`, Switch por tipo, rama media, 3 envíos, 3 nodos de variables, llamadas a sub-flujos, `session_wpp` |
| 2 | 03 Opción inválida | Lecturas de `body.…` (20 nodos), 7 envíos |
| 3 | 04 Inactividad | 1 envío, 1 nodo de variables |
| 4 | 02 Relay Chatwoot | 1 envío (armando `variable`), 1 nodo de variables |
| 5 | 05 Cambio de estado | Sin cambios de proveedor |

El 01 va primero porque define el cuerpo que reciben los sub-flujos 03 y 05; el 03 y el 04 comparten el patrón de envío; el 02 es el único que no recibe el cuerpo del Principal, sino un evento de Chatwoot.

## 5. Dónde ver el modelo (flujo TODO_CLARO)
Posición en el editor de n8n (x, y) de los nodos de referencia, y su tarjeta en el [canvas del modelo](TODO_CLARO/TODO-CLARO-Principal.canvas):
| Nodo | Posición en n8n | Para qué mirarlo |
|---|---|---|
| Webhook entrada | (-3744, 112) | Path y forma del cuerpo |
| If2 | (-3536, 112) | Filtro `fromMe` |
| Switch2 | (-3200, 48) | Rama por `tipo` |
| Normalizar ubicacion1 | (-2592, -512) | Lectura de `location` |
| Obtener estado imagen | (-2704, -112) | Inicio de la rama media |
| HTTP Request13 | (-2128, -160) | Descarga de la imagen en WAHA |
| Enviar Imagen Chatwoot | (-1888, -160) | Adjunto en Chatwoot |
| Variables globales | (-2224, 80) | Tipo, sesión y payload |
| Log de interacción | (-1984, 80) | Insert en `bot_log_v2` |
| Obtener estado del usuario | (-1776, 80) | Consulta con `remitente_id` |
| Variables del menú | (-1568, 80) | Variables de WAHA y Chatwoot, `variable` del envío |
| Enrutar por estado1 | (-1152, -16) | Máquina de estados |
| Validacion error1 | (-912, 544) | Llamada a Cambio de estado |
| Validacion error | (256, 416) | Llamada a Opción inválida |
| HTTP Request4 | (2368, -224) | Envío por el webhook (ejemplo; el 5, 6, 10, 11 y 12 son iguales con otro mensaje) |

Los nodos huérfanos del modelo (HTTP Request7/8/9, Habeas WhatsApp, Rechazo WhatsApp, Obtener Base64 Imagen, Variables imagen, Convert to File) son restos de Evolution: no sirven de referencia.

Volver a [Documentación de n8n](Documentacion%20de%20n8n.md) · [Home](../Home.md)
