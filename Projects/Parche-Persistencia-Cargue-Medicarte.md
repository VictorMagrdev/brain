# Parche: persistencia del Cargue Medicarte

**Qué es:** el módulo Cargue Medicarte (back 2, carpeta `cargueMedicarte`; front `cargueMedicarte.vue`) sube bases de citas y de aplicación de medicamentos a uContact, pero **no guarda nada en la base de datos**. Hay que agregar un registro de cada cargue.
**Estado:** en planificación (2026-10-19). Falta el pitch (Shape Up). No se ha escrito código.
**Origen:** una pregunta del equipo en el chat (ver abajo). Quién pide el parche no quedó escrito en el hilo; confirmar con [Mónica](../People/Monica.md).

## Qué pasó

Se liberó el módulo y el equipo preguntó:

> "Todos tenemos cómo validar si se han enviado los mensajes de WhatsApp de Medicarte desde lo que les liberamos?"

La respuesta honesta hoy es que **no**. El único rastro de cada cargue es una línea JSON en el log del back 2, y el archivo de logs que se procesó **solo contiene el día de hoy**, no la semana pasada. Es decir, no se puede reconstruir qué se cargó antes. Detalle del origen en la [retrospectiva](Retrospectiva-Cargue-Medicarte-Sin-Persistencia.md).

## Cómo se definió el nombre de la tabla (conversación 2026-10-20)

Mónica preguntó si el WhatsApp de Medicarte tenía código analítico. [Sergio](../People/Sergio.md) respondió que lo trabajó directamente en el flujo de uContact, sin código analítico y sin crear tablas. La campaña que se carga es `WhatsAppMedicarteMasivo`. Sergio validó el código para Medicarte confirmación de cita: **2031430222**. Como las cargas que pasan a un bot llevan prefijo `bot_` y esto no es un bot, Mónica propuso `wp_`: la tabla es **`wp_2031430222`**. Víctor dijo que procesaría los logs para sacar cuántas veces se usó el cargue. Detalle en [Código analítico y nombres de tablas](../Wiki/Codigo-Analitico-y-Nombres-de-Tablas.md).

## Contexto del mismo hilo de trabajo (migración de línea de WhatsApp)

Mismo hilo de chat, otro tema, para no perderlo:

- Me compartieron el archivo de inventario de líneas de WhatsApp (el hilo no dice quién) y pidieron arrancar la migración de la línea del **renglón 25**; en el Excel están los accesos.
- [Sergio](../People/Sergio.md) sincronizó esa línea. El agente quedó como `AGENTE_PRUEBA_1`.
- El webhook quedó apuntando a `https://n8n-general.interactivo.com.co/webhook/recepcion-whatsapp-claro-movil`.
- **Bloqueo:** no hay entrada a n8n por falta de permisos. Se le pidió a Mónica gestionarlo. La carpeta en n8n existe. Los demás accesos sí los tengo.
- Pendiente: avisar cuando den acceso a n8n.

Referencia general en la [wiki de WhatsApp](../Wiki/Wiki-WhatsApp-Automatizacion.md) y [Migración de Evolution a WAHA](../Wiki/Migracion-Evolution-a-WAHA.md).

## Qué se averiguó (resumen)

- Back 2 **no usa ORM**: SQL a mano con `mysql2` y `Db2Service`. Las tablas se crean a mano.
- Ninguno de los otros cargues masivos guarda errores de validación. Todos devuelven los inválidos en la respuesta y los descartan.
- Los que persisten guardan **una fila de tabla por fila del Excel**. Nadie guarda el CSV completo en una columna.
- Unicef y Clusters son los únicos con tabla de cabecera (lote).
- Detalle completo: [Cargues masivos: inventario y patrones](../Wiki/Cargues-Masivos-Inventario-y-Patrones.md).

## Diseño y pitch

**Una sola tabla, `wp_2031430222`**, una fila por registro enviado, con el contacto ya transformado en la columna JSON `script_fields`. Se guarda **solo cuando uContact aceptó el archivo**: no se guardan errores ni rechazos, porque ningún otro cargue lo hace. El guardado se espera y nunca altera la respuesta. Detalle completo, con el DDL, los archivos, el impacto y el plan de pruebas, en el [pitch](../Pitches/Pitch-Persistencia-Cargue-Medicarte.md). La [nota de diseño](../Wiki/Cargue-Medicarte-Diseno-Persistencia.md) queda como historial (versión de dos tablas).

**Rama de trabajo:** `feature/confirmaciones-medicarte` (back 2), ya actualizada con `production`.

## Siguiente paso

- [x] Escribir el pitch en [Pitches](../Pitches/Pitch-Persistencia-Cargue-Medicarte.md) con la [Plantilla de Pitch](../Templates/Plantilla-Pitch.md). Está en revisión; **no implementar hasta que Víctor lo diga**.
- [ ] Resolver los pendientes del diseño: fila original en errores, forma de la fila mapeada, `max_allowed_packet`, versión de MySQL, retención y acceso a datos de pacientes.
- [ ] Confirmar los nombres de las tablas y qué se guarda en un lote rechazado.
- [ ] Revisar el pitch con [Sergio](../People/Sergio.md) (contexto histórico del módulo y de por qué no persiste).
- [ ] Aprobación de alcance de [Mónica](../People/Monica.md) antes de programar.
- [ ] Mientras tanto, si hay que responder cuántos cargues se hicieron, contarlos desde los logs con DuckDB (query en el diseño). Los logs que se tengan son los únicos datos disponibles, y [Checho](../People/Checho.md) puede ayudar a ver cuánto histórico conserva el servidor.

## Cómo se sube

Con el [Flujo Git](Flujo-Git.md): feature, integration, develop, test, produccion. Al terminar: funcionando en test, evidencia en la tarea, evidencia en Sonar y tareas liberadas.

Volver a [Home](../Home.md)
