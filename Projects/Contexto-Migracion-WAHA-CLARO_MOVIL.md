# Contexto: qué me mandaron a hacer (migración Evolution → WAHA, CLARO_MOVIL)

Esta es la conversación original, tal cual, que define la tarea. Es la fuente de verdad de lo que toca. Notas derivadas: [Migración WAHA: CLARO_MOVIL](Migracion-WAHA-CLARO_MOVIL.md) y [CLARO_MOVIL: flujos n8n](CLARO_MOVIL-Flujos-n8n.md).

## Fila 25 del inventario (con sus encabezados)
Columnas del Excel de inventario de líneas de WhatsApp y el valor de la fila 25:

| Columna | Valor |
|---|---|
| Línea WhatsApp | 3014048151 |
| Campaña | CLARO_MOVIL |
| Propietario | ICC |
| Tipo (Verificada / No verificada) | No verificada |
| Estado | En migración Waha |
| Responsable | Sergio Moreno (desarrollo) |
| Fecha inicio Producción | (vacío) |
| Fecha bloqueo | (vacío) |
| Servidor | `https://botconnect.interactivo.com.co:8446/manager` - carpeta CLARO_MOVIL (el manager de Evolution) |
| Agente evolution | AGENTE_PRUEBA_1 |
| Agente waha | AGENTE_PRUEBA_1 |
| Chatwoot | Inbox 13 - CLARO_MOVIL |
| URL Chatwoot | `https://chatwoot-general.interactivo.com.co/` |
| Etiqueta | (vacío) |
| Carga base / Acción form | (vacío) |
| Observación | (vacío) |

## Conversación (chat del equipo)
> **Mónica:** Victor Manuel Gómez Rojas te acabo de compartir el archivo de inventario de líneas de WhatsApp.
>
> sería q arranques con la migración de la línea 3014048151 (CLARO_MOVIL), la q se encuentra en el renglón 25.
>
> ahí en el excel encuentras dónde están los accesos.
>
> **Mónica → Sergio:** Sergio Moreno ahí no sé si ya Vic puede arrancar solo? o te necesita?
>
> **Sergio:** Buenos días, dame un momento sincronizo esa línea que la tengo yo.
>
> **Mónica:** oki
>
> **Sergio:** Ya quedó sincronizado, el agente es AGENTE_PRUEBA_1.
>
> **Sergio → Vic:** el webhook quedó apuntando a `https://n8n-general.interactivo.com.co/webhook/recepcion-whatsapp-claro-movil`

## Qué toca (resumen)
- **Tarea:** migrar de Evolution a WAHA el flujo de la línea **CLARO_MOVIL (3014048151)**, fila 25 del Excel. Me la asigna [Mónica](../People/Monica.md).
- **Agente:** `AGENTE_PRUEBA_1` es a la vez el **agente Evolution** y el **agente WAHA** de esa fila; no hay que buscar otro nombre. Sergio ya lo sincronizó en WAHA.
- **Webhook de WAHA:** `…/webhook/recepcion-whatsapp-claro-movil` en `n8n-general`.
- **Accesos:** están en el Excel de inventario (no se copian a la bóveda).
- **Estado de la línea:** "En migración Waha", tipo "No verificada", propietario ICC, responsable Sergio Moreno.

## Reglas de trabajo para esta tarea
- **Los flujos de `n8n-json/` son solo para documentar. No se modifican ni se generan versiones nuevas.** Si hay que cambiar algo en n8n, lo hace Vic.
- Mi rol aquí: estudiar, documentar y ayudar con dudas. Nada de propuestas no pedidas.
- Persistencia y código analítico: no aplican (todo queda en n8n; usan Chatwoot, no la API de WhatsApp).
- Sin credenciales reales en las notas.

Volver a [Home](../Home.md)
