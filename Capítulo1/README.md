# Práctica 1. Manejo de transacciones
## Objetivo
Al finalizar la práctica, serás capaz de:
- Reforzar los conocimientos fundamentales del lenguaje SQL mediante la conexión a una base de datos PostgreSQL, la creación de estructuras básicas (tablas) y la ejecución de consultas simples.

## Duración aproximada
- 60 minutos.

## Objetivo visual
En esta práctica se verá el manejo de transacciones en PostgreSQL. 

![diagrama1](../images/transaccion.jpg)
## Instrucciones

### Tarea 1. Transferencia bancaria con `rollback` controlado
En el siguiente ejercicio, practicarás diferentes escenarios de manejo de transacciones comprobando su funcionamiento.
```
Desde el escritorio de Linux abre una terminal y conéctate a PosgreSQL con el comando:
$ sudo -i -u posgres psql
```

**Paso 1.** Crea una tabla y datos de prueba `psql -U postgres`.

```sql
CREATE TABLE cuentas (
  id SERIAL PRIMARY KEY,
  nombre TEXT,
  saldo NUMERIC
);
```

```sql
INSERT INTO cuentas (nombre, saldo) VALUES ('Juan', 1000), ('Ana', 1000);
```

**Paso 2.** Simula una transferencia exitosa.
```sql
BEGIN;
UPDATE cuentas SET saldo = saldo - 200 WHERE nombre = 'Juan';
UPDATE cuentas SET saldo = saldo + 200 WHERE nombre = 'Ana';
COMMIT;
-- Verifica que la transferencia fue exitosa.
SELECT * FROM cuentas;
```

**Paso 3.** Simula una falla y `rollback`.
```sql
BEGIN;
UPDATE cuentas SET saldo = saldo - 500 WHERE nombre = 'Juan';
-- ERROR: en la siguiente sentencia, la cuenta de Carlos no existe
UPDATE cuentas SET saldo = saldo + 500 WHERE nombre = 'Carlos';
ROLLBACK;
```

**Paso 4.** Verifica que nada cambió.
```sql
SELECT * FROM cuentas;
```

### Tarea 2. Simulación de bloqueo concurrente
El escenario es una transferencia de dinero, donde es crucial evitar inconsistencias si dos transacciones intentan modificar el saldo de la misma cuenta al mismo tiempo.
Usaremos dos sesiones de `psql` para simular una `Sesión A` y una `Sesión B`.
Abra dos terminales en el escritorio de Linux, y antes de iniciar el laboratorio, lleva a cabo los siguientes pasos de preparación.

#### Preparación (En una terminal en la sesión A, una sola vez)

**Paso 1.** Primero, limpia la tabla `cuentas` y agrega los datos:

```sql
TRUNCATE TABLE cuentas;
INSERT INTO cuentas (nombre, saldo) VALUES ('Juan', 1000), ('Ana', 1000);
```

Puedes verificar los saldos iniciales con:

```sql
SELECT * FROM cuentas;
```

Salida esperada:
```
   id | nombre | saldo
   ----+--------+-------
    3 | Juan   |  1000
    4 | Ana    |  1000
   (2 rows)
```

**Paso 2.** Escenario de bloqueo con `SELECT ... FOR UPDATE`.

En este ejemplo, la `Sesión A` intentará retirar dinero de la cuenta de Juan y la `Sesión B` intentará hacer lo mismo concurrentemente. Verás cómo el bloqueo de fila evita un problema.

`Sesión A` (Terminal 1)
inicia una transacción y bloquea la cuenta de Juan para actualizarla.

```sql
BEGIN;
--Selecciona el saldo de Juan y bloquea la fila
SELECT saldo FROM cuentas WHERE nombre = 'Juan' FOR UPDATE;
```
**Salida (Sesión A)**
 
```
saldo
-------
  1000
(1 row)
```

La `Sesión A` ahora tiene un bloqueo exclusivo sobre la fila de `'Juan'`. Esto significa que cualquier otra transacción que intente modificar o bloquear esta misma fila esperará hasta que la `Sesión A` libere el bloqueo.

**Paso 3.** Ahora, simula una operación de retiro en esta misma sesión.

```sql
UPDATE cuentas SET saldo = saldo - 200 WHERE nombre = 'Juan';
```

Esta actualización se realiza dentro de la transacción de la `Sesión A`. El saldo de Juan ahora es `800` dentro de esta transacción, pero los cambios aún no son permanentes en la base de datos y la fila sigue bloqueada por la `Sesión A`.
 
`Sesión B` (Terminal 2)
intenta leer la cuenta de Juan (sin bloqueo).

```sql
SELECT saldo FROM cuentas WHERE nombre = 'Juan';
```

**Salida (Sesión B)**
```
saldo
-------
 1000
(1 row)
```

