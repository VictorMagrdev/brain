# Migración WAHA: línea CLARO_MOVIL (3014048151)

Primera línea migrada de Evolution a WAHA, siguiendo la [guía de migración](../Wiki/Migracion-Evolution-a-WAHA.md). Origen: inventario de líneas de WhatsApp (Excel, fila 25), compartido por mi jefa [Mónica](../People/Monica.md).

> Contexto original de la tarea (chat textual): [Contexto migración WAHA](Contexto-Migracion-WAHA-CLARO_MOVIL.md). Los flujos de `n8n-json/` son solo para documentar; no se tocan.

## Datos de la línea
| Campo | Valor |
|---|---|
| Número | 3014048151 |
| Campaña | CLARO_MOVIL |
| Propietario | ICC |
| Tipo | No verificada |
| Estado | En migración Waha |
| Responsable | Sergio Moreno (desarrollo) |
| Servidor (Evolution) | `https://botconnect.interactivo.com.co:8446/manager` (carpeta CLARO_MOVIL) |
| Agente Evolution | AGENTE_PRUEBA_1 |
| Agente WAHA | AGENTE_PRUEBA_1 |
| Chatwoot | Inbox 13 - CLARO_MOVIL, `https://chatwoot-general.interactivo.com.co/` |
| Webhook WAHA | `https://n8n-general.interactivo.com.co/webhook/recepcion-whatsapp-claro-movil` |

Los accesos (usuarios/claves) están en el Excel; **no se copian aquí**.

## Estado
- Sergio sincronizó la línea en WAHA (sesión lista). Agente: AGENTE_PRUEBA_1.
- Webhook ya apunta al flujo n8n `recepcion-whatsapp-claro-movil`.
- Falta: adaptar el flujo n8n del lado Evolution al payload de WAHA.

## Siguientes pasos (según la guía)
0. Verificar que el `session` usado en el flujo sea el correcto.
1. Exportar el workflow actual de Evolution de CLARO_MOVIL (y dejarlo desactivado, sin borrar).
2. Capturar un payload real de WAHA que llegue al webhook nuevo y compararlo con el de Evolution ([mapeo de campos](../Wiki/Migracion-Evolution-a-WAHA.md)).
3. Ajustar: `chatId` con `@c.us`, header `X-Api-Key`, `session` en el body, evento `message`, filtro anti-bucle `fromMe`.
4. Confirmar el flujo hacia Chatwoot (inbox 13) y el agente AGENTE_PRUEBA_1.
5. Probar texto, media, tildes/emojis y reinicio de la sesión; guardar evidencia.

## Respuestas a la regla fija
- **Persistencia:** no aplica; todo queda dentro de n8n. Lo que hay que verificar es que el `session` del flujo sea el correcto (el de WAHA para esta línea).
- **Código analítico:** no aplica. No usan la API de WhatsApp sino Chatwoot, principalmente por webhooks. Ya existen flujos de conexión que este flujo reutiliza, así que esa parte casi no se revisa.

## Flujos estudiados
Documentación completa en [CLARO_MOVIL: flujos n8n](CLARO_MOVIL-Flujos-n8n.md) (5 workflows, 12 nodos `sendText` por cambiar, 5 hallazgos por resolver).

## Pendiente
- Carpeta de JSON: [n8n-json](../n8n-json/README.md).
- JSON del workflow n8n actual de Evolution (Vic lo está buscando).
- Confirmar el motor de la sesión WAHA (NOWEB/GOWS/WEBJS) y que el nombre de `session` coincida.

Relacionado: [Módulo WhatsApp](Modulo-WhatsApp.md) · [WAHA](../Wiki/WAHA.md) · [WAHA webhooks](../Wiki/WAHA-Webhooks.md) · [WAHA + n8n](../Wiki/WAHA-con-n8n.md) · [Evolution API](../Wiki/Evolution-API.md)

Volver a [Home](../Home.md)
