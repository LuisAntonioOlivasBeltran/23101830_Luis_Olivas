# 19. Informe

Informe de las actividades de protección de datos realizadas en PostgreSQL 18 (pgAdmin 4) sobre la tabla `clientes`, con datos de prueba (5 registros).

**Estructura de la tabla `clientes`:** `id`, `nombre`, `correo`, `telefono`, `rfc`, `edad`, `ciudad`, `codigo_postal`, `fecha_alta`.

---

## 19.1 Clasificación

Se analizó cada campo para determinar si identifica directamente, si puede contribuir a identificar y si requiere protección.

| Campo | ¿Identifica directamente? | ¿Puede contribuir a identificar? | ¿Requiere protección? | Justificación |
|---|---|---|---|---|
| `id` | No | Sí | Sí | Identificador interno único del registro. |
| `nombre` | Sí | Sí | Sí | Identificador directo de la persona física. |
| `correo` | Sí | Sí | Sí | Identifica al titular y es un canal de contacto personal. |
| `telefono` | Sí | Sí | Sí | Dato de contacto vinculado directamente a una persona. |
| `rfc` | Sí | Sí | Sí | Clave fiscal oficial y única. |
| `edad` | No | Sí | Sí | Combinada con la ubicación facilita la reidentificación. |
| `ciudad` | No | Sí | Sí | Acota el universo de personas en un análisis geográfico. |
| `codigo_postal` | No | Sí | Sí | Define una zona geográfica pequeña y aumenta el riesgo de reidentificación cruzada. |
| `fecha_alta` | No | No | No | Dato operativo interno, sin riesgo de privacidad por sí mismo. |

**Datos que requieren protección y por qué**

- **Identificadores directos** (`nombre`, `correo`, `telefono`, `rfc`): permiten señalar a una persona por sí solos, por lo que requieren la protección más fuerte.
- **Cuasi-identificadores** (`edad`, `ciudad`, `codigo_postal`): no identifican por sí solos, pero la combinación de edad + ciudad + código postal puede reducir mucho el universo de personas y permitir la reidentificación.
- **Identificador interno** (`id`): es el vínculo con el resto de los datos y con la tabla de correspondencia, por lo que también se protege.
- **`fecha_alta`**: no requiere protección por sí misma, aunque puede sumar riesgo si se cruza con otros atributos (ver 19.4).

**Nota jurídica:** no todo dato que debe protegerse es un dato personal *sensible*. Los datos sensibles son los que afectan la esfera más íntima del titular. Sin embargo, cualquier dato que identifique o permita perfilar a una persona (RFC, teléfono, etc.) requiere medidas de seguridad y privacidad.

---

## 19.2 Enmascaramiento

**Qué se ocultó**

Se creó la vista `v_clientes_soporte`, que muestra solo parte de los datos:

| Campo | Ejemplo de resultado | Regla aplicada |
|---|---|---|
| `correo` | `an*****@example.test` | Primeros 2 caracteres + `*****` + dominio desde la `@` |
| `telefono` | `******4567` | `******` + últimos 4 dígitos |
| `rfc` | `*********SABC` | `*********` + últimos 4 caracteres |

Los campos `id` y `nombre` se muestran sin modificar. Los campos `edad`, `ciudad`, `codigo_postal` y `fecha_alta` no se incluyen en la vista.

**Para quién**

Para el rol `soporte` (creado con `CREATE ROLE soporte NOLOGIN`), que atiende a clientes y no necesita ver los datos completos.

**Por qué**

- Aplicar el **principio de mínimo privilegio**: soporte solo ve lo necesario para su trabajo.
- Ocultar los datos únicamente en la aplicación no basta: si alguien se conecta directo a la base de datos vería la información completa.
- Si soporte tuviera acceso directo a `clientes`, vería información confidencial que no necesita.

**Mecanismo utilizado**

Enmascaramiento dinámico mediante una **vista SQL** (`CONCAT`, `LEFT`, `RIGHT`, `SUBSTRING`, `POSITION`) combinado con **permisos (GRANT/REVOKE)**:

```sql
GRANT USAGE ON SCHEMA public TO soporte;
GRANT SELECT ON v_clientes_soporte TO soporte;
REVOKE ALL ON clientes FROM soporte;
```

**Verificación:** `SELECT * FROM v_clientes_soporte` devuelve los 5 registros enmascarados, mientras que `SELECT * FROM clientes` termina en `ERROR: permiso denegado a la tabla clientes` (SQL state 42501). La tabla original conserva todos los datos completos.

