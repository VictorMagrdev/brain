# Cargue Medicarte: diseño propuesto de persistencia

> **Superado por el [pitch](../Pitches/Pitch-Persistencia-Cargue-Medicarte.md)** (2026-10-20): el diseño final es **una sola tabla `wp_2031430222`**, una fila por registro enviado, con el contacto ya transformado en `script_fields` (JSON) y **solo cuando uContact aceptó el archivo** (no se guardan errores). Los nombres `pipe_outbound_medicarte*`, las dos tablas y los 11 campos comunes que aparecen abajo ya no aplican. Esta nota queda como historial del razonamiento.

Borrador de diseño (2026-10-19), **sin implementar**. Es insumo para el pitch del [parche de persistencia](../Projects/Parche-Persistencia-Cargue-Medicarte.md). Los patrones que imita están en [Cargues masivos: inventario y patrones](Cargues-Masivos-Inventario-y-Patrones.md).

## Punto de partida

Hoy `CitasService` y `AplicacionMedicamentosService` (back 2, carpeta `cargueMedicarte`) hacen: extraer el Excel, validar estructura y registros, mapear, armar el CSV y subirlo a uContact. **No insertan nada.** Solo emiten un único evento JSON al final (`emitirEvento`, en `cargue-respuestas.util.ts`):

`[INFO] <ISO> - {"evento":"cargue-medicarte","loteId":"medicarte-<tipo>-<timestamp>","tipo":...,"usuario":...,"archivo":...,"totalFilas":...,"cargables":...,"invalidos":...,"resultado":...,"outcome":...,"duration_ms":...}`

`resultado` toma uno de cuatro valores: `exito`, `error`, `rechazado-validacion`, `base-no-procesable`.

Datos que existen dentro del flujo y hoy se pierden al terminar el request:
- **Éxito:** el CSV completo (`buildCsv(pipeline.cargables)`).
- **Rechazo por validación:** `pipeline.detalleErrores`, una lista de `ProcessErrorDto` con `fila`, `numero`, `nombre`, `etapa`, `motivo`.
- **Base no procesable:** solo `pipeline.fatalError`.
- **Falla al subir a uContact:** el CSV y `carga.motivo`.

La validación es estricta: cualquier registro inválido rechaza toda la base. En la práctica o hay CSV o hay errores.

## Decisión de forma: dos tablas (lote y registros)

Se discutieron una tabla única, una fila por cargue con el CSV completo y la opción de dos tablas. Se eligió **dos tablas** porque es lo que hacen Unicef y Clusters, los dos módulos más completos, y evita repetir los datos del lote en cada fila. Una tabla única es el patrón más común (Back 1, WFM, Conexión, Coordinator), pero no tiene cabecera: un cargue que falla antes de leer filas no dejaría rastro.

## Nombres (pendiente de confirmar)

La convención más cercana es Unicef (`bot_outbound_unicef` y `bot_outbound_unicef_lote`). Medicarte no es un bot de WhatsApp, así que no uso el prefijo `bot_outbound_`.

- `cargue_medicarte_lote`: cabecera, una fila por cargue.
- `cargue_medicarte`: registros, una fila por fila del Excel.

## Tabla `cargue_medicarte_lote`

| Columna | Contenido |
|---|---|
| `id` | PK autoincrement. Es el `lote_id` de los registros, como en Unicef (`insertId`) |
| `codigo_lote` | UNIQUE. El `loteId` que ya existe en logs y uContact |
| `tipo` | `citas` o `aplicacion-medicamentos` |
| `nombre_archivo` | Excel original |
| `total_registros` | (se llama así en Unicef) |
| `cargables`, `invalidos` | contadores |
| `resultado` | los cuatro valores de arriba |
| `mensaje` | texto devuelto al usuario (`outcome`) |
| `etapa_fallo`, `motivo_fallo` | NULL salvo en fallos (incluye el motivo de uContact) |
| `duracion_ms` | de `duration_ms` |
| `fecha_creacion`, `usuario_creacion` | como `bot_outbound_unicef_lote` |

## Tabla `cargue_medicarte`

| Columna | Contenido |
|---|---|
| `id` | PK autoincrement |
| `lote_id` | FK a `cargue_medicarte_lote.id`, con índice |
| `fila` | fila del Excel (1-based) |
| `numero_telefono`, `nombre` | para consultar sin abrir el JSON, como Unicef |
| `estado` | `CARGADO`, `NO_CARGADO`, `ERROR` o `ERROR_CARGA` (mayúscula y una palabra, como `PENDIENTE`, `ENVIADO`, `ERROR` en Unicef) |
| `etapa`, `motivo` | NULL salvo en errores (de `ProcessErrorDto`) |
| `ori_fields` | JSON (como texto) con la fila original del Excel |
| `script_fields` | JSON (como texto) con la fila ya mapeada para uContact, más metadatos de usuario y fecha como en Colsubsidio |
| `fecha_creacion`, `fecha_modificacion`, `usuario_modificacion` | patrón `bot_outbound*` |

El CSV subido se reconstruye ordenando por `fila` y tomando `script_fields`. No se guarda aparte.

## Qué se inserta en cada desenlace

