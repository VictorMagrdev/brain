# Análisis lógico de los flujos CLARO_MOVIL (tablas de verdad y álgebra booleana)

Revisión de las condiciones (IF, Switch, regex, filtros SQL) de los 5 flujos como si fueran circuitos lógicos. Fuente: los JSON de `n8n-json/`, solo lectura. Parte de [Documentación de n8n](Documentacion%20de%20n8n.md). Lo marcado **(verificar)** es una deducción que depende de datos que no tengo (Chatwoot, índices de MySQL).

## 1. Tablas de verdad que se pudieron calcular

### 1.1 Filtro anti-eco (Principal → "Es texto entrante?")
Expresión actual: `fromMe !== true && !fromMe`. Con `f = fromMe`: `(f ≠ true) ∧ ¬f`.

| fromMe | `f ≠ true` | `¬f` | Resultado |
|---|---|---|---|
| true | F | F | F |
| false / undefined / null / 0 | V | V | V |
| 1 | V | F | F |

Como `¬f ⇒ (f ≠ true)`, el primer término es absorbido: `(f ≠ true) ∧ ¬f ≡ ¬f`. **Se simplifica a `!fromMe`.** Sin cambio de comportamiento (probado con los 7 valores).

### 1.2 Respuestas SI / NO (regex `(^|\s)(si|sí|s|no|n)(\s|$|\.)` )
Probado con 34 entradas. Resumen de lo que importa:

| Entrada | ¿Dijo SI? (paso 1) | ¿SI o NO? (paso 3) | Habeas (paso 5) |
|---|---|---|---|
| `si`, `Sí`, `SI`, `s`, `si.` | acepta | válido | **acepta** |
| `no`, `NO`, `n` | inválido | válido | rechaza |
| **`si,`  `Sí, acepto`  `sí, claro`** | **inválido** | **inválido** | **inválido** |
| **`si!`  `¡si!`  `sii`  `sip`** | **inválido** | **inválido** | **inválido** |
| `ok`, `dale`, `👍`, `1` | inválido | inválido | inválido |
| **`no si`  `si no`  `no, si quiero`** | acepta | válido | **acepta** |
| `quiero s`, `claro que si` | acepta | válido | acepta |
| `no gracias`, `no sé` | inválido | válido | rechaza |

Lectura booleana: el patrón busca la palabra suelta en **cualquier** posición y solo admite como cierre espacio, fin de texto o punto. De ahí salen dos fallas, una por exceso y una por defecto (ver hallazgos H1 y H2).

### 1.3 Relay: filtro de mensajes del asesor ("If2")
Cinco condiciones en AND: `outgoing ∧ private≠true ∧ sender=user ∧ source≠bot ∧ event=message_created`.
2⁵ = 32 combinaciones, **solo 1 pasa** (todas verdaderas). Es un AND puro: se puede escribir como un solo IF (ya lo es) y, junto con "inbox = 13", como un solo IF de 6 condiciones.

Condición completa para que el relay envíe un mensaje al cliente:
`event=message_created ∧ outgoing ∧ ¬private ∧ sender=user ∧ source_id≠'bot' ∧ inbox=13 ∧ paso=-1`

### 1.4 Reutilizar conversación ("¿Reutilizar conversación?")
`reutiliza = (paso ≠ '1000') ∧ ∃ conversación: (inbox = 13 ∧ status ∈ {open, pending})`

| paso | ¿Hay conv. open/pending en inbox 13? | reutiliza |
|---|---|---|
| 1000 | sí | **no** (crea nueva) |
| 1000 | no | no |
| otro / '' | sí | sí |
| otro / '' | no | no (crea nueva) |

Es consistente y completa.

### 1.5 Máquina de estados del Principal
Entradas posibles después de pasar el filtro de eco y el de ubicación: `válida` o `inválida` para el paso actual.

| paso | válida | inválida |
|---|---|---|
| 0 / 1000 | bienvenida → 1 (cualquier texto es "válido") | — |
| 1 | menú → 2 | Opción inválida (error +1) |
| 2 | opción 1 → 1000; 2 y 3 → 5; 4 → 4; 5 → 1000 | Opción inválida |
| 3 | handoff migración → -1 (SI **y** NO van al mismo lugar) | Opción inválida |
| 4 | handoff planes → -1 | Opción inválida |
| 5 | SI → 3; NO → 1000 | Opción inválida |
| -1 | — | cae en "Validacion error1" |
| cualquier otro | — | cae en "Validacion error1" |

