# 07 - Comparación: datos guardados y datos en tránsito

## Objetivo

Ver qué se usa para proteger cada cosa y qué es lo que protege.

## Procedimiento

Completé la tabla del enunciado. No usé comandos.

## Resultado obtenido

| Situación | Qué se usa | Qué protege |
|---|---|---|
| Contraseña de usuario | Hash con sal (`crypt` y `gen_salt`; en una app real bcrypt o Argon2) | Que si se filtra la base no se vea la contraseña, solo se puede comprobar |
| RFC guardado en una tabla | Cifrado de la columna (`pgp_sym_encrypt`) y permisos | El dato guardado: sin la clave no se puede leer el RFC |
| Consulta del cliente al servidor | TLS (mejor con `verify-full`) | Lo que viaja por la red y que el servidor sea el correcto |
| Respaldo de la base | Cifrar el respaldo y cuidar quién entra a los archivos | Que si lo roban o lo pierden no puedan leer los datos |
| Clave criptográfica | Un lugar seguro para secretos, fuera del código | Que la clave no se filtre ni esté junto a lo que protege |

## Análisis

Unas cosas protegen lo que está guardado (hash, cifrado de columna, respaldos cifrados) y otra protege lo que viaja (TLS). No se reemplazan entre sí. La clave también necesita su propia protección porque el cifrado depende de ella.

## Conclusión

Cada situación tiene un riesgo distinto, así que cada una usa algo distinto. Lo mejor es combinarlas y siempre con control de acceso.
