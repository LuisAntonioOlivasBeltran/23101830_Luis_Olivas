# 01 - Protección de credenciales

## Objetivo

Ver por qué no se deben guardar contraseñas en texto claro y cómo guardarlas con un hash usando `crypt()` y `gen_salt()`.

## Comandos

Script: `sql/01-credenciales.sql`

```sql
-- tabla insegura (solo para la demostración)
CREATE TABLE usuarios_insegura (
    id BIGSERIAL PRIMARY KEY,
    usuario VARCHAR(50) NOT NULL,
    password VARCHAR(100) NOT NULL
);
INSERT INTO usuarios_insegura (usuario, password) VALUES ('juan', 'Password123');
SELECT * FROM usuarios_insegura;

-- hash (lo ejecuté dos veces)
SELECT crypt('Password123', gen_salt('bf'));
SELECT crypt('Password123', gen_salt('bf'));

-- tabla con hash
CREATE TABLE usuarios_segura (
    id BIGSERIAL PRIMARY KEY,
    usuario VARCHAR(50) UNIQUE NOT NULL,
    password_hash TEXT NOT NULL
);
INSERT INTO usuarios_segura (usuario, password_hash)
VALUES ('juan', crypt('Password123', gen_salt('bf')));
SELECT * FROM usuarios_segura;

-- verificar contraseña
SELECT usuario, password_hash = crypt('Password123', password_hash) AS contraseña_correcta
FROM usuarios_segura WHERE usuario = 'juan';

SELECT usuario, password_hash = crypt('OtraPassword', password_hash) AS contraseña_correcta
FROM usuarios_segura WHERE usuario = 'juan';
```

## Resultado obtenido

- Tabla `usuarios_insegura`: `[PEGAR SALIDA REAL]`
- Los dos hashes: `[PEGAR SALIDA REAL]`
- Tabla `usuarios_segura`: `[PEGAR SALIDA REAL]`
- Con `Password123`: `[PEGAR SALIDA REAL]`
- Con `OtraPassword`: `[PEGAR SALIDA REAL]`

## Análisis

**¿Qué pasaría si alguien puede leer la tabla insegura?**
Vería las contraseñas tal cual y podría entrar con la cuenta de cualquier usuario. Además, mucha gente usa la misma contraseña en varios lados, así que también podrían usarla en otras páginas.

**¿Los dos hashes son iguales?**
No. Aunque la contraseña es la misma, el resultado cambia cada vez.

**¿Por qué no son iguales?**
Porque `gen_salt()` crea una sal aleatoria diferente cada vez y esa sal se mezcla con la contraseña.

**¿Para qué sirve `gen_salt()`?**
Crea la sal aleatoria (y dice qué algoritmo se usa, `bf` es bcrypt). Gracias a eso dos usuarios con la misma contraseña no tienen el mismo hash.

**¿Para qué sirve `crypt()`?**
Calcula el hash de la contraseña con la sal. Para verificar, se le pasa el hash guardado como sal y se compara el resultado.

**¿Es cifrado reversible?**
No, el hash es de una sola vía. No hay forma de sacar la contraseña original a partir del hash.

**¿Por qué no hace falta recuperar la contraseña?**
Porque solo se necesita saber si la persona escribió la contraseña correcta. Se calcula el hash de lo que escribió y se compara con el guardado.

**¿Por qué un hash no es lo mismo que cifrado?**
El cifrado se puede revertir con una clave y el hash no. El cifrado es para datos que necesito volver a leer, el hash es para cosas que solo necesito verificar.

## Conclusión

Las contraseñas no se deben guardar en texto claro ni cifradas de forma que se puedan recuperar. Se guardan como hash con sal. Aquí lo hice con `crypt()` y `gen_salt('bf')` para entender la idea, pero en una aplicación real se usa el método que recomiende el framework y no se inventa uno propio.
