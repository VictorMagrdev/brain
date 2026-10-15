# Consultas SQL para tickets

Plantillas para resolver la mayoría de tickets de soporte (esquema `dev_miattend_interactivo`). Los valores son de ejemplo: reemplázalos por los del ticket. **Antes de cualquier UPDATE/DELETE, verifica con un SELECT que el `WHERE` afecta solo a quien debe.**

> **Recordatorio:** todo usuario debe tener siempre skill y rol (por defecto **Fichaje**; supervisores: Fichaje + Coordinador). Ver [Reglas operativas](Reglas-Operativas.md).

## Skills

```sql
-- Validar qué skills tiene un empleado
SELECT * FROM employee_skill WHERE employee_id = <EMPLOYEE_ID>;

-- Asignar skill
INSERT INTO employee_skill (employee_id, skill_id) VALUES (<EMPLOYEE_ID>, <SKILL_ID>);

-- Validar campañas (buscar skill por nombre)
SELECT id, name FROM skill WHERE name LIKE '%<NOMBRE_CAMPAÑA>%' AND active = 1;
```

## Permisos

```sql
-- Asignar permiso
INSERT INTO employee_permission (employee_id, permission_id) VALUES (<EMPLOYEE_ID>, <PERMISSION_ID>);
```

## Consultar y editar empleado

```sql
-- Consultar el id del empleado por documento
SELECT id FROM employee WHERE document = '<DOCUMENTO>';

-- Cambiar usuario, nombre o apellido
UPDATE dev_miattend_interactivo.employee
SET last_name = '<NUEVO_APELLIDO>'
WHERE last_name = '<APELLIDO_ACTUAL>';

-- Activar usuario (1 = activo)
UPDATE employee SET active = 1 WHERE document = '<DOCUMENTO>';
```

## Crear usuario

Patrón habitual: usuario = documento y contraseña = MD5 del documento. Valores fijos usados hasta ahora: `home_working=1`, `network_permit=1`, `img_url='avatar1.png'`, `id_category=5`, `assignment=3`, `agreement=6`, `id_company=24`, `id_employment_relationship=40`, `break_permission=1`, `country_id=3`.

```sql
INSERT INTO dev_miattend_interactivo.employee (
    user, password, first_name, last_name, document,
    home_working, network_permit, img_url, id_category, assignment,
    agreement, id_company, id_employment_relationship, break_permission, country_id
) VALUES (
    '<DOCUMENTO>', MD5('<DOCUMENTO>'), '<NOMBRES>', '<APELLIDOS>', '<DOCUMENTO>',
    1, 1, 'avatar1.png', 5, 3, 6, 24, 40, 1, 3
);
```

Variante con campaña: agrega la columna `id_campaign` y su valor al final (`..., 3, <ID_CAMPAÑA>`).

## Accesos y seguridad

```sql
-- Reset de contraseña (se deja el documento o la clave estándar vigente)
UPDATE employee SET password = MD5('<NUEVA_CLAVE>') WHERE document = '<DOCUMENTO>';

-- Desbloquear usuario (0 = desbloqueado)
UPDATE employee SET blocked_user = 0 WHERE document = '<DOCUMENTO>';

-- Reset de preguntas de seguridad: 1) marcar en 0, 2) borrar las anteriores
UPDATE employee SET security_questions = 0 WHERE document = '<DOCUMENTO>';
DELETE FROM employee_security_questions WHERE employee_id = '<EMPLOYEE_ID>';
```

Para el reset de preguntas, obtén primero el `employee_id` con la consulta por documento.

Reglas de qué skill y rol asignar: [Checklist de usuarios GLPI](Checklist-Usuarios-GLPI.md).

Volver a [Home](../Home.md) · Relacionado: [Herramientas Internas](Herramientas-Internas.md)
