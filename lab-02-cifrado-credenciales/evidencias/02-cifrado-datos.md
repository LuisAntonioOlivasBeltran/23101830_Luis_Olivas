# 02 - Cifrado de información sensible

## Objetivo

Cifrar un dato sensible (el RFC) para que no se vea en la tabla, pero que se pueda recuperar con una clave.

## Comandos

Script: `sql/02-cifrado-datos.sql`

```sql
CREATE TABLE clientes_protegidos (
    id BIGSERIAL PRIMARY KEY,
    nombre VARCHAR(120) NOT NULL,
    rfc_cifrado BYTEA NOT NULL
);

INSERT INTO clientes_protegidos (nombre, rfc_cifrado)
VALUES ('Ana López', pgp_sym_encrypt('LOPA900101ABC', 'CLAVE-DE-PRUEBA'));

SELECT * FROM clientes_protegidos;

SELECT id, nombre, pgp_sym_decrypt(rfc_cifrado, 'CLAVE-DE-PRUEBA') AS rfc
FROM clientes_protegidos;
```

La clave `CLAVE-DE-PRUEBA` y el RFC son inventados, solo para el laboratorio.

## Resultado obtenido

- Al hacer `SELECT *`: `[PEGAR SALIDA REAL]`
- Al descifrar con la clave correcta: `[PEGAR SALIDA REAL]`
- Al probar con una clave incorrecta: `[PEGAR SALIDA REAL]`

## Análisis

**¿Qué diferencia hay entre `crypt()` y `pgp_sym_encrypt()`?**
`crypt()` hace un hash que no se puede revertir y sirve para contraseñas. `pgp_sym_encrypt()` cifra con una clave y el dato se puede recuperar con `pgp_sym_decrypt()`.

**¿Cuál permite recuperar el dato original?**
`pgp_sym_encrypt()`, siempre que tenga la clave correcta.

**¿Qué pasa si uso una clave incorrecta?**
Marca error y no devuelve el dato.

**¿Qué problema hay con poner la clave directo en la consulta?**
Que la clave queda escrita en el código. Puede acabar en scripts, en el historial, en los registros del servidor, en capturas o en un repositorio, y quien la vea puede descifrar todo.

## Conclusión

El cifrado hace que el RFC no se vea en la tabla, pero no reemplaza a los permisos: si alguien tiene la clave, puede ver el dato. Todo depende de cómo se cuide la clave.
