# 10 - Política de protección de claves

## Objetivo

Escribir mis propias reglas para cuidar las claves de cifrado.

## Procedimiento

Escribí 10 reglas que cubren: generación, almacenamiento, acceso, rotación, respaldo, revocación, exposición accidental y eliminación.

## Resultado obtenido

1. **Generación:** las claves se crean con un generador seguro y deben ser largas. No se hacen con palabras o fechas fáciles y no se usa la misma en laboratorio y en producción.
2. **Almacenamiento:** se guardan en un gestor de secretos o un lugar seguro. Nunca en el código, en scripts SQL ni en el repositorio.
3. **Acceso:** solo la usa quien la necesita para su trabajo y se anota quién la usa.
4. **Separación:** la clave no se guarda junto con los datos que protege ni con sus respaldos, para que robar una cosa no dé acceso a la otra.
5. **Rotación:** se cambian cada cierto tiempo, y también si alguien con acceso se va o si hay sospecha de filtración. Al cambiarla se vuelven a cifrar los datos.
6. **Respaldo:** debe haber una copia segura de la clave guardada aparte, porque si se pierde no se pueden recuperar los datos. También se prueba que ese respaldo funcione.
7. **Revocación:** si una clave se compromete o ya no se necesita, se deja de usar de inmediato y se reemplaza, anotando cuándo y por qué.
8. **Si se expone sin querer:** si una clave sale en un repositorio, un log o una captura, se considera comprometida aunque se borre. Se cambia y se vuelve a cifrar lo que protegía.
9. **Revisión:** antes de cada commit se revisan los archivos buscando claves o contraseñas, y se evita escribirlas en consultas que queden en los logs.
10. **Eliminación:** cuando una clave ya no se necesita, se borra de todos lados, incluidos los respaldos, y se deja registro.

## Análisis

La idea de estas reglas es que una clave no dependa de un solo archivo o persona, y que si se filtra se pueda detectar, cambiar y volver a cifrar los datos.

## Conclusión

De nada sirve un buen algoritmo si la clave está mal cuidada. Por eso una política de claves es tan importante como el cifrado.