---

## 19.3 Seudonimización

**Cómo se generaron los tokens**

Se crearon dos esquemas, `pii` y `analitica`, y la tabla de correspondencia:

```sql
CREATE TABLE pii.mapa_cliente (
    token UUID PRIMARY KEY,
    cliente_id BIGINT UNIQUE REFERENCES public.clientes(id)
);

CREATE EXTENSION IF NOT EXISTS pgcrypto;

INSERT INTO pii.mapa_cliente (token, cliente_id)
SELECT gen_random_uuid(), id FROM clientes;
```

Cada cliente recibió un **UUID aleatorio** generado con `gen_random_uuid()` (5 tokens para 5 clientes). Como el token es aleatorio, no se puede deducir el cliente a partir de él.

Con ese mapa se construyó el conjunto para analítica, que reemplaza la identidad por el token y excluye nombre, correo, teléfono y RFC:

```sql
CREATE TABLE analitica.clientes AS
SELECT m.token AS cliente_token, c.edad, c.ciudad, c.codigo_postal, c.fecha_alta
FROM pii.mapa_cliente m
JOIN clientes c ON c.id = m.cliente_id;
```

**Dónde se encuentra el mapa de correspondencia**

En la tabla `pii.mapa_cliente`, dentro del esquema `pii`, separado del esquema `analitica` y de `public`.

**Quién debería poder acceder a él**

Solo un rol administrativo o de custodia de datos (por ejemplo, responsable de protección de datos o administrador autorizado), con acceso auditado y justificado. El rol `analista` **no** debe tener acceso.

**Por qué se separaron los permisos**

Se aplicó **separación de funciones** y **mínimo privilegio**. El rol `analista` (`NOLOGIN`) solo recibió:

```sql
GRANT USAGE ON SCHEMA analitica TO analista;
GRANT SELECT ON analitica.clientes TO analista;
```

Se comprobó que `analista` recibe `permiso denegado al esquema pii` al consultar `pii.mapa_cliente` y `permiso denegado a la tabla clientes` al consultar `public.clientes`. Si el analista pudiera consultar el mapa, podría relacionar los tokens con las identidades reales y se rompería por completo el propósito de la seudonimización.

**Seudonimización ≠ anonimización:** los datos aún pueden volver a vincularse con su titular usando la tabla de correspondencia y el token, por lo que siguen siendo datos personales.

---

## 19.4 Anonimización

**Qué identificadores fueron eliminados**

`nombre`, `correo`, `telefono` y `rfc`. Los resultados se obtienen únicamente con consultas agregadas, sin mostrar registros individuales.

**Qué atributos fueron generalizados**

- **Edad:** convertida en rangos (`18-30`, `31-45`, `46+`). Resultado: `31-45` = 4 personas y `46+` = 1 persona.
- **Región:** `codigo_postal` reducido a sus 2 primeros dígitos con `LEFT(codigo_postal, 2)`. Resultado: región `80` = 3, región `81` = 1, región `82` = 1.
- **Combinación de ambos:** `31-45`/`80` = 3, `31-45`/`81` = 1 y `46+`/`82` = 1.

**Qué grupos fueron suprimidos**

Se aplicó un umbral mínimo de 5 registros por grupo (`HAVING COUNT(*) >= 5`). Con los datos de prueba **ningún grupo alcanzó el umbral**, por lo que el resultado fue de 0 filas: todos los grupos fueron suprimidos.

**Qué riesgo de reidentificación permanece**

- Los **cuasi-identificadores** que permanecen son la edad (en rangos) y la región (2 dígitos).
- Cruzar rango de edad, región y `fecha_alta` podría reducir el anonimato.
- Los grupos con pocos registros (por ejemplo, los de 1 persona) aumentan drásticamente el riesgo de reidentificar a quienes pertenecen a ellos.
- Por lo tanto **no es correcto afirmar que el conjunto sea "100 % anónimo"**: siempre existe un riesgo residual, sobre todo si se combina con bases de datos externas.

---

## 19.5 Retención

**Qué criterio se utilizó**

La antigüedad del registro según `fecha_alta`, con un plazo de **5 años**:

```sql
SELECT * FROM clientes
WHERE fecha_alta < CURRENT_DATE - INTERVAL '5 years';
```

Con los datos de prueba la consulta devolvió **0 filas**, porque todos los registros tienen fecha de alta 2026-10-18, es decir, ninguno ha superado el plazo.