Cobertura: cada par (paso × entrada) tiene salida definida, **sin huecos**. Ramas muertas o sin efecto en 2.3.

### 1.6 Inactividad: umbrales y condición de búsqueda
`paso ∈ {1..5} ∧ num_advertencia < 3 ∧ id_conversacion no vacío ∧ minutos ≥ (num_advertencia+1)×1`

| num_advertencia | umbral (min sin actividad) | acción |
|---|---|---|
| 0 | ≥ 1 | recordatorio 1 + menú del paso |
| 1 | ≥ 2 | recordatorio 2 + menú del paso |
| 2 | ≥ 3 | aviso de cierre, `paso = 1000`, Chatwoot resolved |
| 3 | — | no entra (`< 3`) |

Como cada recordatorio hace `fecha_modificacion = NOW()`, los minutos se cuentan **desde el último recordatorio**: los tres avisos salen a los 1, 3 y 6 minutos (1 + 2 + 3), no a los 1, 2 y 3.

### 1.7 Opción inválida: contador de errores
`num_errores' = num_errores + 1`; `cierra = (num_errores' ≥ 3)`.

| errores previos | num_errores' | acción |
|---|---|---|
| 0 | 1 | aviso + menú del paso |
| 1 | 2 | aviso + menú del paso |
| ≥ 2 | ≥ 3 | cierre, resolved, `paso = 1000` |

`Principal → Guardar estado` reinicia `num_errores = 0` en cada respuesta válida, así que el contador mide errores **consecutivos**. Correcto.

### 1.8 Cambio de estado
`actúa = (status = 'resolved') ∧ (inbox_id = 13)`. La Switch con una sola regla equivale a un IF. Tabla de 4 combinaciones: solo `V∧V` actualiza.

## 2. Hallazgos

