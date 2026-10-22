# Cargues masivos: inventario y patrones (back 1 y back 2)

Hallazgos de revisar el código de los dos backs el 2026-10-19. Sirven de base para el [parche de persistencia de Cargue Medicarte](../Projects/Parche-Persistencia-Cargue-Medicarte.md) y para el [diseño propuesto](Cargue-Medicarte-Diseno-Persistencia.md).

- **Back 1:** `miattend-back` (Node/Express). Todo vive en `models/service.js` (más de 19.000 líneas) y `routes/serviceRoutes.js`.
- **Back 2:** `miattend-back-v2-confirmaciones` (NestJS). Una carpeta por módulo, con `data/`, `services/`, `controllers/`.

## Inventario

### Back 1: un registro por llamada

El front lee el Excel con `xlsx` y llama a la ruta **una vez por fila**. El back inserta esa fila y la mete a una cola en memoria que manda a un webhook de n8n.

| Módulo (front) | Ruta POST | Tabla | Webhook n8n | Espera entre envíos |
|---|---|---|---|---|
| Colsubsidio (`cargueColsubsidio.vue`) | `/bot-outbound/insertar` (con jwt) | `bot_outbound` | `carga-base-colsubsidio` y `carga-base-colsubsidio-bienvenida` | ~1 min por registro (estimado) |
| Claro móvil (`cargueClaro.vue`, `claroOrganico.vue`) | `/bot-outbound/insertarClaro` (sin jwt) | `bot_outbound_20314` | `carga-base-claro-movil` y `carga-base-claro-inbound` | 3 min |
| Claro terminales TYT (`cargueTerminales.vue`) | `/bot-outbound/insertarClaroTYT` (sin jwt) | `bot_outbound_2031509` | `carga-base-claro-tyt` | 5 min |
| Claro orgánico (`A30670101BackV2.vue`) | `/bot-outbound/insertarClaroOrganico` (sin jwt) | `bot_outbound_A30670101` | `confirmacion-pedido-claro-organico` | sin revisar |
| Claro terminales out | `/bot-claro-terminales-out/insertarRegistro` (sin jwt) | `bot_A30670101_out` | `carga-base-claro-terminales-out` | sin revisar |

Columnas de todas las `bot_outbound*`: `ori_fields` (JSON con la fila del Excel), `script_fields` (JSON), `fecha_creacion`, `fecha_modificacion`, `usuario_modificacion`. La fecha sale de `CONVERT_TZ(NOW(), @@session.time_zone, "-05:00")`.

Los GET de lectura sí llevan `jwt`. Tres de las rutas POST no.

### Back 2: archivo completo en el servidor

| Módulo | Tabla(s) | Cómo inserta |
|---|---|---|
| **Cargue Unicef** (WhatsApp) | `bot_outbound_unicef_lote` y `bot_outbound_unicef` | Transacción: valida la skill, cuenta los registros del día con `FOR UPDATE` para aplicar un límite diario, inserta el lote y luego todos los registros con `INSERT ... VALUES ?` |
| **Cargue Medicarte** (citas y medicamentos) | **ninguna** | No persiste. Envía el CSV a uContact y deja un evento JSON en logs (`evento:"cargue-medicarte"`) |
| **Clusters / loads** | `cluster_load`, `cluster_load_record_file`, la tabla de cada formato (`INSERT INTO ${tableName}`) y `ticket_data` | Cabecera más detalle, con `loadId`, `loadFormatId`, `loadTicketId`, `recordPosition` y `loadRecordStateId` |
| **Coordinator** (Excel de asignaciones) | `asignaciones`, `employee_role`, `employee_skill` | Transacción. Arma el `INSERT` concatenando strings (`coordinator.data.ts`, sin parámetros) |
| **Calendar WFM** | `plan_turnos` | Transacción, un solo `INSERT ... VALUES ?`. Las filas con error van a `errors[]` sin tumbar el lote |
| **Calendar Conexión** | `turnos_conexion` | Igual que WFM |

Sin revisar a fondo: `tickets` y `format145`. En Back 1 hay muchos otros `INSERT` de negocio (ausencias, empleados, ventas de Claro) que son formularios de un registro, no cargues.

## Patrones comunes

1. **Dos familias.** Back 1 inserta una fila por request, desde el front fila a fila. Back 2 sube el archivo completo y hace un insert masivo con `VALUES ?` en transacción.
2. **Nombres de tabla.**
   - Back 1: `bot_outbound` más sufijo de proyecto o skill (`_20314`, `_2031509`, `_A30670101`). La de Colsubsidio no tiene sufijo.
   - Unicef: `bot_outbound_unicef` y `bot_outbound_unicef_lote` (sufijo `_lote` para la cabecera).
   - Clusters: tablas dinámicas, el nombre sale de la configuración del formato.
   - WFM y Conexión: una tabla fija por dominio (`plan_turnos`, `turnos_conexion`).
   - Todo snake_case, dominio primero.
3. **Una fila de tabla equivale a una fila del Excel** en todos los que persisten. Nadie guarda el Excel ni el CSV completo en una columna.
4. **El contenido del documento sí queda guardado**, fila por fila: como JSON en `ori_fields` (Back 1 y Unicef), como columnas (Clusters, Coordinator) o columnas más un JSON (`detalle_turno` en WFM, `tiempo_json` en Conexión). El archivo original no se puede reconstruir exacto.
5. **Disparo posterior.** Los de bots mandan el registro a un webhook de n8n tras el insert. En Back 1 con una cola en memoria y pausa fija. En Unicef con un servicio externo.
6. **Sin estado de envío en Back 1.** La tabla no guarda si el webhook respondió bien, solo hay `console.error`. La cola está en memoria: un reinicio del servidor pierde lo pendiente.
7. **Trazabilidad por lote.** Solo Unicef (`bot_outbound_unicef_lote`: archivo, total, usuario) y Clusters (`cluster_load`) guardan una cabecera. En Back 1 no hay forma de saber qué archivo originó cada fila.
8. **Validación.** Unicef, WFM, Conexión y Coordinator validan antes de insertar. Back 1 no valida nada en el back: confía en lo que manda el front, incluido `usuario_modificacion`.
9. **Errores: ningún cargue los persiste.**