**Qué información debería conservarse**

- Los registros con menos de 5 años de antigüedad.
- Los datos que una obligación legal, fiscal o contractual exija conservar por más tiempo.
- Datos ya anonimizados (estadísticas agregadas), que ya no permiten identificar a personas.

**Qué información podría eliminarse**

- Los registros de `clientes` con `fecha_alta` anterior a 5 años, cuando ya no exista finalidad ni obligación legal que justifique conservarlos.
- Sus tokens asociados en `pii.mapa_cliente` y los registros derivados en `analitica.clientes`, para que no queden datos huérfanos ni vínculos que permitan reidentificar.

**Qué controles deberían existir antes de eliminarla**

1. Validar que no haya obligaciones legales, fiscales o contractuales vigentes ni procesos que requieran conservar el dato.
2. Ejecutar primero el `SELECT` y revisar qué registros serían afectados.
3. Obtener la autorización del responsable de los datos.
4. Respaldar la información, con su propio plazo de retención y borrado.
5. Considerar la dependencia entre tablas (`pii.mapa_cliente` referencia a `clientes`), por lo que hay que eliminar en el orden correcto, idealmente dentro de una transacción.
6. Registrar la eliminación en una bitácora (qué, cuándo y quién).
7. Verificar después que los datos ya no existan en tablas derivadas ni en copias.

**Control de acceso durante la retención**

Se verificó que soporte consulta `v_clientes_soporte` y que analítica consulta `analitica.clientes`, pero ninguno accede a `public.clientes` ni a `pii.mapa_cliente` (ambos con permiso denegado).

---

## 19.6 Conclusiones

**¿Qué técnica utilizaría para un área de soporte?**

**Enmascaramiento** mediante una vista con acceso restringido. Soporte necesita reconocer al cliente y verificar datos parciales (últimos dígitos del teléfono o RFC), pero no necesita verlos completos.

**¿Qué técnica utilizaría para un área de analítica?**

**Seudonimización** con separación de esquemas. Analítica necesita analizar comportamiento y poder seguir a cada registro mediante un token, sin conocer nombre, correo, teléfono ni RFC. El mapa de correspondencia queda fuera de su alcance. Si el análisis no requiere seguir individuos, convendría usar datos agregados o anonimizados.

**¿Qué técnica utilizaría para publicar estadísticas?**

**Anonimización**: eliminar identificadores directos, generalizar edad y región, y suprimir los grupos pequeños (umbral mínimo de registros), publicando solo resultados agregados.

**¿Por qué ninguna de estas técnicas sustituye al control de acceso?**

- El enmascaramiento no protege si el usuario puede consultar la tabla original: el dato completo sigue existiendo.
- La seudonimización depende de que la tabla de correspondencia esté protegida; quien acceda a ella puede revertir el proceso.
- La anonimización siempre deja un riesgo residual de reidentificación, y los datos de origen siguen existiendo en la base.
- Las tres técnicas funcionan precisamente gracias a los permisos (`GRANT`, `REVOKE`, esquemas separados, roles con mínimo privilegio). Sin control de acceso, cualquiera podría saltarse la vista, el esquema o el filtro y leer los datos originales.
- Por eso la protección debe ser **por capas**: técnica de protección + control de acceso + retención + auditoría.

---

### Observaciones para mejorar

- La vista `v_clientes_soporte` muestra el `nombre` completo y el `id`; para mayor protección podría enmascararse también el nombre o dejar solo el `id`.
- `analitica.clientes` conserva `edad`, `ciudad` y `codigo_postal` exactos, por lo que los datos siguen siendo cuasi-identificadores; se recomienda generalizarlos antes de compartirlos con terceros.
- Con 5 registros de prueba, el umbral de 5 deja la salida anonimizada vacía; con volúmenes reales los resultados serían más útiles.

---

# 20. Preguntas de reflexión

### 1. ¿Cuál es la diferencia entre enmascaramiento y seudonimización?

El **enmascaramiento** oculta u ofusca parte de un dato para que quien lo consulta no lo vea completo (por ejemplo, `ana.lopez@...` se muestra como `an*****@example.test`). Está pensado para la **visualización**, y el dato original sigue intacto en la tabla. La **seudonimización** reemplaza el identificador de la persona por un token (UUID) y guarda la relación token–persona en una tabla aparte (`pii.mapa_cliente`). Está pensada para **analizar o procesar** datos sin conocer la identidad, y permite volver a vincular el registro con su titular si se tiene el mapa.

