# 08 - Análisis de un escenario de seguridad

## Objetivo

Encontrar los riesgos de una empresa que tiene bien los usuarios y roles, pero tiene otras fallas.

## Procedimiento

Analicé el escenario del enunciado, sin comandos. La empresa guarda nombre, correo, contraseña, RFC, información financiera y claves. Las contraseñas y el RFC están en texto claro, las conexiones no validan bien al servidor, las claves están en archivos SQL y los respaldos no tienen protección.

## Resultado obtenido

### Pregunta 1. ¿Cuáles son los principales riesgos?

| Problema | Riesgo |
|---|---|
| Contraseñas en texto claro | Si alguien lee la tabla o se filtra la base, ya tiene las contraseñas listas para usar |
| RFC en texto claro | Se expone un dato fiscal si hay una filtración o alguien accede a los archivos |
| Conexiones sin validar al servidor | Alguien podría hacerse pasar por el servidor o espiar el tráfico |
| Claves en archivos SQL | Quien vea los scripts se lleva la clave y puede descifrar los datos |
| Respaldos sin protección | Si roban un respaldo, tienen toda la base sin pasar por los permisos |

### Pregunta 2. ¿Qué usaría para cada elemento?

| Elemento | Qué usaría |
|---|---|
| Nombre y correo | Roles y permisos, y enmascarar si alguien no necesita verlos completos |
| Contraseña | Hash con sal |
| RFC | Cifrado de la columna y enmascarado para mostrarlo |
| Información financiera | Cifrado, acceso mínimo y registro de quién la consulta |
| Claves | Un gestor de secretos, fuera del repositorio y cambiándolas cada cierto tiempo |
| Conexiones | TLS con `sslmode=verify-full` |
| Respaldos | Cifrarlos y controlar quién accede |

### Pregunta 3. ¿Cuál problema es el más crítico y por qué?

Para mí el más crítico son **las claves en archivos SQL**. Si se expone la clave, el cifrado ya no sirve y se pueden abrir todos los datos que protegía, incluidos los respaldos cifrados con esa clave. Además, arreglarlo significa cambiar la clave y volver a cifrar todo. Las contraseñas en texto claro también son muy graves, pero lo de las claves afecta a todo lo demás.

### Pregunta 4. ¿Tener bien los usuarios y permisos elimina estos riesgos?

No. Los permisos solo controlan quién consulta dentro de PostgreSQL. No protegen un respaldo robado, lo que viaja por la red, una clave publicada en un script ni contraseñas en texto claro. Se necesitan varias capas: permisos, cifrado, TLS, buen manejo de claves y respaldos protegidos.

## Análisis

Aunque los roles estuvieran perfectos, las demás fallas dejan los datos expuestos por otros lados. Los permisos ayudan, pero solos no alcanzan.

## Conclusión

La seguridad no depende de una sola cosa. Hay que combinar los permisos con hash, cifrado, TLS, cuidado de las claves y respaldos protegidos.
