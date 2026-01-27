# TODO

## node-semantic-release
- Funciona, pero es necesario restringir permisos y permitir el commit del changelog de forma controlada. [RESUELTO] Se utiliza una github app y ruleset con bypass para hacer commits en branch protegidas solo por la gt app.

## create-release
- Funciona correctamente, pero falta ajustar la logica para que agregue el changelog. [RESUELTO] Se agrega la condicion de que si las release notes estan vacias se agregue el change log del tag indicado.
- Verificar como se maneja la comunidad y validar si no es podible usar el semantic reelase solo para generar el reelase y no el tag. [RESUELTO] Se logra generar el tag por separado en una action automatica y con una action manual se crea el release.

## rollback-release
- Agregar un job manual que dispare el commit en el repo de Argo con un release anterior indicado.

## aprobaciones
- Agregar aprobadores para pr de develop a main.
- Agregar aprobadores para action manual de creacion de release.

## Workflow custom
- probar el caso en el que los devs necesiten agregar sus workflows propios en su repó y se complementen con los existentes.

## Reglas
- contemplar bypass para el caso de hotfix

## Environments
- cambiar staging por stg para linear a la nomenclatura del values.stg.yaml y podes usar al misma variable para ambos
