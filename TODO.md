# TODO

## node-semantic-release
- Funciona, pero es necesario restringir permisos y permitir el commit del changelog de forma controlada. [RESUELTO] Se utiliza una github app y ruleset con bypass para hacer commits en branch protegidas solo por la gt app.

## create-release
- Funciona correctamente, pero falta ajustar la logica para que agregue el changelog.
- Verificar como se maneja la comunidad y validar si no es podible usar el semantic reelase solo para generar el reelase y no el tag.

## rollback-release
- Agregar un job manual que dispare el commit en el repo de Argo con un release anterior indicado.