### 2.1 Errores (lógicos)
| # | Sev. | Dónde | Problema | Prueba |
|---|---|---|---|---|
| H1 | Alta | Principal: ¿Dijo SI?, ¿SI o NO?, ¿Habeas válido?, ¿Aceptó habeas? | El regex solo cierra con espacio, fin o `.`. **"Sí, acepto", "si!", "sip" se rechazan** como inválidas; tres veces seguidas cierra la conversación (con 3 errores, §1.7). Un cliente que acepta el habeas con "Sí, acepto" es expulsado. | Tabla §1.2 |
| H2 | Alta | Principal: ¿Aceptó habeas? | Acepta por **cualquier aparición** de la palabra suelta "si"/"s": "no, si quiero" y "no si" cuentan como ACEPTA. El habeas es un consentimiento legal; la ambigüedad debe ir a inválido o a confirmación. | Tabla §1.2 |
| H3 | Alta | Principal: validadores | Leen solo `data.message.conversation`. Una respuesta que sea **cita de un mensaje o contenga un link** llega como `extendedTextMessage.text`, queda en `''` y se toma como inválida. `Registrar entrante Chatwoot1` sí contempla `extendedTextMessage`, los validadores no. | Revisión de expresiones |
| H4 | Alta | Principal: tras `Es ubicacion?` = falso | Cualquier mensaje que no sea texto (imagen, audio, sticker, reacción, mensaje borrado/protocolo) pasa al menú: se registra en Chatwoot como "[Ubicación enviada]" y cuenta como respuesta inválida. **Tres imágenes cierran la conversación.** No hay rama para "no es texto". | Rutas del grafo |
| H5 | Alta | Inactividad: Actualizar advertencia | Hace `SET paso = {{paso_nuevo}}`, donde `paso_nuevo` es el paso leído al inicio del ciclo. Si el cliente responde entre el SELECT y el UPDATE, el cron **pisa el paso nuevo con el viejo** (lost update). Debe tocar `paso` solo al cerrar. | Lectura del flujo |
| H6 | Media | Principal: Menos cargado + Etiqueta asesor **(verificar)** | La carga se mide con `labels[]=asignado-asesor`, pero el único nodo que etiqueta usa `claro-terminales` / `claro-migracion`, y `POST …/labels` en Chatwoot reemplaza las etiquetas. Si nada más pone `asignado-asesor`, **la carga siempre es 0 y todos los traspasos van al primer agente online**. También solo mira la página 1. | Nodos Conversaciones totales / Etiqueta asesor |
| H7 | Media | Principal: handoff sin agentes | Cuando no hay agentes online, "Menos cargado" lanza error **después** de haberle dicho al cliente "un asesor te contactará" y de dejar `paso = -1`. El cliente queda sin asignar ni etiqueta y el bot ya no le responde. | Orden: Guardar estado → Enviar → ¿Handoff? |
| H8 | Media | Todos los flujos | Los nodos de envío tienen `onError = continueRegularOutput`: si WhatsApp (o WAHA) falla con 4xx/5xx, **la ejecución sale en verde** y el estado ya avanzó. Esto esconde errores de la migración. | Ajustes de nodos |
| H9 | Media | Principal: cierre/reapertura **(verificar)** | Para `paso = 1000`, `Registrar entrante Chatwoot1` escribe el mensaje en la conversación **vieja** (`id_conversacion` guardado) y luego otra vez en la nueva: duplica el primer mensaje y puede reabrir la conversación resuelta en Chatwoot. Para clientes nuevos (`id_conversacion = ''`) intenta `/conversations//messages` y falla (el error queda tapado por H8). | Orden del grafo |
| H10 | Media | Cambio de estado: UPDATE | Filtra por cliente, no por conversación. Resolver una conversación vieja (o una duplicada por H9) **cierra el paso de la conversación activa**. Falta `AND id_conversacion = …`. | Query |
| H11 | Media | MySQL **(verificar)** | `INSERT … ON DUPLICATE KEY UPDATE` solo funciona con índice único en (`cliente`,`tipo`). Los `ORDER BY fecha_modificacion DESC LIMIT 1` por todo el flujo sugieren que **puede haber varias filas por cliente**; entonces cada INSERT crea una fila nueva y el cron de Inactividad recorrería filas viejas. Comprobar con `SHOW INDEX FROM claro_movil;`. | Queries |
| H12 | Baja | Opción inválida: "Activar inactividad" | Llama a `/webhook/inactividad-claro-movil`, pero el flujo Inactividad **no tiene webhook** (solo cron). Da 404 que `continueRegularOutput` oculta. | Nodos de ambos flujos |
| H13 | Baja | Principal: Validacion error1 | Manda las variables a cambio-estado, que solo actúa si `body.status = 'resolved'`; Principal no envía `status`. **Llamada sin efecto** (rama del paso -1 y de pasos desconocidos). | Condiciones |
| H14 | Baja | Principal: etiquetas de ramas sin handoff | `Opción 4`, `Habeas 2-3`, `Opción 2-3`, `Rechazo habeas` definen `etiqueta` y se guarda en la BD, pero solo la rama con handoff la aplica en Chatwoot. Etiquetas `claro-portabilidad` y las de ofertas nunca aparecen en Chatwoot. | Rutas del grafo |
| H15 | Baja | Inactividad: umbrales | `(num_advertencia+1)×1` minuto parece un valor de prueba: el cliente recibe "¿Sigues con nosotros?" tras **1 minuto** de silencio. | Query |
| H16 | Baja | Relay | El texto del asesor pasa por `.replace(/\n/g," ")` y se eliminan comillas: **pierde saltos de línea** y comillas. Además ignora adjuntos (el nombre del flujo lo indica: "sin imágenes"). | Nodo Enviar Texto WhatsApp |

### 2.2 Guardas frágiles
- **Anti-bucle del relay.** `source_id ≠ 'bot'` solo lo ponen `Enviar Chatwoot` y el relay; las 11 demás publicaciones del bot en Chatwoot (bienvenida, avisos, menús, recordatorios, cierre) **no lo llevan**. Hoy no hay bucle solo porque en esos momentos `paso ≠ -1`. Es una dependencia de tiempos, no una garantía.
- **Respuesta tardía de Chatwoot:** el relay lee `paso` cuando llega el webhook; si el bot cambió el estado en medio, la decisión cambia. Poner `source_id: 'bot'` en todo lo que publica el bot elimina la dependencia.

