# Código analítico y nombres de tablas

Nota de referencia (actualizada 2026-10-20 con la conversación del equipo). Nació del [parche de persistencia del Cargue Medicarte](../Projects/Parche-Persistencia-Cargue-Medicarte.md), donde el nombre de la tabla se pidió tarde.

## Regla de trabajo
**Al iniciar o planificar un requerimiento, preguntar siempre por el código analítico del servicio y por la persistencia** (ver sección 0 de la [plantilla de pitch](../Templates/Plantilla-Pitch.md)). La pregunta va antes de proponer solución, porque el nombre de la tabla depende de ese código.

## Confirmado (conversación del equipo, 2026-10-20)

**Cómo se llegó al nombre de la tabla de Medicarte:**
1. [Mónica](../People/Monica.md) preguntó si "lo de Medicarte de WhatsApp" tenía código analítico. [Sergio](../People/Sergio.md) respondió que lo había trabajado **directamente en el flujo de uContact**, sin código analítico y **sin crear ninguna tabla** (usó las de uContact).
2. La campaña que se carga es **`WhatsAppMedicarteMasivo`**, solo esa.
3. Mónica recordó que Sebas había indicado que **WhatsApp no genera código analítico**, así que al código del formulario se le adiciona lo de WhatsApp. Sergio revisó y dio el código: **para Medicarte confirmación de cita es `2031430222`**.
4. Convención dicha en el chat: a las cargas que pasaban al **bot** se les agregaba el prefijo **`bot_`**. Mónica: *este no es un bot*, así que se le puede poner **`wp_`** más el código.
5. Resultado: tabla **`wp_2031430222`**.

**Reglas que quedan de ahí:**
- El nombre de una tabla de cargue = prefijo según el tipo de canal + código analítico del servicio. Prefijo `bot_` si la carga pasa a un bot; `wp_` si es WhatsApp que no es un bot.
- Si el servicio no tiene código analítico propio, se pide a Sergio que valide el del formulario o campaña.
- Víctor dijo que procesaría los logs y sacaría cuántas veces se usó el cargue.

**Ejemplos que ya existían en Back 1:** `bot_outbound_20314`, `bot_outbound_2031509`, `bot_outbound_A30670101`, y otras tablas con código (`bsc_2032001`, `bsc_2131511-1_ch`). Unicef usa `bot_outbound_unicef`, sin código.

## Qué sigue sin saberse
El número `2031430222` lo dio Sergio directamente. **No se confirmó de qué tabla o pantalla sale** (de dónde se construye). Hipótesis que se evaluaron al buscarlo en el código, todas sin confirmar como origen del número:

| Hipótesis | Evidencia |
|---|---|
| `ceco.id_ceco` (centro de costo) | tabla `ceco`; en la pantalla de Códigos de Servicio el desplegable "servicio" se llena con `ceco.id_ceco` |
| Código compuesto de `codigo_servicio` / `service_code` | la app lo llama "código analítico" (`actualizarCodigoAnalitico`); es país + provincia + vertical + cliente + servicio + subservicio + campaña |
| `project.id` | Unicef usa `UNICEF_PROJECT_ID = 2031415` como `project_id` de sus skills |

Víctor aclaró que el código **no es el permiso ni la skill, ni está ligado a una campaña**.

## Cómo averiguarlo si hace falta (solo lectura)
```sql
SELECT id, id_ceco, descrip_ceco FROM ceco WHERE id_ceco IN ('2031430222','20314','2031509','2031415','2032001');
SELECT id, name, client_id FROM project WHERE name LIKE '%MEDICARTE%';
```
La respuesta definitiva la da Sergio.

## Dónde mirar en el código
- Back 1: `models/service.js` (`getAllCodes`, `getAllServicio`, `actualizarCodigoAnalitico`) y `routes/serviceRoutes.js`.
- Back 2: `src/servicecodes/data/servicecodes.data.ts` (tabla `service_code`).
- Front: `src/views/pages/codigoServicio.vue` y `editor/codigoServicio/DataViewSidebar.vue`.

Volver a [Home](../Home.md)
