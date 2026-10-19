# Wiki: WhatsApp, WAHA, n8n y webhooks

Índice de la wiki. Investigado el 2026-10-16 con la documentación oficial. Es material de referencia, no tareas.

## Notas
- [Webhooks: conceptos](Webhooks-Conceptos.md): qué son, cómo se asegura y depura uno.
- [WAHA](WAHA.md): qué es, motores, sesiones, instalación y envío de mensajes.
- [WAHA: webhooks y eventos](WAHA-Webhooks.md): configuración, eventos, payload, HMAC y reintentos.
- [WAHA: seguridad y antibloqueo](WAHA-Seguridad-y-Antibloqueo.md): API key, buenas prácticas y límites de WhatsApp.
- [n8n: nodo Webhook](n8n-Webhook.md): URLs test/production, autenticación, respuesta y proxy.
- [WAHA + n8n](WAHA-con-n8n.md): integración paso a paso, patrones y errores típicos.
- [Evolution API](Evolution-API.md): lo que se usaba antes de WAHA (instancias, endpoints, webhooks, variables).
- [Migración Evolution → WAHA](Migracion-Evolution-a-WAHA.md): equivalencias, plan por fases, rollback y checklist.

## Relación con mi trabajo
- El [Módulo WhatsApp](../Projects/Modulo-WhatsApp.md) es un desarrollo distinto (envío de listas vía uContact). Estas notas son conocimiento general para futuras consultas.
- Si de esto sale un desarrollo, va primero un pitch (ver [Plantilla de Pitch](../Templates/Plantilla-Pitch.md)).

## Alertas de la investigación
- WAHA automatiza WhatsApp Web, que Meta no soporta oficialmente. Para uso empresarial serio, la alternativa es la API oficial de WhatsApp Business.
- Fuentes en conflicto sobre qué motores trae la versión gratuita (Core) frente a Plus: verificar la imagen actual antes de decidir.

Volver a [Home](../Home.md)
