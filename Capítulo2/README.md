# Práctica 2. Índices y tipos de datos en PostgreSQL
## Objetivos
Al finalizar la práctica, serás capaz de:
- Utilizar de manera efectiva los diferentes tipos de datos que existen en PostgreSQL.
- Aprender a indexar diferentes tipos de datos y cómo utilizar la exclusión de `constraints`.

## Duración aproximada
- 120 minutos.

## Objetivo visual
En esta práctica se verá la creación y manejo de índices en PostgreSQL. El siguiente diagrama muestra la construcción de los índices B-Tree en PostgreSQL.

![diagrama2](../images/btree.png)
## Instrucciones
### Tarea 1. Criticas de Peliculas con `TEXT` y `VARCHAR`

**Paso 1.** Crea la tabla `comentarios` e inserta los datos.

```sql
CREATE TABLE comentarios (
    id SERIAL PRIMARY KEY,
    autor VARCHAR(100),
    critica TEXT,
    calificacion INT CHECK (calificacion BETWEEN 1 AND 5),
    fecha TIMESTAMP DEFAULT NOW()
);
```
```sql
INSERT INTO comentarios (autor, critica, calificacion)
SELECT
    'Usuario' || i,
    repeat('Muy buen producto. ', (random()*3 + 1)::int),
    (random()*4 + 1)::int
FROM generate_series(1, 1000) i;
```

**Paso 2.** Crea los índices.

```sql
CREATE INDEX idx_autor ON comentarios(autor);
CREATE INDEX idx_comentarios_positivos ON comentarios(critica)
WHERE calificacion > 4;
```

**Paso 3.** Verifica los resultados.

Esta consulta aprovecha el índice parcial si `calificacion > 4`,
 reduciendo el número de filas escaneadas. Se espera un `Sequential Scan` si no se usa el índice.
`EXPLAIN ANALYZE SELECT * FROM comentarios WHERE calificacion > 4 AND critica IS NOT NULL;`


### Tarea 2. Manejo de encuestas con `ARRAY` y `ENUM`

**Paso 1.** Crea el tipo `nivel_satisfaccion`, la tabla `encuestas`, inserta los datos y crea el índice.

```sql
CREATE TYPE nivel_satisfaccion AS ENUM ('muy_bajo', 'bajo', 'medio', 'alto', 'muy_alto');
```
```sql
CREATE TABLE encuestas (
    id SERIAL PRIMARY KEY,
    respuestas TEXT,
    satisfaccion nivel_satisfaccion,
    fecha DATE DEFAULT CURRENT_DATE
);
```
```sql
INSERT INTO encuestas (respuestas, satisfaccion)
SELECT
    (ARRAY['si', 'no', 'tal vez'])[:(random()*3+1)::int],
    (ARRAY['muy_bajo', 'bajo', 'medio', 'alto', 'muy_alto'])[(random()*4+1)::int]::nivel_satisfaccion
FROM generate_series(1, 500);
```
```sql
CREATE INDEX idx_respuestas_satisfaccion ON encuestas (respuestas, satisfaccion);
```

 Esta consulta puede beneficiarse del índice en respuestas mas satisfacción,
 si el optimizador detecta que es más eficiente que un escaneo secuencial.
 Verifica con EXPLAIN algunas búsquedas para ver si se esta usando el indice o no.
`EXPLAIN ANALYZE SELECT * FROM encuestas WHERE satisfaccion = 'muy_alto';`

## Tarea 3. Manejo de preferencias de usuario usando el tipo `JSONB` y el índice `GIN`.

**Paso 1.** Crea los elementos necesarios para el ejercicio.

```sql	
CREATE TABLE usuarios (
    id SERIAL PRIMARY KEY,
    nombre VARCHAR(100),
    preferencias JSONB,
    creado_en TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```
**Paso 2.** Inserta los datos en la tabla.

```sql
INSERT INTO usuarios (nombre, preferencias)
SELECT
    'usuario_' || i,
    jsonb_build_object(
        'notificaciones', CASE WHEN i % 2 = 0 THEN 'email' ELSE 'push' END,
        'tema', CASE WHEN i % 3 = 0 THEN 'oscuro' ELSE 'claro' END
    )
FROM generate_series(1, 1000) i;
```

**Paso 3.** Crea los índices.

```sql

CREATE INDEX idx_tema ON usuarios ((preferencias->>'tema'));
```

**Paso 4.** Usa `Explain` con `Select` para observar el comportamiento.

```sql
EXPLAIN ANALYZE SELECT * FROM usuarios WHERE preferencias @> '{"tema": "oscuro"}';
```

