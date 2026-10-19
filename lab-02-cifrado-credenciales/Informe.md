# 22. Informe final

LAB-02 - Cifrado, credenciales y comunicaciones. Las evidencias están en la carpeta `evidencias/` y los scripts en `sql/`.

## 1. Introducción

En el LAB-01 vimos cómo clasificar los datos y cómo ocultarlos o limitar quién los ve. En este laboratorio vimos qué pasa si alguien llega a los datos o los intercepta: cómo guardar contraseñas, cómo cifrar datos sensibles, cómo proteger la comunicación con TLS y por qué hay que cuidar las claves.

## 2. Protección de credenciales

- **Texto claro:** el dato se lee directamente. Si alguien lee la tabla, ve las contraseñas.
- **Hash:** se calcula con `crypt()` y `gen_salt('bf')`. No se puede revertir, solo sirve para comparar. La sal hace que la misma contraseña dé un hash diferente cada vez.
- **Cifrado:** se puede revertir con una clave (`pgp_sym_encrypt`). Sirve para datos que hay que volver a leer, no para contraseñas.

## 3. Protección de datos sensibles

Protegí el **RFC** porque es un dato fiscal único que identifica a una persona (en el LAB-01 lo clasifiqué como dato a proteger). Lo guardé cifrado en `clientes_protegidos.rfc_cifrado` con `pgp_sym_encrypt` y lo recuperé con `pgp_sym_decrypt` usando la clave. Con una clave incorrecta marca error.

## 4. TLS

TLS sirve para proteger la información mientras viaja entre el cliente y PostgreSQL, porque sin eso alguien podría leerla desde la red. En PostgreSQL revisé `SHOW ssl`, `SHOW ssl_min_protocol_version`, `SHOW port`, `SHOW listen_addresses` y la tabla `pg_stat_ssl` para ver si mi sesión usaba TLS. Resultado: `[RESUMIR MI SALIDA REAL]`.

También vi los modos de `sslmode`. `require` cifra pero no verifica bien al servidor, y `verify-full` verifica el certificado y el nombre del servidor.

## 5. Gestión de claves

Las claves no deben guardarse junto con los scripts porque quien tenga el script tiene la clave y puede descifrar los datos. En Git es peor, porque aunque borre el archivo, la clave se queda en el historial. Lo correcto es guardarlas en un gestor de secretos, con acceso limitado, cambiándolas cada cierto tiempo y con un respaldo seguro.

## 6. Resultados

- Vi la diferencia entre una contraseña en texto claro y una con hash, y comprobé que el hash cambia cada vez que lo genero.
- Cifré y descifré un RFC y en la tabla ya no se ve en claro.
- Revisé el estado de TLS en mi sesión.
- Hice la comparación entre protección en reposo y en tránsito, el análisis del escenario y una política de claves con 10 reglas.
- `[AGREGAR LO QUE ME SALIÓ EN MI EJECUCIÓN]`

## 7. Problemas encontrados

`[ESCRIBIR LOS ERRORES O DIFICULTADES QUE SÍ ME PASARON, por ejemplo si pgcrypto no estaba, si TLS estaba apagado, errores al descifrar, etc.]`

## 8. Conclusiones

Aprendí que cada técnica sirve para un riesgo distinto: hash para contraseñas, cifrado para datos guardados y TLS para lo que viaja por la red. También que el cifrado vale lo que vale la protección de su clave, y que ninguna técnica reemplaza a los roles y permisos, sino que se complementan.

Si fuera un sistema real usaría hash para las contraseñas con el método que recomiende el framework, cifraría los datos sensibles y los respaldos, usaría TLS con `verify-full`, guardaría las claves en un gestor de secretos y revisaría que no haya secretos antes de cada commit.

---

# 23. Preguntas de reflexión

**1. ¿Por qué una contraseña no debe almacenarse como texto claro?**
Porque si alguien lee la tabla, un respaldo o hay una filtración, ya tiene las contraseñas. Y como mucha gente repite contraseñas, las podría usar en otros lugares.

**2. ¿Por qué un hash no es equivalente a un cifrado?**
El cifrado se puede revertir con una clave y el hash no. El hash solo sirve para comprobar si una contraseña es la correcta.

**3. ¿Por qué el cifrado de una columna no sustituye al control de acceso?**
Porque quien tenga permiso de descifrar o tenga la clave puede ver el dato. El cifrado no decide quién debe ver qué, y además no protege otras columnas, la red ni los permisos de más.

**4. ¿Por qué TLS no sustituye a los mecanismos de autorización de PostgreSQL?**
TLS solo protege el camino de la información. No limita lo que puede hacer cada usuario, así que un usuario con demasiados permisos sigue viendo todo aunque la conexión sea cifrada.

**5. ¿Qué ocurriría si una organización pierde una clave usada para cifrar información?**
Los datos cifrados con esa clave ya no se podrían recuperar, a menos que exista un respaldo seguro de la clave.

**6. ¿Qué ocurriría si una clave criptográfica se publica accidentalmente en GitLab?**
Hay que considerarla comprometida, porque cualquiera con acceso puede descifrar los datos y la clave queda en el historial. Habría que cambiarla, volver a cifrar los datos y revisar los accesos.

**7. ¿Por qué un repositorio privado no debe considerarse un almacén de secretos?**
Porque aunque sea privado lo pueden ver varias personas, se puede clonar o respaldar, puede volverse público por error y guarda el historial. Los secretos necesitan un lugar con su propio control de acceso.

**8. ¿Qué diferencia existe entre proteger información almacenada y proteger información durante su transmisión?**
La almacenada se protege en la base y en los respaldos (hash, cifrado de columna). La que viaja se protege con TLS. Son cosas distintas y una no reemplaza a la otra.

**9. ¿Por qué la gestión de claves es tan importante como el algoritmo criptográfico?**
Porque aunque el algoritmo sea muy bueno, si la clave está expuesta, es débil o se pierde, no protege nada.

**10. ¿Qué mecanismo adicional implementaría para proteger los respaldos?**
Cifrarlos, guardar la clave por separado, limitar quién puede acceder a ellos, tener una copia en otro lugar y probar cada cierto tiempo que se puedan restaurar.
