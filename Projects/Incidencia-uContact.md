# Incidencia: timeout al subir lista a uContact

**Síntoma:** al subir la lista a uContact, el back 2 responde `No se pudo subir la lista a uContact. Motivo: timeout of 60000ms exceeded`.
**Causa identificada:** los servidores de **test (10.36.35.8)** y **producción (192.168.100.91)** no tienen acceso a la instancia `https://interactivo3op.ucontactcloud.com`.
**Estado:** cerrado (informativo). Se pidió el acceso y se revisaron los logs de test del back 2 con [Checho](../People/Checho.md), como pidió [Mónica](../People/Monica.md).

## Qué se necesitaba (histórico)
- [ ] Que habiliten el acceso de ambos servidores a uContact.
- [ ] Acceso a los logs de test del back 2 (revisar con Checho).
- [ ] Reprobar tras el acceso.

## Contexto
Forma parte de la [prueba del módulo de WhatsApp](Modulo-WhatsApp.md): además del timeout, había un problema de ruta que no aparecía en el sidebar y un texto del nombre del módulo en el front (ya corregido).

Volver a [Home](../Home.md)