Esta consulta utiliza el índice para realizar búsquedas eficientes dentro del campo `JSONB`. Es mucho más rápido que escanear fila por fila.

**Explicación de los pasos anteriores y del código**

1.  `CREATE INDEX idx_tema ON usuarios ((preferencias->>'tema'));`

Crea un índice basado en una expresión, no directamente sobre una columna, sino sobre el resultado de una operación sobre una columna.
En este caso, `preferencias` es un campo de tipo `JSONB`.

`preferencias->>'tema'` extrae el valor del campo "tema" como texto (por ejemplo, "oscuro" o "claro").

El índice se crea sobre esa expresión para acelerar búsquedas por ese campo específico del `JSON`. PostgreSQL no puede indexar directamente elementos internos de un `JSONB` con `BTREE`. 

Con esta técnica, puedes usar el valor del campo "tema" como si fuera una columna común y obtener beneficios de rendimiento.

2.  `SELECT * FROM usuarios WHERE preferencias @> '{"tema": "oscuro"}';`

Busca todos los registros de la tabla `usuarios` donde el campo preferencias contiene el par clave-valor `"tema": "oscuro"`.

El operador `@>` es el operador de contención de `JSONB` en PostgreSQL.

Esto significa que el campo `preferencias` debe tener al menos esa clave y valor. Puede tener más elementos, pero `"tema": "oscuro"` debe existir dentro del `JSON`.

En este ejemplo, este `JSONB` **sí** califica:
```
{
  "tema": "oscuro",
  "notificaciones": "push"
}
```

Pero este **no**:
```
{
  "tema": "claro"
}
```
Se observa la compatibilidad con índices, lo que permite búsquedas rápidas, incluso dentro de estructuras complejas como `JSON`.

### Tarea 4. Manejo de una tabla de sensores con índice `BRIN` y `BTREE`

**Paso 1.** Crea los elementos necesarios para el ejercicio.

```sql
CREATE TABLE sensores (
    id SERIAL PRIMARY KEY,
    zona TEXT,
    temperatura NUMERIC(5,2),
    fecha TIMESTAMP
);
```

**Paso 2.** Inserta los datos de prueba.

```sql
INSERT INTO sensores (zona, temperatura, fecha)
SELECT
    'zona_' || (i % 10),
    ROUND( (20 + random() * 15)::numeric, 2),
    now() - INTERVAL '1 minute' * i
FROM generate_series(1, 1000000) i;
```

**Paso 3.** Crea los índices `BRIN` y `BTREE`.

```sql
CREATE INDEX idx_brin_fecha ON sensores USING brin (fecha);
CREATE INDEX idx_btree_fecha ON sensores USING btree (fecha);
```

Evaluación del espacio ocupado por los índices `BTREE` y `BRIN`:

```sql
SELECT indexrelid::regclass AS index_name, pg_size_pretty(pg_relation_size(indexrelid)) AS size
FROM pg_index
WHERE indrelid = 'sensores'::regclass;
```

Esta consulta debería aprovechar el índice `BRIN` si las fechas están ordenadas.
 El objetivo es reducir la cantidad de bloques leídos del disco.

```sql
EXPLAIN ANALYZE
SELECT * FROM sensores
WHERE fecha BETWEEN now() - INTERVAL '2 days' AND now() - INTERVAL '1 day';
```

**Paso 4.** Revisa el espacio ocupado por los índices.

```sql
SELECT indexrelid::regclass AS index_name, 
       pg_size_pretty(pg_relation_size(indexrelid)) AS size
FROM pg_index
WHERE indrelid = 'sensores'::regclass;
```

**Comentarios**

- Al ejecutar las sentencias anteriores, se mostrará el tamaño en disco de todos los índices asociados a la tabla `sensores`.



**Preguntas adicionales del ejercicio**

¿El índice `BRIN` realmente ocupa menos espacio que `BTREE`?
¿Qué tan costoso en disco sería mantener varios índices?

## Resultado esperado
El query ejecutado sobre la tabla `pg_index` mostrará las siguientes columnas de resultados.

- `pg_index`: catálogo del sistema que almacena información sobre índices.

- `indrelid = 'sensores'::regclass`: filtra solo los índices de la tabla `sensores`.

- `indexrelid::regclass`: convierte el ID del índice a nombre legible (como `idx_btree_fecha`).

- `pg_relation_size(indexrelid)`: obtiene el tamaño en bytes del índice.

- `pg_size_pretty(...)`: convierte esos bytes a un formato legible como 12 MB, 180 kB, etcétera.
