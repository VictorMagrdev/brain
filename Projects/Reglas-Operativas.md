# Reglas operativas (recapitulación de todas las conversaciones)

Resumen de lo que [Mónica](../People/Monica.md) me ha dicho en todas las conversaciones. Si ya me lo dijo una vez, **debe estar aquí**: no quiero que me lo repita.

## Las reglas que nunca se olvidan
1. **Todo usuario debe tener siempre skill y rol.** Al crear, al arreglar y al revisar cualquier ticket, sin excepción.
2. **El rol por defecto es siempre Fichaje**, independiente de lo que sea. Si no sé qué rol poner, es Fichaje.
3. **Supervisores y coordinadores:** Fichaje **+ Coordinador**. No existe un rol "supervisor".
4. **Sin rol no inicia sesión.** Es lo primero que reviso cuando alguien "no puede ingresar" aunque todo lo demás esté bien.
5. **Skill según la campaña** del ticket o del Excel (ej: todas las de Cruz Verde, VERTI).
6. **Si el usuario ya existe, no lo creo de nuevo:** corrijo lo que esté mal (nombre, estado, skill, rol). Valido que la cédula esté bien y que no haya dos registros.

## Checklist al recibir un ticket de usuario
- [ ] ¿Existe? (buscar por documento)
- [ ] ¿Está activo? (`active = 1`)
- [ ] ¿Tiene skill? (`employee_skill`)
- [ ] ¿Tiene rol? (`employee_role`: Fichaje, + Coordinador si es supervisor)
- [ ] ¿Tiene preguntas de seguridad? Si no: reset de preguntas.
- [ ] ¿Bloqueado? (`blocked_user = 0`)
- [ ] ¿`db_login` en 1 si debe entrar por BD? Contraseña exacta de test.
- [ ] ¿Inicia sesión? Probar antes de responder.

## Cómo probar el inicio de sesión
- Id del rol **Fichaje = 156** (ver [Consultas SQL](Consultas-SQL-Tickets.md)).
- `db_login` en 1 y la contraseña **exacta** almacenada en el ambiente de test de iAttend.
- Si inicia, está bien configurado; si no, es un problema de configuración (rol, skill, estado).
- Hacerlo a los usuarios **afectados**, no a quien reportó el caso.
- Adjuntar evidencia en cada ticket: que se vea el usuario, y el usuario almacenado en BD.
- Si sigue fallando: escribir por Teams a quien restableció la contraseña para que dé usuario y contraseña, y pedir la IP del equipo (`ipconfig`) por si hay restricción de red.

## Cuando se termina una tarea
- [ ] Funcionando en **test**.
- [ ] Evidencia subida en la tarea.
- [ ] Evidencia de lo solucionado en **Sonar**.
- [ ] Tareas liberadas.
- [ ] Flujo Git completo: [Flujo Git](Flujo-Git.md).

## Forma de trabajar
- Antes de un desarrollo: pitch con [Plantilla de Pitch](../Templates/Plantilla-Pitch.md).
- Ante la duda de si corrijo el usuario o respondo al GLPI, pregunto a Mónica y valido todo (nombre, cédula, skill, rol, activo) primero.
- Si el front publicado no se ve: probar otro navegador antes de escalar.
- Sonar con el escáner completo pide permisos de administrador (variables de entorno); la extensión de VS Code solo analiza archivo por archivo.
- Todo lo que me dice Mónica por segunda vez se anota **aquí**.

## Notas relacionadas
- [Checklist de usuarios GLPI](Checklist-Usuarios-GLPI.md)
- [Consultas SQL para tickets](Consultas-SQL-Tickets.md)

Volver a [Home](../Home.md)
