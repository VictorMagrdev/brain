# Flujo Git para subir cambios

Flujo estándar para subir cambios en los proyectos principales. Reemplaza `<rama-feature>` por la rama en la que trabajas (ej: `asignarCoordinador`).

Orden de ramas: **feature → integration → develop → test → produccion**

## 1. Subir mi rama

```bash
git checkout <rama-feature>
git add .
git commit -m "Mis cambios en <rama-feature>"
git push
```

## 2. Integration

```bash
git checkout integration
git pull
git merge <rama-feature>
git push
```

## 3. Develop

```bash
git checkout develop
git pull
git merge <rama-feature>
git push
```

## 4. Test

```bash
git checkout test
git pull
git merge <rama-feature>
git push
```

## 5. Producción

```bash
git checkout produccion
git pull
git merge <rama-feature>
git push
```

## Al terminar una tarea (lo que pide Mónica)
- [ ] Funcionando en **test**.
- [ ] Evidencia subida en la tarea.
- [ ] Evidencia de lo solucionado en **Sonar**.
- [ ] Tareas liberadas.

## Notas
- En el `git merge` va **solo** la rama feature. Escribir además el nombre de la rama destino (`git merge feature integration`) hace un merge de dos ramas a la vez, que no es lo que se busca.
- Haz siempre `git pull` antes del `merge` para no mezclar sobre una rama desactualizada.
- Usa el **mismo** nombre de rama feature en todos los pasos (en el ejemplo original se mezclaban `asignarCoordinador` y `feature/confirmaciones-medicarte`).
- Antes de producción, confirma que el cambio ya pasó por test.

Volver a [Home](../Home.md) · Relacionado: [Herramientas Internas](Herramientas-Internas.md)