### 2.3 Ramas muertas o redundantes
| Nodo | Por qué |
|---|---|
| `Filtro Inbox ` (Principal) | Compara `inbox_id` de `Variables del menú` (constante `'13'`) con `'13'`: **tautología**, siempre verdadero. Además, ningún mensaje llega con inbox distinto porque ya está fijo. |
| `Validar opción (1-5)` + `Switch Opción` | El Switch ya tiene salida "otro → Llamar opción inválida". La validación previa duplica la regla; un solo Switch con fallback cubre ambos. |
| `¿Habeas válido?` + `¿Aceptó habeas?` | Son dos IF sobre dos regex. Un Switch de 3 salidas (SI / NO / otro) lo expresa en un solo nodo y evita H2. |
| `Switch Opción` salidas 2 y 3 | Ambas van a `Habeas 2-3`: se fusionan con la condición `valor ∈ {2,3}`. |
| `¿SI o NO?` (paso 3) | **SI y NO van al mismo destino** (Migración handoff). La respuesta no se usa ni se guarda. Es un nodo de paso que puede ser una sola salida; o, si la respuesta importa, guardarla (ver 3.4). |
| `Es texto entrante?` | Se simplifica a `!fromMe` (§1.1). |
| Switch de Cambio de estado | Una regla = un IF; unible con "inbox" en un solo IF con AND (§1.8). |
| `Variables ubicacion` / `Variables del menú` / Relay `Variables` / Inactividad `Variables` | Cuatro copias de las mismas constantes (cuenta, inbox, URLs, claves, sesión). |

## 3. Optimizaciones
1. **Un solo nodo de variables compartido** (sub-workflow o variables de entorno de n8n / credenciales) en lugar de 4+ nodos Set con las claves escritas. Menos mantenimiento y menos riesgo en la migración: cambiar la URL/clave de WAHA en un sitio, no en seis.
2. **Textos de los menús en una sola fuente.** Hoy el mismo texto vive en Principal, en Opción inválida (5 pares) y en Inactividad (`menus`), con diferencias de formato. Una tabla o un Code único evita que se desincronicen.
3. **Opción inválida: de 23 a unos 10 nodos.** Los 5 pares `Menú N Chatwoot` + `Menú N WhatsApp` (10 nodos) se reducen a un Code que elige el texto por `estado_actual` y **un** par de envío. Lo mismo con Cierre/Aviso. Contador atómico: `UPDATE … SET num_errores = num_errores + 1` evita la carrera de leer, sumar y escribir.
4. **Responder rápido y procesar después.** El webhook usa respuesta inmediata (`onReceived`), bien. Falta serializar mensajes del mismo cliente (dos mensajes seguidos corren dos ejecuciones en paralelo sobre la misma fila).
5. **Guardar lo que se pregunta.** La aceptación del habeas y la respuesta "¿recibe llamadas?" solo quedan como texto en Chatwoot; guardar en `ori_fields` (JSON ya existente) con fecha da trazabilidad y evidencia legal.
6. **Consultas parametrizadas.** Los `INSERT/UPDATE/SELECT` interpolan teléfono y textos; `Obtener estado` del Principal ni siquiera pone comillas (`cliente = 57300…`, comparación numérica contra un varchar). Usar parámetros del nodo MySQL.
7. **Paginación y carga de asesores.** Contar con el total de la bandeja (`meta.assigned_count`) o recorrer páginas; o repartir round-robin.
8. **Errores visibles.** Cambiar `continueRegularOutput` por una salida de error que registre y avise, al menos en los `sendText`, para que una falla de WAHA se vea.
9. **Fusionar filtros baratos del Relay** (If2 + inbox) en un IF y dejar el SELECT para después, como ya está.

## 4. Qué cambia para la migración a WAHA
Estos hallazgos **no se arreglan con la migración**, pero la afectan:
- **H8** puede ocultar errores nuevos de WAHA (404 por `chatId`, 401 por `X-Api-Key`): mira Executions nodo por nodo, no el color de la ejecución.
- **H1/H3/H4** cambian de forma: el texto llegará en `payload.body` (sea cita, link o texto simple), lo que **resuelve H3 sin querer**; los mensajes sin texto vendrán con `body` vacío y `hasMedia`, así que **H4 sigue** y conviene filtrar por `hasMedia`/tipo.
- **H11** se vuelve urgente si WAHA entrega el teléfono como `@lid`: guardaría una clave distinta y crearía filas nuevas por cliente.

Volver a [Documentación de n8n](Documentacion%20de%20n8n.md) · [Guía de pruebas](Guia%20de%20pruebas%20CLARO_MOVIL.md) · [Home](../Home.md)
