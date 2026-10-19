# Mónica

**Rol:** jefa.
**Relación:** es quien asigna el trabajo: los tickets de GLPI y los desarrollos.

## Qué asigna
- **Tickets GLPI:** arreglos de usuarios que se resuelven con queries. Ver las plantillas en [Consultas SQL para tickets](../Projects/Consultas-SQL-Tickets.md).
- **Desarrollos:** nuevas funcionalidades o cambios. Cada uno empieza con un [pitch](../Pitches/Pitches.md) usando la [Plantilla de Pitch](../Templates/Plantilla-Pitch.md).

## Qué necesito de ella
- Prioridad cuando hay varios tickets o desarrollos a la vez.
- Aprobación del alcance (appetite y no-gos) antes de programar.
- Confirmación antes de subir cambios a producción con el [flujo Git](../Projects/Flujo-Git.md).

## Reglas que me ha dado
- Todo usuario debe tener siempre skill y rol; si no sé el rol, es **Fichaje**.
- Para supervisores: Fichaje + Coordinador.
- Detalle completo en [Checklist de usuarios GLPI](../Projects/Checklist-Usuarios-GLPI.md).
- Me responde rápido por chat; si algo es raro, confirmar con ella antes de asumir.

## Pendientes con ella
Ninguno. Todo lo conversado hasta ahora está cerrado y se guarda solo como información (ver [Migración de grabaciones](../Projects/Migracion-Grabaciones.md), [Incidencia uContact](../Projects/Incidencia-uContact.md), [Módulo WhatsApp](../Projects/Modulo-WhatsApp.md)).

## Cómo trabajamos
- Cuando un usuario no puede ingresar y todo parece bien (existe, activo, campañas, sin bloqueos), lo primero es revisar **rol** (`employee_role`) y **skill** (`employee_skill`): sin rol no inicia sesión.
- Si sigue fallando: revisar `db_login` y estado, y escribirle por Teams a quien dijo que restableció la contraseña para que dé usuario y contraseña y validar el login.
- Para descartar restricciones de red: pedir la IP del equipo (`ipconfig` en cmd) y que la revise quien administra la red.

## Relacionado
- [Sergio](Sergio.md), dev principal

Volver a [Home](../Home.md)
