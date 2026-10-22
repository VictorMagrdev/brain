# Guía de pruebas: flujos CLARO_MOVIL con WAHA

Cómo hacer funcionar y probar la **copia** de los 5 flujos en n8n mientras los adaptas a WAHA. Parte de [Documentación de n8n](Documentacion%20de%20n8n.md) y del [pitch](../Pitches/Pitch-Migracion-WAHA-CLARO_MOVIL.md). Los nodos a cambiar están en la sección "Intervención para WAHA" de cada documento en `CLARO_MOVIL/`.

## 0. Antes de empezar: qué puede salir mal
La copia comparte con producción **MySQL, Chatwoot (inbox 13), el cron y las rutas de webhook**. Tres cuidados:

| Riesgo | Por qué pasa | Qué hacer |
|---|---|---|
| Choque de rutas de webhook | Dos workflows activos con el mismo path y método no pueden convivir en n8n | Poner sufijo `-waha` a los paths de la copia (paso 2) |
| La copia llama a los flujos **originales** | Principal llama a `/webhook/opcion-invalida-…` y `/webhook/cambio-estado-…`; si el original está activo, responde el original (con Evolution) | Cambiar esas URLs en la copia al path con sufijo |
| Escribir o enviar a clientes reales | Misma tabla `claro_movil`, mismo inbox, y el cron de Inactividad recorre **todos** los clientes | Probar solo con **tu número**; filtrar el cron por tu número (paso 6) |

Regla: **un solo proveedor por línea a la vez.** Mientras pruebas con WAHA, el flujo original sigue atendiendo por Evolution a los demás.

## 1. Datos que necesitas (del Excel de inventario)
- URL de WAHA y su API key (header `X-Api-Key`).
- Nombre de la sesión: `AGENTE_PRUEBA_1`.
- Un número de WhatsApp tuyo para escribirle a la línea, escrito como `57XXXXXXXXXX` (así se guarda en `claro_movil.cliente`).
- Acceso a: n8n (la carpeta con la copia), Chatwoot (inbox 13) y MySQL (`claro_movil`).

## 2. Preparar la copia en n8n
1. Abre la carpeta de la copia y renombra cada workflow con sufijo `(WAHA)`. Déjalos **inactivos** hasta el paso 4.
2. Cambia los paths de los webhooks de la copia para no chocar con los originales:

| Workflow (copia) | Path original | Path en la copia |
|---|---|---|
| Principal V3 | `bot-claro-movil` | `recepcion-whatsapp-claro-movil` (el que Sergio dejó en WAHA; el original usa otro, no choca) |
| Opción inválida | `opcion-invalida-claro-movil` | `opcion-invalida-claro-movil-waha` |
| Cambio de estado | `cambio-estado-conversacion-claro-movil` | `cambio-estado-conversacion-claro-movil-waha` |
| Webhook Chatwoot (relay) | `chatwoot-claro-movil` | `chatwoot-claro-movil-waha` |

3. Actualiza las llamadas internas de la copia a esos paths nuevos:
   - Principal → nodo **Llamar opción inválida**: `/webhook/opcion-invalida-claro-movil-waha`.
   - Principal → nodo **Validacion error1**: `/webhook/cambio-estado-conversacion-claro-movil-waha`.
   - Opción inválida → nodo **Activar inactividad**: el flujo Inactividad no tiene webhook, así que esa llamada no hace nada útil hoy; déjala o desactívala mientras pruebas.
4. Confirma el dominio de n8n: `url_n8n` (nodo **Variables del menú**) apunta a `2031510-n8n…` y WAHA a `n8n-general…`. Usa el dominio de la instancia donde tienes la copia.
5. Credenciales: en los nodos Set que tengan `url_evolution` / `api_evolution`, pon la URL y la clave de WAHA (idealmente renombra a `url_waha` / `api_waha` en todos los nodos que las leen, incluido el body que Principal manda a Opción inválida).

