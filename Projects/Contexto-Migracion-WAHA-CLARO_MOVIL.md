# Contexto: qué me mandaron a hacer (migración Evolution → WAHA, CLARO_MOVIL)

Esta es la conversación original, tal cual, que define la tarea. Es la fuente de verdad de lo que toca. Notas derivadas: [Migración WAHA: CLARO_MOVIL](Migracion-WAHA-CLARO_MOVIL.md) y [CLARO_MOVIL: flujos n8n](CLARO_MOVIL-Flujos-n8n.md).

## Conversación (chat del equipo)
> **Mónica:** Victor Manuel Gómez Rojas te acabo de compartir el archivo de inventario de líneas de WhatsApp.
>
> sería q arranques con la migración de
> `3014048151 · CLARO_MOVIL · ICC · No verificada · En migración Waha · Sergio Moreno (desarrollo) · https://botconnect.interactivo.com.co:8446/manager - carpeta CLARO_MOVIL · AGENTE_PRUEBA_1 · AGENTE_PRUEBA_1 · Inbox 13 - CLARO_MOVIL · https://chatwoot-general.interactivo.com.co/`
> la q se encuentra en el renglón 25.
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
- **Tarea:** migrar de Evolution a WAHA el flujo de la línea **CLARO_MOVIL (3014048151)**, fila 25 del Excel de inventario. Me la asigna [Mónica](../People/Monica.md).
- **Quién sincronizó la línea en WAHA:** [Sergio](../People/Sergio.md). Sesión/agente: `AGENTE_PRUEBA_1`.
- **Webhook de WAHA:** `…/webhook/recepcion-whatsapp-claro-movil` en `n8n-general`.
- **Accesos:** están en el Excel de inventario (no se copian a la bóveda).
- Fila 25 del Excel: línea de tipo ICC, estado "No verificada / En migración Waha", Chatwoot Inbox 13.

## Reglas de trabajo para esta tarea
- **Los flujos de `n8n-json/` son solo para documentar. No se modifican ni se generan versiones nuevas.** Si hay que cambiar algo en n8n, lo hace Vic.
- Mi rol aquí: estudiar, documentar y ayudar con dudas. Nada de propuestas no pedidas.
- Persistencia y código analítico: no aplican (todo queda en n8n; usan Chatwoot, no la API de WhatsApp).
- Sin credenciales reales en las notas.

Volver a [Home](../Home.md)