### 2. ¿Por qué la seudonimización no significa anonimización?

Porque es **reversible**: quien tenga acceso a la tabla de correspondencia puede relacionar el token con el `id` real del cliente y recuperar su identidad. Además, el conjunto seudonimizado conserva atributos como edad, ciudad y código postal, que pueden permitir reidentificar. Los datos seudonimizados siguen siendo datos personales. Con la anonimización no debería ser posible volver al titular.

### 3. ¿Por qué eliminar `nombre` y `RFC` no garantiza que un conjunto de datos sea anónimo?

Porque quedan los **cuasi-identificadores** (edad, ciudad, código postal, fecha de alta). Combinados entre sí, o cruzados con otras bases de datos externas, pueden señalar a una persona. En nuestro ejemplo, un grupo de una sola persona (por ejemplo, `46+` en la región `82`) es fácilmente identificable aunque no tenga nombre ni RFC. Para anonimizar también hace falta generalizar atributos y suprimir grupos pequeños, y aun así queda un riesgo residual.

### 4. ¿Qué función tienen los cuasi-identificadores?

No identifican a una persona por sí solos, pero **pueden contribuir a identificarla cuando se combinan**. Sirven para segmentar y analizar a la población (por edad o zona), pero son justamente los datos que un atacante usa para la reidentificación por cruce de información. Por eso se generalizan (rangos de edad, 2 dígitos del código postal) y se controla el tamaño mínimo de los grupos.

### 5. ¿Por qué es importante separar el mapa de seudonimización?

Porque el mapa es la **única llave** para revertir los tokens. Si estuviera junto a los datos de analítica, o fuera accesible para los mismos usuarios, la seudonimización perdería su sentido. En la práctica se separó en el esquema `pii`, distinto de `analitica` y de `public`, y se comprobó que el rol `analista` recibe `permiso denegado al esquema pii`. Así, un incidente o un mal uso en el área de analítica no expone las identidades.

### 6. ¿Qué principio de seguridad se aplica al impedir que el analista acceda al mapa?

Se aplican el **principio de mínimo privilegio** (el analista solo recibe lo necesario: `USAGE` en el esquema `analitica` y `SELECT` en `analitica.clientes`) y el de **separación de funciones** (quien analiza los datos no es quien puede revertir la seudonimización).

### 7. ¿Por qué una vista puede ser preferible a entregar acceso directo a la tabla?

Porque la vista **expone solo las columnas y la forma de los datos necesarias** (por ejemplo, correo, teléfono y RFC enmascarados), mientras que el acceso directo a `clientes` daría todos los datos completos. Con una vista, el control se aplica en la propia base de datos y no depende de la aplicación, y los permisos se pueden otorgar y revocar sobre la vista sin tocar la tabla original. En la actividad 2, soporte consulta `v_clientes_soporte` pero recibe `permiso denegado` en `clientes`.

### 8. ¿Qué diferencia existe entre proteger un dato para visualización y protegerlo contra acceso no autorizado?

Proteger para **visualización** (enmascarar, abreviar, mostrar parcialmente) controla *lo que ve* un usuario legítimo en pantalla o en un resultado. Proteger contra **acceso no autorizado** controla *quién puede llegar* al dato original: permisos, roles, esquemas separados, autenticación, auditoría. Ocultar un dato solo en la interfaz no basta: quien se conecte directo a la base vería la información completa. Ambas capas son necesarias y la segunda es la base de la primera.

### 9. ¿Por qué la retención forma parte de la protección de datos?

Porque un dato que ya no existe no puede filtrarse, perderse ni usarse indebidamente. La retención aplica el principio de **limitación del plazo de conservación**: los datos personales se conservan solo mientras exista una finalidad legítima u obligación legal que los justifique. Definir un criterio (como los 5 años según `fecha_alta`) y controlar quién puede acceder a los datos mientras se conservan reduce la superficie de riesgo.

### 10. ¿Qué riesgo existe si una organización conserva información indefinidamente?

- **Mayor impacto de una brecha de seguridad**: se expondrían datos de personas que ya ni siquiera son clientes.
- **Incumplimiento normativo**, al conservar datos más allá de la finalidad o del plazo permitido.
- **Datos desactualizados o inexactos**, que pueden llevar a decisiones erróneas.
- **Más riesgo de reidentificación y de uso para fines distintos** al original.
- **Mayores costos** de almacenamiento, respaldo y administración de accesos.