## 3. Verificar WAHA antes de tocar n8n
1. Estado de la sesión: `GET {URL_WAHA}/api/sessions/AGENTE_PRUEBA_1` con header `X-Api-Key`. Debe decir `WORKING`. Si dice `SCAN_QR_CODE` o `STOPPED`, avísale a Sergio.
2. Envío de prueba a tu número (desde Swagger de WAHA o con curl):
```bash
curl -X POST "{URL_WAHA}/api/sendText" \
  -H "X-Api-Key: <API_KEY_WAHA>" -H "Content-Type: application/json" \
  -d '{"session":"AGENTE_PRUEBA_1","chatId":"57XXXXXXXXXX@c.us","text":"prueba"}'
```
   Te debe llegar el mensaje. Este es el formato exacto que usarán los 12 nodos `sendText`.
3. Webhook de la sesión: en `GET /api/sessions/AGENTE_PRUEBA_1` revisa `config.webhooks`: url `…/webhook/recepcion-whatsapp-claro-movil` y que incluya el evento `message`.

## 4. Capturar un payload real (primer paso de verdad)
Es lo que falta para cambiar bien las expresiones.
1. Activa **solo** Principal (WAHA) con el path `recepcion-whatsapp-claro-movil`.
2. Escribe "hola" desde tu número a la línea.
3. En n8n: Executions → abre la ejecución → mira la salida de **Webhook entrada**. Guarda el JSON (sin datos sensibles) en `n8n-json/`.
4. Anota dónde viene: `body.event`, `body.payload.from` (¿`@c.us` o `@lid`?), `body.payload.body`, `body.payload.fromMe`. Repite enviando una **ubicación** para ver `body.payload.location`.

Forma esperada según la documentación de WAHA (valídala contra la tuya):
```json
{ "event": "message", "session": "AGENTE_PRUEBA_1",
  "payload": { "id": "…", "from": "57XXXXXXXXXX@c.us", "fromMe": false,
               "body": "hola", "hasMedia": false } }
```

## 5. Ciclo de trabajo para hacer cambios
1. En la ejecución guardada, usa **"Copy to editor"** / pin data en **Webhook entrada**: así relanzas el flujo con el mismo mensaje sin escribir desde el celular.
2. Cambia un grupo de nodos (primero la recepción en Principal, luego los envíos) y usa **Execute step** nodo por nodo.
3. Revisa cada vez tres lugares: la salida del nodo en n8n, la fila en `claro_movil` y la conversación en Chatwoot.
4. Orden recomendado de cambios: webhook de entrada → nodos que leen el payload → variables → `sendText` de Principal → resto de flujos.
5. Para simular sin celular, manda un POST a la URL de test del webhook con el payload de ejemplo:
```bash
curl -X POST "https://n8n-general.interactivo.com.co/webhook-test/recepcion-whatsapp-claro-movil" \
  -H "Content-Type: application/json" \
  -d '{"event":"message","session":"AGENTE_PRUEBA_1","payload":{"id":"t1","from":"57XXXXXXXXXX@c.us","fromMe":false,"body":"si","hasMedia":false}}'
```
(`/webhook-test/` solo responde mientras el nodo está en "Listen for test event".)

## 6. Probar cada flujo
Limpia primero tu estado para empezar de cero:
```sql
UPDATE claro_movil SET paso='1000' WHERE cliente='57XXXXXXXXXX' AND tipo='CLARO_MOVIL';
SELECT * FROM claro_movil WHERE cliente='57XXXXXXXXXX' AND tipo='CLARO_MOVIL';
```