La `Sesión B` puede leer la fila. Observa que percibe el saldo como `1000`, no `800`, porque los cambios de la `Sesión A` todavía no se han confirmado `(COMMIT)`.

**Paso 4.** Ahora, intenta realizar un retiro de la misma cuenta (lo que intentará adquirir un bloqueo `FOR UPDATE`).

```sql
-- Desde la Session B:
UPDATE cuentas SET saldo = saldo - 150 WHERE nombre = 'Juan';
```

Salida (`Sesión B`): verás que este comando se queda esperando (o "colgado") indefinidamente.

La `Sesión B` está bloqueada porque la `Sesión A` tiene un bloqueo `FOR UPDATE` sobre esa fila. La `Sesión B` esperará hasta que `Sesión A` libere su bloqueo.

Vuelve a la `Sesión A` (Terminal 1) y confirma la transacción.

```sql
COMMIT;
```

Al ejecutar `COMMIT`, la `Sesión A` guarda sus cambios permanentemente (el saldo de Juan ahora es `800`) y, crucialmente, libera el bloqueo sobre la fila de 'Juan'.

Vuelve a la `Sesión B` (Terminal 2) y observa la salida: tan pronto como la `Sesión A` ejecuta `COMMIT`, el `UPDATE` de la `Sesión B` que estaba esperando finaliza su ejecución.

Salida (`Sesión B`):
`UPDATE 1`

El `UPDATE` de la `Sesión B` ahora se ejecuta correctamente. El `UPDATE` aplicó el cambio de `-150` al saldo actual que `Sesión B` leyó después de que `Sesión A` hiciera `COMMIT (800)`. Por lo tanto, el saldo final de Juan es `800 - 150 = 650`.


**Paso 5.** Verificación final (en cualquier sesión, después de que ambas transacciones hayan terminado).

```sql
SELECT * FROM cuentas WHERE nombre = 'Juan';
```
```Salida esperada:
 id | nombre | saldo
 ----+--------+-------
  1 | Juan   |   650
 (1 row)
```
 

## Resultado esperado
-	El comando `SELECT ... FOR UPDATE` adquirió un bloqueo exclusivo sobre la fila de la cuenta de Juan en la `Sesión A`.
-	Esto impidió que la `UPDATE` concurrente de la `Sesión B` se ejecutara inmediatamente; la `Sesión B` tiene que esperar.
-	Una vez que la `Sesión A` hizo `COMMIT`, el bloqueo se liberó, permitiendo que la `UPDATE` de la `Sesión B` procediera y aplicara sus cambios.
-	Gracias al bloqueo, evitamos que ambas transacciones intentaran modificar el mismo saldo basándose en un valor inicial obsoleto, garantizando que todas las operaciones se aplicaran secuencialmente.

### Tarea 3. Comparación de niveles de aislamiento

**Paso 1.** Abre dos terminales `psql`.

Terminal A:
```sql
BEGIN;
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
SELECT COUNT(*) FROM cuentas;
```
Mantén abierta la ventana.

Terminal B:
```sql
INSERT INTO cuentas (nombre, saldo) VALUES ('Luis', 100);
```

De nuevo, en `Terminal A`:
```sql
SELECT COUNT(*) FROM cuentas; -- verás el nuevo registro con READ COMMITTED
COMMIT;
```

Repite el mismo flujo usando `REPEATABLE READ` y compara los resultados.

### Tarea 4. Simulación de Deadlock

**Paso 1.** Crea una tabla para el experimento.

```sql
CREATE TABLE recursos (
  id SERIAL PRIMARY KEY,
  nombre TEXT
);
INSERT INTO recursos (nombre) VALUES ('A'), ('B');
```

**Paso 2.** Simula desde dos terminales.

`Terminal A`:
```sql
BEGIN;
UPDATE recursos SET nombre = 'A1' WHERE id = 1;
--Espera aquí sin hacer COMMIT
```
`Terminal B`:
```sql
BEGIN;
UPDATE recursos SET nombre = 'B1' WHERE id = 2;
```
De nuevo en `Terminal A`:
```sql
UPDATE recursos SET nombre = 'A2' WHERE id = 2; -- queda esperando
```
Luego en `Terminal B`:
```sql
UPDATE recursos SET nombre = 'B2' WHERE id = 1; -- PostgreSQL detectará el deadlock
```
## Resultado esperado
En la Terminal B, PostgreSQL detectará el deadlock y enviará un mensaje de error. Cuando un deadlock ocurre, el sistema lo detecta y termina una de las transacciones involucradas para resolver el bloqueo. Esto significa que una de las transacciones fallará y será revertida (rollback), liberando los bloqueos que tenía y permitiendo que la otra continúe su ejecución. El sistema elige a una de las transacciones involucradas como la "víctima", la elección se basa en varios factores, pero generalmente se selecciona la transacción que ha realizado menos trabajo. Esto se hace para minimizar la pérdida de progreso.
