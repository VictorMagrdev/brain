# Checklist: crear y arreglar usuarios (tickets GLPI)

Reglas que me dio [Mónica](../People/Monica.md). Las queries están en [Consultas SQL para tickets](Consultas-SQL-Tickets.md).

## Regla de oro
**Todo usuario debe tener siempre skill y rol.** Al crear cualquier usuario que pidan, se le asignan ambos, sin excepción.

## Roles
- **Fichaje:** es el rol por defecto. Si no sé qué rol poner, siempre es **Fichaje**, independiente de lo que sea.
- **Coordinador:** se agrega además de Fichaje cuando el usuario es supervisor o coordinador (aunque no exista un rol llamado "supervisor").

## Skills
- Asignar la skill según la campaña indicada en el ticket (ej: si dice Cruz Verde, todas las skills de Cruz Verde).
- **Nombre exacto, sin recortar:** si el Excel dice `PAPERNEST IN`, la skill es `PAPERNEST IN` (id 5471), no cualquiera que contenga "PAPERNEST". Hay skills con palabras en común. Asignar siempre por id verificado.
- Buscar las skills de la campaña con la query `SELECT id, name FROM skill WHERE name LIKE '%...%' AND active = 1`.

## Si el usuario "no puede ingresar" o sale "usuario no encontrado"
Orden de revisión:
1. ¿Existe el usuario? (si existe, no hay que crearlo aunque el ticket lo diga)
2. ¿Está activo? (`active = 1`)
3. ¿Tiene skills? Si no, asignarlas.
4. ¿Tiene rol? Si no, asignar Fichaje (+ Coordinador si es supervisor).
5. ¿Tiene preguntas de seguridad? Si no, reset de preguntas y dejar que las configure.
6. Reset de contraseña y, si hace falta, marcar `db_login` en 1 para que repita todo el proceso.

> Caso de referencia: ticket 54173. El usuario existía y estaba activo, pero no tenía skills ni rol; esa era la causa del "usuario no encontrado", no el reset de contraseña.

> Segundo caso (2026-10-19, "Reseteo usuario iattend", campaña Cruz Verde): soporte ya había reseteado la clave y seguía saliendo "usuario no encontrado". Causa: **faltaba el rol**. Solución: `INSERT` en `employee_role` con Fichaje (id 156, ver [Consultas SQL](Consultas-SQL-Tickets.md)). Resuelto. Moraleja: ante "usuario no encontrado", revisar rol y skill **antes** de resetear la clave.

## Prueba de inicio de sesión (regla de Mónica)
1. Poner `db_login` en **1** (login por base de datos; si estaba en 0, cambiarlo).
2. Asignar la contraseña **exacta** que está almacenada en el ambiente de **test** de iAttend (la que se puso antes puede no ser la correcta).
3. Probar que el usuario inicie sesión.
   - **Si inicia:** está bien configurado en BD.
   - **Si no inicia:** hay un problema de configuración.
4. Aplicar esto a los **usuarios afectados**, no a quien reportó el caso.
5. Si el usuario entra por usuario de red, pedir validar que el usuario de red sea el correcto y que usen la contraseña nueva.
6. Adjuntar evidencia del inicio de sesión en cada ticket: que se vea el usuario, e indicar qué usuario hay almacenado en BD.

## Si todo parece bien y aun así no entra
Existe, activo, con campañas y sin bloqueos, pero no inicia sesión:
1. Revisar la asociación en `employee_role`: **sin rol no inicia sesión** (fue la causa en un caso reciente).
2. Revisar `employee_skill`.
3. Revisar `db_login` y el estado.
4. Escribirle por Teams a quien restableció la contraseña para que dé usuario y contraseña y validar el login.
5. Si es de red: pedir la IP del equipo (`ipconfig`) para que quien administra la red revise restricciones.
6. Nota: la query de crear usuario (`INSERT` en `employee`) **no** asigna rol ni skill; hay que hacerlo aparte.

## Sonar
El escáner del proyecto completo pide permisos de administrador por las variables de entorno. La extensión de VS Code funciona, pero solo archivo por archivo.

## Si el usuario ya existe pero está mal creado
- No crear otro: corregir el nombre. Validar que la cédula esté bien, que no haya dos registros, y que tenga skill (el Excel del ticket la indica, ej: VERTI), rol y esté activo.
- Si el GLPI dice "crear" pero el usuario ya existe, validar y corregir en vez de duplicar.

Volver a [Home](../Home.md)
