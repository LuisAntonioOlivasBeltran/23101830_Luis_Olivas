# 03 - Gestión de claves

## Objetivo

Entender por qué es mala idea poner las claves dentro del código y revisar que mi repositorio no tenga secretos.

## Comandos

Para la parte de mala práctica creé de forma temporal `sql/ejemplo-mala-practica.sql` con:

```sql
SELECT pgp_sym_decrypt(rfc_cifrado, 'CLAVE-DE-PRUEBA')
FROM clientes_protegidos;
```

Ese archivo **no se queda en la entrega final** (lo borré antes del commit).

Revisión de secretos antes de subir a GitLab (Actividad 9):

```bash
grep -RniE 'password|passwd|secret|token|api_key|clave' .
grep -Rni 'postgresql://' .
```

## Resultado obtenido

- Resultado del primer `grep`: `[PEGAR SALIDA REAL]`
- Resultado del segundo `grep`: `[PEGAR SALIDA REAL]`

## Análisis

**¿Qué pasaría si la clave fuera real y el archivo se subiera a Git?**
Cualquiera con acceso al repositorio podría descifrar los datos. Aunque borre el archivo después, la clave se queda en el historial de commits y en las copias que ya se hayan clonado. Habría que cambiar la clave y volver a cifrar todo.

**¿Y si el repositorio es privado?**
Sigue sin ser buena idea. Menos gente lo ve, pero igual se puede clonar, respaldar o cambiar a público después.

**Otras formas en que se puede exponer una clave:** scripts, archivos de configuración, registros de ejecución, capturas de pantalla, respaldos, documentación y variables mal manejadas.

**Sobre el `grep`:** salieron coincidencias porque en mis propios textos uso palabras como "clave" y "password". Las revisé una por una y todas son valores inventados de laboratorio, no hay claves ni contraseñas reales. Esta búsqueda no encuentra todos los secretos posibles, pero sirve para agarrar el hábito de revisar antes de subir.

## Conclusión

La clave es lo más importante del cifrado: si se filtra, el cifrado ya no sirve. Las claves deben estar fuera del código y de los scripts, y en el laboratorio solo usé valores ficticios.