## Errores: qué pasa con los inválidos

| Módulo | Qué pasa |
|---|---|
| Colsubsidio, Claro, TYT, orgánico | El front descarta las filas inválidas y solo las notifica en pantalla. El back recibe solo las válidas. Si el `INSERT` falla: `console.error` y `{ Code: false }`. Si el webhook falla: `console.error`, la fila ya quedó insertada |
| Unicef | Los `rechazados` (teléfono mal formado o repetido, indicativo inválido) se devuelven en la respuesta y no se guardan. Lo único que se guarda es `estado` (`ENVIADO` o `ERROR`) del envío a n8n |
| WFM y Conexión | Las filas con error van a `result.errors[]` (fila, cédula, fecha, motivo): se devuelven en la respuesta y se loguean. Las válidas se insertan |
| Coordinator | `errors[]` y `errorCount` en la respuesta |
| Clusters | Solo estados por registro (`loadRecordStateId`), no errores con motivo |
| Medicarte | Los errores van en la respuesta (`detalleErrores`) y en logs |

**Ninguna tabla de cargue tiene columnas de error, etapa o motivo.** Guardar errores en Medicarte no tiene precedente en el código.

## Cosas especiales al insertar (para imitarlas)

1. **Una sola hora para todo el cargue** (Unicef): `SELECT DATE_FORMAT(CONVERT_TZ(NOW(), @@session.time_zone, '-05:00'), '%Y-%m-%d %H:%i:%s') AS ahora` una vez, y ese string va al lote y a todos los registros.
2. **`SQL_NOW_COLOMBIA`** como constante en `unicef-chatbot.constants.ts`: `CONVERT_TZ(NOW(), @@session.time_zone, '-05:00')`.
3. **Interfaz de la fila a insertar** exportada desde el archivo de datos (`UnicefRegistroInsert`), con `ori_fields` y `script_fields` ya como string (`JSON.stringify`).
4. **Esqueleto de transacción:** `Db2Service.createConnection()`, `beginTransaction()`, insert del lote, `insertId`, `INSERT ... VALUES ?`, `commit`. En el `catch`, `rollback` y `throw`. En el `finally`, `connection.end()`.
5. **Tandas de 1000 filas** (Clusters): `fileData.slice(size, size + 1000)` en un bucle dentro de la misma transacción. Ojo: su bucle usa `while (size <= fileData.length)`. Si el total es múltiplo exacto de 1000, la última vuelta manda un array vacío y `VALUES ?` con `[]` falla. Usar `<`.
6. **Guarda de `ECONNRESET`** (Clusters): `if (error.code === 'ECONNRESET') throw error;` antes del `rollback`, porque con la conexión caída el `rollback` también falla y esconde el error original.
7. **Resultado como objeto, sin lanzar** (Unicef): `{ ok, loteId, guardados, ... }`. Los errores de BD sí se relanzan.
8. **Parámetros `?` siempre.** Coordinator concatena strings en el SQL: no imitarlo (riesgo de inyección SQL).
9. **Usuario:** ambos controladores (Unicef y Medicarte) toman `req.user.username` del JWT. Medicarte lo hace en `usuarioDe(req)`, con `'Sistema'` de respaldo.

## Back 2 no usa ORM

El `package.json` de Back 2 solo trae `mysql` y `mysql2`. No hay TypeORM, Sequelize, Prisma ni Knex, y no hay `@Entity`, `@InjectRepository` ni `@InjectModel` en `src`. Todo es SQL a mano con `connection.query(sql, params)` a través de `Db2Service.createConnection()`.

- Las tablas se crean a mano con `CREATE TABLE`. No hay migraciones.
- La forma de la fila es una interfaz de TypeScript y el `INSERT` es un string.
- `CommonModule` exporta `Db2Service`, y el módulo de Medicarte ya importa `CommonModule`.

## Por qué estos cargues se pasan por alto al buscar a mano

- **Back 1 es un archivo gigante.** `service.js` tiene más de 19.000 líneas con unos 80 `INSERT INTO` mezclados. Nada en el nombre de las funciones dice que son cargues: `insertarBotOutbound` es Colsubsidio.
- **Buscar "cargue" o "masivo" en Back 1 da solo tres coincidencias** y ninguna es el cargue real. TYT son cinco líneas bajo el comentario `CARGUE TERMINALES (TYT)`.
- **El Excel se lee en el navegador.** El back solo ve un POST por fila, parece un insert individual.
- **Tablas con el código de proyecto** (`bot_outbound_2031509`) no dicen qué módulo las usa. Hay que cruzar front, ruta y modelo.
- **El `INSERT` dinámico de Clusters** no deja ver en el código a qué tabla escribe.
- **Medicarte tiene el problema opuesto:** está muy visible y bien nombrado, pero no inserta nada.

**Método que funcionó:** ir por el front. Listar las vistas `cargue*.vue`, ver los `$http.post` de cada una, seguir cada ruta hasta su `INSERT`, y buscar en el back los `INSERT INTO bot_` y los `VALUES ?`.

Volver a [Home](../Home.md)