| Desenlace (`resultado`) | Lote | Registros |
|---|---|---|
| `exito` | sí | todas las filas, `CARGADO` |
| `rechazado-validacion` | sí | las inválidas como `ERROR` y las válidas como `NO_CARGADO` |
| `error` (falla uContact) | sí, con `motivo_fallo` | todas, `ERROR_CARGA` |
| `base-no-procesable` | sí, con `motivo_fallo` | ninguna |

Con la tabla de lote, la falla total no necesita una fila de registro falsa: basta la cabecera.

## Cómo se implementaría en el back (SQL a mano, sin ORM)

- `cargueMedicarte/data/cargue-medicarte.data.ts`: clase `CargueMedicarteData` con `Db2Service`. Método `guardarCargue(...)`:
  - `createConnection()` y `beginTransaction()`.
  - `SELECT DATE_FORMAT(<SQL_NOW_COLOMBIA>, ...) AS ahora`, una sola vez, igual que Unicef.
  - `INSERT INTO cargue_medicarte_lote (...) VALUES (?, ...)`, tomando `insertId`.
  - `INSERT INTO cargue_medicarte (...) VALUES ?` con las filas, por tandas de 1000 (bucle con `<`, no `<=`).
  - `commit`. En el `catch`: guarda de `ECONNRESET`, `rollback` y `throw`. En el `finally`: `connection.end()`.
  - Devuelve `{ loteId, guardados }`.
- Constantes (estados, `SQL_NOW_COLOMBIA`, tamaño de tanda) en `cargueMedicarte/config/`, junto a `medicarte.config.ts`. Se duplican en vez de importar de Unicef porque el módulo es autocontenido (tiene su propio logger).
- Interfaz `CargueMedicarteRegistroInsert`, con `ori_fields` y `script_fields` ya como string.
- Helper compartido en `services/shared/` que arma lote y filas desde el pipeline y el evento. Lo usan los dos servicios.
- Registrar `CargueMedicarteData` en `providers` de `cargue-medicarte.module.ts`. `CommonModule` ya exporta `Db2Service`.
- Llamarlo en el `finally` de `procesar()`, después de `emitirEvento`.
- Quitar el comentario "No hay persistencia" de ambos servicios.

**Diferencia deliberada con los demás módulos:** el guardado va en su propio `try/catch` y solo hace `logger.error` si falla. En Unicef el insert es la operación principal. Aquí es registro y no debe tumbar el cargue ni la subida a uContact.

El log JSON no cambia, para que la query de DuckDB siga funcionando.

## Cómo se leería después

```sql
-- usos por día y resultado
SELECT DATE(fecha_creacion) dia, tipo, resultado, COUNT(*)
FROM cargue_medicarte_lote GROUP BY dia, tipo, resultado;

-- errores de un lote
SELECT r.fila, r.numero_telefono, r.etapa, r.motivo
FROM cargue_medicarte r
JOIN cargue_medicarte_lote l ON l.id = r.lote_id
WHERE l.codigo_lote = ? AND r.estado = 'ERROR';
```

Estas queries no se ejecutaron.

## Mientras no exista la tabla: contar usos desde los logs (DuckDB)

Cada cargue emite una línea con el JSON del evento, así que se puede filtrar por `"evento":"cargue-medicarte"`. Query propuesta (no ejecutada, hay que ajustar la ruta del log):

```sql
WITH raw AS (
  SELECT line
  FROM read_csv('logs/*.log', columns={'line':'VARCHAR'}, delim='\x01', header=false, quote='')
  WHERE line LIKE '%"evento":"cargue-medicarte"%'
),
ev AS (
  SELECT
    CAST(regexp_extract(line, '^\[INFO\] (\S+) -', 1) AS TIMESTAMP) AS ts,
    json(regexp_extract(line, '(\{.*\})\s*$', 1)) AS j
  FROM raw
)
SELECT date_trunc('day', ts) AS dia, j->>'tipo' AS tipo, j->>'resultado' AS resultado,
       count(*) AS cargues, sum((j->>'totalFilas')::INT) AS filas
FROM ev GROUP BY ALL ORDER BY dia, tipo;
```

- Los fallos por excepción salen con `[ERROR]` y no son wide events. Pero un cargue con excepción igual emite su wide event con `resultado:"error"`, porque `emitirEvento` está en el `finally`.
- Si pm2 o docker agregan su propio prefijo de fecha, hay que ajustar la regex de `ts`.

## Pendientes antes de implementar

1. Confirmar que el pipeline conserve la fila original del Excel para las filas con error (`CitasSourceRow`, `extract` y `validate-records`). No se ha abierto.
2. Ver la forma de `MappedCitas` y su equivalente de medicamentos, para `script_fields`.
3. Versión de MySQL y `max_allowed_packet` (no hay acceso a la BD para verlos). Define el tamaño máximo por `INSERT`, y la versión define si se puede usar `JSON_TABLE` al leer.
4. Datos de pacientes en texto plano (teléfonos, nombres, todo `ori_fields`): definir acceso y retención.

## Decisiones abiertas

- Nombres `cargue_medicarte` y `cargue_medicarte_lote`, o prefijo `bot_outbound_`.
- ¿Se guardan las filas válidas de un lote rechazado como `NO_CARGADO`? Recomendado: sí.
- Retención de datos.

El `CREATE TABLE` de ambas tablas lo ejecuta Víctor, no Claude.

Volver a [Home](../Home.md)
