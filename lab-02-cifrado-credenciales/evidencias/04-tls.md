# 04 - TLS

## Objetivo

Entender qué pasa con la información mientras viaja entre el cliente y PostgreSQL, y revisar si mi conexión usa TLS.

## Comandos

Script: `sql/03-verificacion-tls.sql`

```sql
SHOW ssl;
SHOW ssl_min_protocol_version;
SHOW port;
SHOW listen_addresses;

SELECT inet_client_addr(), inet_client_port();

SELECT ssl, version, cipher
FROM pg_stat_ssl
WHERE pid = pg_backend_pid();
```

Con `psql` (cambiando los datos por los de mi entorno):

```bash
psql "host=SERVIDOR port=5432 dbname=TU_BASE user=TU_USUARIO sslmode=require"
```

## Resultado obtenido

- `SHOW ssl`: `[PEGAR SALIDA REAL]`
- `SHOW ssl_min_protocol_version`: `[PEGAR SALIDA REAL]`
- `SHOW port`: `[PEGAR SALIDA REAL]`
- `SHOW listen_addresses`: `[PEGAR SALIDA REAL]`
- `inet_client_addr()` y `inet_client_port()`: `[PEGAR SALIDA REAL]`
- `pg_stat_ssl` (columnas `ssl`, `version`, `cipher`): `[PEGAR SALIDA REAL]`

## Análisis

**¿Por qué importa TLS?**
Cuando hago un `SELECT * FROM clientes;`, la consulta y el resultado viajan por la red. Si la conexión no está protegida, alguien que vea el tráfico podría leer esos datos. TLS cifra esa comunicación.

**¿La conexión actual usa TLS?** `[según mi resultado: ssl = t o f]`

**¿Qué versión de TLS se usa?** `[la de la columna version, por ejemplo TLSv1.3. Si no hay TLS sale vacía]`

**¿Qué algoritmo de cifrado aparece?** `[el de la columna cipher. Si no hay TLS sale vacío]`

**¿Qué diferencia hay entre una conexión con TLS y una sin TLS?**
Con TLS la información viaja cifrada y se puede comprobar que el servidor es el correcto. Sin TLS viaja en texto claro y se puede leer o modificar.

**Modos de `sslmode`:**

| Modo | Qué hace |
|---|---|
| `disable` | No usa TLS |
| `prefer` | Usa TLS si el servidor puede, si no, se conecta sin TLS |
| `require` | Obliga a usar TLS, pero no comprueba bien quién es el servidor |
| `verify-ca` | Usa TLS y comprueba que el certificado venga de una autoridad confiable |
| `verify-full` | Hace lo de `verify-ca` y además comprueba que el nombre del servidor coincida con el certificado |

Para una validación fuerte hay que entender `verify-full`.

## Conclusión

TLS protege los datos mientras viajan, pero no decide quién puede ver qué dentro de la base de datos. Eso lo siguen haciendo los roles y permisos. También vi que `require` cifra pero no verifica bien al servidor.
