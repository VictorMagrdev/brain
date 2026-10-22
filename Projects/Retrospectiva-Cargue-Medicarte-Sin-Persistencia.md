# Retrospectiva: por qué el Cargue Medicarte salió sin persistencia

Escrita el 2026-10-19 como parte del [parche de persistencia](Parche-Persistencia-Cargue-Medicarte.md). Es una retrospectiva sin culpables: el objetivo es entender qué falló en el proceso, no justificar la omisión. Por eso separo lo que está verificado de lo que es una explicación mía.

## Qué pasó

El módulo salió sin guardar el cargue en ninguna tabla. Al preguntar si se podía validar el envío de los mensajes de Medicarte, solo había logs, y el archivo de logs únicamente tenía el día de hoy, no la semana anterior.

## Hechos verificados en el código

1. Los dos servicios (`CitasService` y `AplicacionMedicamentosService`) dicen en un comentario: *"No hay persistencia: el resultado queda en los logs."* Es decir, el diseño del módulo asumió que los logs eran el registro. No lo traté como un descuido aislado: está escrito en el código.
2. El módulo está bien instrumentado para logs: emite **un evento JSON por cargue** con lote, tipo, usuario, archivo, contadores, resultado y duración. Pensado para contar usos desde los logs.
3. Ese diseño depende de que los logs se conserven. El archivo disponible solo tenía lo de hoy, así que el supuesto no se cumplió.
4. Los demás cargues sí persisten, pero por razones que no aplican igual a Medicarte (ver abajo).

## Factores que contribuyeron

Algunos son míos, otros del contexto. Los marco como **verificado** o **interpretación** para no contarme una historia más bonita de la real.

| Factor | Tipo | Nota |
|---|---|---|
| Creí que, al ser un pipeline intermediario (recibe Excel, valida y reenvía a uContact), no hacía falta guardar nada | Interpretación mía | Es una suposición razonable, pero nadie la validó con [Mónica](../People/Monica.md) ni con el equipo |
| El código de Back 1 es confuso: los cargues están escondidos en un archivo de más de 19.000 líneas con nombres que no dicen "cargue" (`insertarBotOutbound` es Colsubsidio) | Verificado | Hizo difícil ver que los otros cargues sí guardan. Ver [inventario](../Wiki/Cargues-Masivos-Inventario-y-Patrones.md) |
| En los otros módulos, quien escribe el `INSERT` lo hace como parte del propio flujo (Back 1 inserta y luego encola; Unicef inserta el lote y luego actualiza `estado`). En Medicarte el flujo termina en un POST a uContact, sin un paso de BD que copiar | Verificado | La tabla en los demás no es solo registro: es el punto de partida del envío o el estado del mismo |
| El patrón de los demás **no** incluye guardar errores ni cabecera de archivo (salvo Unicef y Clusters). No había un ejemplo de "trazabilidad completa" que imitar | Verificado | Ver la sección de errores del [inventario](../Wiki/Cargues-Masivos-Inventario-y-Patrones.md) |
| El flujo de WhatsApp de Medicarte en uContact lo montó Sergio directamente, sin código analítico y sin crear tablas ("use las de uContact directamente"); el módulo de Víctor se conecta a eso | Dicho en el chat (2026-10-20) | Nadie definió una tabla propia ni un código analítico desde el inicio, así que el módulo nació sin dónde guardar |
| No encontré en el repo ni en la bóveda un requisito escrito de trazabilidad o de retención de logs para este módulo | Verificado (en lo que revisé) | No revisé el ticket original ni lo que pidió Mónica por chat |
| Nadie definió cuánto tiempo se conservan los logs del back 2 en el servidor | No verificado | Hay que confirmarlo con [Checho](../People/Checho.md) |

## Lo que no puedo afirmar

- No sé quién escribió el comentario "No hay persistencia" ni cuándo. No corrí `git blame`. Si fue una decisión consciente de alguien del equipo, conviene preguntarle a [Sergio](../People/Sergio.md) por qué.
- No sé si en el pedido original se pidió trazabilidad y se omitió, o si nunca se pidió.
- No es honesto decir "fue por Back 1 confuso" como causa única. Eso explica por qué no vi los ejemplos de persistencia, pero no por qué no se pensó en respaldar.

## Causa raíz (mi lectura)

No existía una definición explícita de **qué debe dejar registrado un cargue masivo**. Cada módulo decidió por su cuenta (Back 1 guarda filas, Unicef agrega lote, Clusters agrega cabecera, Medicarte decidió dejarlo en logs), y nada en el proceso obligaba a preguntarse: *¿cómo voy a saber después qué se cargó y qué pasó?* La pregunta del equipo solo apareció al liberar.

## Qué hacer para que no se repita

- [ ] Agregar al cierre de todo desarrollo con carga de datos una pregunta fija: **¿cómo se responde, una semana después, "qué se cargó, quién lo hizo y qué pasó con cada fila"?**
- [ ] Si la respuesta es "logs", escribir cuánto se conservan y quién los puede ver. Si no se sabe, no se cierra.
- [ ] Incluir "persistencia y trazabilidad" como sección fija en el pitch (en Rabbit holes o No-gos, explícitamente), no dejarla implícita. **Hecho 2026-10-20:** es la sección 0 de la [plantilla](../Templates/Plantilla-Pitch.md) y una regla fija en el [contexto](../Contexto-para-Claude.md), junto con preguntar por el código analítico.
- [ ] Documentar los patrones de cargue en la [wiki](../Wiki/Cargues-Masivos-Inventario-y-Patrones.md) para que el próximo módulo tenga un ejemplo claro.

## Si me preguntan por qué pasó

Una respuesta honesta y corta: *"Se diseñó asumiendo que los logs bastaban como registro y no se validó cuánto tiempo se conservan ni si el equipo necesitaba consultar el histórico. Lo corregimos con persistencia en BD y dejamos una pregunta fija en el cierre de desarrollos para no repetirlo."* No incluye excusas que no pueda sostener.

Volver a [Home](../Home.md)