| # | Prueba | Cómo | Debe pasar | Verifica en |
|---|---|---|---|---|
| 1 | Bienvenida | Escribe "hola" | Llega bienvenida; `paso = 1`; conversación nueva en Chatwoot con el mensaje | WhatsApp, `claro_movil`, Chatwoot |
| 2 | Menú | Responde "SI" | Llega menú de 5 opciones; `paso = 2` | Ídem |
| 3 | Opciones | 1 / 4 / 5 / 2 | 1 deuda y 5 *611 → `paso 1000`; 4 ofertas → `paso 4`; 2 → habeas, `paso 5` | Ídem |
| 4 | Portabilidad | 2 → SI → SI/NO | Pasa por habeas (5) y pregunta de llamadas (3); NO al habeas → rechazo, `1000` | Ídem |
| 5 | Handoff | Desde ofertas elige 1–4; o desde `paso 3` responde SI/NO | Mensaje de asesor, `paso = -1`, etiqueta y agente asignado (requiere un agente **online** en el inbox) | Chatwoot, `claro_movil` |
| 6 | Opción inválida | Responde "xyz" 3 veces en un paso | 1.º y 2.º: aviso + menú del paso (`num_errores` 1, 2); 3.º: cierre, Chatwoot `resolved`, `paso = 1000` | WhatsApp, Chatwoot, `claro_movil` |
| 7 | Ubicación | Envía ubicación | Llega a Chatwoot como mensaje con lugar y mapa; `ori_fields` guarda el evento. Con coordenadas inválidas: aviso por WhatsApp | Chatwoot, `ori_fields` |
| 8 | Asesor → cliente | Con `paso = -1`, escribe como asesor en Chatwoot | Llega a tu WhatsApp (flujo Relay; ver nota) | WhatsApp |
| 9 | Cambio de estado | Marca la conversación como resuelta en Chatwoot | `paso = 1000` (flujo Cambio de estado; ver nota) | `claro_movil` |
| 10 | Inactividad | Ejecuta el flujo a mano (ver abajo) | Recordatorio 1 y 2 con el menú; el 3.º cierra y deja `paso = 1000` | WhatsApp, Chatwoot, `claro_movil` |
| 11 | Mensajes propios y grupos | Envía algo desde la línea; escribe desde un grupo | Se ignoran (no responde ni crea nada) | Executions |

**Notas de las pruebas 8 y 9:** esos flujos los llama Chatwoot, y Chatwoot apunta a los paths **originales**. Para probar la copia, mándale un POST de ejemplo a su path `-waha`, o pídele a quien administre Chatwoot que agregue temporalmente un webhook al path `-waha`. Ojo: si el webhook original sigue activo y el cliente está en `paso = -1`, el flujo original también intentará responder por Evolution.

**Prueba 10, Inactividad (cuidado):** no actives el cron de la copia. Ejecútalo a mano y, **antes**, en el nodo **Buscar inactivos** añade `AND cliente='57XXXXXXXXXX'` a la consulta; si no, enviaría recordatorios a todos los clientes reales que estén en pasos 1–5. Para no esperar el umbral, deja tu fila con `fecha_modificacion` antigua:
```sql
UPDATE claro_movil SET fecha_modificacion = NOW() - INTERVAL 10 MINUTE
WHERE cliente='57XXXXXXXXXX' AND tipo='CLARO_MOVIL';
```
El umbral es `(num_advertencia + 1) × 1` minuto. Quita el filtro de tu número cuando termines (y no lo subas así a producción).

## 7. Si algo falla
| Síntoma | Revisa |
|---|---|
| No llega nada a n8n | `config.webhooks` de la sesión, que el workflow esté activo, path exacto, `event = message` |
| Llega pero no responde | Ejecución en Executions; probablemente `from`/`body` aún leen `body.data…` |
| 401/403 al enviar | Header `X-Api-Key` y la clave de WAHA |
| 422/404 al enviar | Body: `session` y `chatId` con `@c.us`, sesión en `WORKING` |
| Responde dos veces | Evolution y WAHA atendiendo a la vez, o dos workflows activos |
| Crea conversaciones repetidas | `cliente` guardado distinto (por ejemplo `@lid` en vez del número): revisa el valor de `payload.from` |
| Ubicación no llega | Estructura de `payload.location` distinta a la esperada: revisa el payload real |
| `opcion-invalida` responde por Evolution | La copia está llamando al path original: revisa la URL de **Llamar opción inválida** |
| No asigna agente | Error "No hay agentes online en el inbox": pon un agente online |

## 8. Al terminar las pruebas
1. Deja tu número en `paso = 1000` y cierra la conversación de prueba en Chatwoot.
2. Quita filtros de prueba (el de `Buscar inactivos`) y los datos fijos.
3. Guarda la evidencia (capturas de ejecuciones, WhatsApp y Chatwoot) para la tarea y para Sonar si aplica.
4. Corte a producción: lo acuerdas con Sergio y Mónica (un solo proveedor por línea, orden: desactivar originales, activar copias con los paths definitivos). Los originales de Evolution se desactivan, no se borran.

Volver a [Documentación de n8n](Documentacion%20de%20n8n.md) · [Home](../Home.md)
