# Práctica 4. Respaldos
## Objetivo
Al finalizar la práctica, serás capaz de:
- Realizar respaldos físicos en PostgreSQL y ajustar de manera avanzada Autovacuum para mantener el rendimiento en bases de datos transaccionales.

## Duración aproximada
- 120 minutos.

## Objetivo visual
En estas prácticas se verá cómo realizar respaldos y restauraciones físicas de PostgreSQL, así como la herramienta Autovacuum.

![diagrama1](../images/respaldos.jpg)

## Instrucciones

### Tarea 1. Hacer un respaldo del clúster PostgreSQL

**Paso 1.** Crea el usuario "replicador" con el rol de réplica.
Debes conectarte a tu instancia de PostgreSQL como un superusuario (por ejemplo, `postgres`) y ejecuta el siguiente comando `SQL:
CREATE USER replicador WITH REPLICATION ENCRYPTED PASSWORD 'tu_contraseña_segura';`

**Paso 2.** Configura `pg_hba.conf` en el servidor primario.
-	El archivo `pg_hba.conf` controla la autenticación de clientes de PostgreSQL. Necesitas agregar una entrada que permita al usuario replicador conectarse desde la máquina donde ejecutarás `pg_basebackup`.
-	Localiza el archivo `pg_hba.conf`. Este archivo se encuentra típicamente en el directorio de datos de tu instalación de PostgreSQL; por ejemplo, `/var/lib/postgresql/16/main/pg_hba.conf` o similar, dependiendo de tu versión y sistema operativo (SO).
-	Edita el archivo `pg_hba.conf`. Abre el archivo con un editor de texto (requerirás permisos de superusuario, como `sudo`).
-	Agrega la siguiente línea (o similar):

	`host    replication     replicador      0.0.0.0/0               md5`

En donde:  
- `host`: indica que la conexión es a través de `TCP/IP`.
- `replication`: es un `pseudo-database` especial que se usa para conexiones de replicación.
- `replicador`: es el nombre del usuario que creaste.
- `0.0.0.0/0`: permite la conexión desde cualquier dirección IP. Para un entorno de producción, es **altamente recomendable** que cambies esto a la dirección IP específica de la máquina desde la que ejecutarás `pg_basebackup` (ejemplo: `192.168.1.100/32` si la IP es `192.168.1.100`).
- `MD5`: especifica que se utilizará autenticación con contraseña `MD5`.

**Paso 3.** Recarga la configuración de PostgreSQL.
-	Después de modificar `pg_hba.conf`, necesitas recargar la configuración de PostgreSQL para que los cambios surtan efecto. Puedes hacerlo de una de las siguientes maneras.
	Usando `SQL` (recomendado si puedes conectarte):
	```sql
	SELECT pg_reload_conf();
	```
-	 la línea de comandos (usando `systemd` o `init.d`):
	`sudo systemctl reload postgresql`
-	`O`, dependiendo de tu versión y SO, podría ser:
	`sudo /etc/init.d/postgresql reload`

**Paso 4.** Verifica y configura el archivo `/etc/postgresql/16/main/postgresql.conf`.
-	Crea el directorio `/var/lib/postgresql/archive` desde el usuario `posgresql`.
	```
	mkdir /var/lib/postgresql/archive
	```
-	Cambia en `postgresql.conf` a la ruta válida en tu sistema si deseas archivar los Write-Ahead Log (WAL) de rotación usando la variable `archive_command`.
-	Asegúrate de que el `wal_level`, `archive_mode` y `archive_command` estén configurados para permitir respaldos.
	```
	wal_level = replica
	archive_mode = on
	max_wal_senders = 2
	archive_command = cp %p /var/lib/postgresql/archive/%f
	```

-	El valor `cp %p /var/lib/postgresql/archive/%f` indica que hay que copiar cada archivo WAL generado por PostgreSQL al directorio `/var/lib/postgresql/archive/`.
	- `%p`: ruta completa del archivo WAL original.
	- `%f`: nombre del archivo WAL.
-	La variable `max_wal_senders` define cuántos procesos de envío de WAL pueden ejecutarse simultáneamente. 
-	Después de cambiar estos parámetros, reinicia PostgreSQL:
	`sudo systemctl restart postgresql`.

**Paso 4.** Ejecuta `pg_basebackup`.
-	Crea el directorio donde se harán los respaldos desde el `usuario postgre`:
	`mdkir /var/lib/postgresql/respaldos`.
-	Ahora, desde la máquina donde deseas almacenar el respaldo (que puede ser el mismo servidor o uno diferente, siempre que la red lo permita y `pg_hba.conf` esté configurado correctamente), puedes ejecutar desde la línea de comandos del shell tu comando `pg_basebackup`:
`pg_basebackup -h tu_ip_servidor_primario -D /respaldos/pg -Ft -z -P -U usuario_replicador`.

Ejemplo:
`pg_basebackup -h localhost -D /var/lib/postgresql/respaldos -Ft -z -P -U replicador`.

En donde:
- `localhost`: es la máquina local.
- `h localhost`: indica a cuál `host` conectarse.
- `D /var/lib/postgresql/respaldos`: directorio de destino donde se almacenará el respaldo.
- `F` `t`: formato del respaldo, `t` significa `tarball` (archivo `.tar`).
- `z`: comprime el respaldo generado (`gzip`). El archivo final será `.tar` `.gz`.
- `P`: muestra una barra de progreso durante la copia.
- `U replicador`: usuario de PostgreSQL que ejecuta la copia. El usuario de la base de datos debe tener permisos de replicación.

**Archivos generados del respaldo con `pg_basebackup`**
- `base.tar.gz`: contiene una copia completa y consistente del directorio de datos de tu base de datos (excluyendo los archivos WAL activos en el momento del backup, que están en `pg_wal.tar.gz`).
- `pg_wal.tar.gz`: contiene los archivos del WAL necesarios para que la base de datos se recupere y alcance un estado consistente al iniciar después de la restauración.
- `backup_manifest`: contiene metadatos sobre el backup, la lista de archivos incluidos, las sumas de verificación y la información del punto de control (`checkpoint`) del backup, no se extrae directamente en el directorio de datos para el inicio del servidor.

### Tarea 2. Restaurar el clúster de PostgreSQL

Del ejercicio anterior, restaura todo el clúster de PostgreSQL. Después verifica que las bases de datos y las tablas junto con sus datos existen y que los datos originales se mantengan.

**Paso 1.** Detén PostgreSQL:
`sudo systemctl stop postgresql`

**Paso 2.** Elimina o renombra el `$PGDATA` actual:
`mv  /var/lib/postgresql/16/main   /var/lib/postgresql/16/main_old`

**Paso 3.** Crea el directorio `$PGDATA` desde el usuario `postgre`:
`mkdir  /var/lib/postgresql/16/main` 

**Paso 4.** Copia desde el directorio de respaldos, el respaldo físico.

`tar -xzf base.tar.gz -C /var/lib/postgresql/16/main`

**Paso 5.** Restaura los archivos WAL.

`tar -xzf pg_wal.tar.gz -C /var/lib/postgresql/16/main/pg_wal`

- Asegúrate de tener el `restore_command` bien definido.
- Opcional: si hay `PITR` colocar el archivo `recovery.signal` en el nuevo `$PGDATA`.

**Paso 6** (muy importante). Actualiza el propietario y los permisos del directorio `main`.
```
sudo chown postgres:postgres /var/lib/postgresql/16/main
sudo chmod 700 /var/lib/postgresql/16/main
```

**Paso 7.** Inicia PostgreSQL:
	`sudo systemctl start postgresql`

**Paso 8.** Verifica `logs` y estado:
	`tail -f /var/log/postgresql/postgresql-16-main.log`

### Tarea 3. Uso de Autovacuum: configuración y monitoreo en PostgreSQL
Comprenderás el funcionamiento del proceso Autovacuum en PostgreSQL, la configuración de sus parámetros y el monitoreo de su actividad.

**Requisitos**
- PostgreSQL instalado (versión 9.6 o superior).
- Acceso a una base de datos con permisos de superusuario o suficientes privilegios.
- Herramienta `psql` o `pgAdmin` para conectarse a la base de datos.

**Paso 1.** Verificación del estado de Autovacuum.
1.	Conéctate a la base de datos PostgreSQL usando `psql`.
```
psql -U postgres -d nombre_base_datos
```

2.	Verifica si Autovacuum está activado.

```sql
SHOW autovacuum;
```

3.	Consulta los parámetros actuales de Autovacuum.

```sql
SELECT name, setting, short_desc FROM pg_settings 
WHERE name LIKE 'autovacuum%' OR name LIKE 'vacuum%';
```

**Paso 2.** Crea una tabla de prueba y generación de actividad.

1.	Crea una tabla de prueba.

```sql
CREATE TABLE laboratorio_autovacuum (
    id SERIAL PRIMARY KEY,
    dato TEXT,
    fecha TIMESTAMP DEFAULT NOW()
);
```

2.	Genera datos y actualizaciones para simular actividad.

```sql
-- Inserta 10,000 registros
INSERT INTO laboratorio_autovacuum (dato)
SELECT md5(random()::text) FROM generate_series(1, 10000);
```

```sql
-- Actualiza todos los registros varias veces.
UPDATE laboratorio_autovacuum SET dato = md5(random()::text);
UPDATE laboratorio_autovacuum SET dato = md5(random()::text);
UPDATE laboratorio_autovacuum SET dato = md5(random()::text);
```

**Paso 3.** Monitorea las estadísticas.

1.	Verifica las estadísticas de la tabla.

```sql
SELECT relname, n_live_tup, n_dead_tup, last_autovacuum, last_autoanalyze 
FROM pg_stat_user_tables 
WHERE relname = 'laboratorio_autovacuum';
```

2.	Monitorea los procesos de Autovacuum en ejecución.

```sql
SELECT datname, usename, query, state 
FROM pg_stat_activity 
WHERE query LIKE '%autovacuum%' OR query LIKE '%VACUUM%';
```

**Paso 4.** Configura los parámetros.

1.	Modifica los parámetros de Autovacuum para la tabla de prueba.

```sql
ALTER TABLE laboratorio_autovacuum SET (autovacuum_vacuum_scale_factor = 0.01);
ALTER TABLE laboratorio_autovacuum SET (autovacuum_vacuum_threshold = 500);
ALTER TABLE laboratorio_autovacuum SET (autovacuum_analyze_scale_factor = 0.005);
```

2.	Verifica la configuración específica de la tabla.

```sql
SELECT relname, reloptions 
FROM pg_class 
WHERE relname = 'laboratorio_autovacuum';
```

**Paso 5.** Ejecuta `VACUUM` y `ANALYZE` manualmente.

1.	Ejecuta `VACUUM` manualmente y observa la diferencia.

```sql
VACUUM (VERBOSE) laboratorio_autovacuum;
```

2.	Ejecuta `ANALYZE` manualmente.

```sql
ANALYZE VERBOSE laboratorio_autovacuum;
```

**Paso 6.** Analiza los resultados.

1.	Vuelve a consultar las estadísticas después de las operaciones.

```sql
SELECT relname, n_live_tup, n_dead_tup, last_autovacuum, last_autoanalyze 
FROM pg_stat_user_tables 
WHERE relname = 'laboratorio_autovacuum';
```

2.	Compara los resultados antes y después de las operaciones.

**Preguntas del ejercicio**
1.	¿Cuántas tuplas muertas se generaron antes de que Autovacuum actuara?
2.	¿Cuánto tiempo tardó Autovacuum en ejecutarse automáticamente después de las actualizaciones?
3.	¿Cómo afectó la configuración personalizada al comportamiento de Autovacuum?
4.	¿Qué diferencias observó entre el `VACUUM` manual y el automático?
5.	¿Por qué es importante el proceso `Autovacuum` en PostgreSQL?

**Extensión opcional**
1.	Desactiva `Autovacuum` para la tabla de prueba y observa el comportamiento.

```sql
ALTER TABLE laboratorio_autovacuum SET (autovacuum_enabled = false);
```

2.	Genera más actualizaciones y observa el crecimiento de tuplas muertas sin `Autovacuum`.
3.	Vuelve a activar `Autovacuum` y observa cómo se recupera la situación.


## Resultado esperado
Para realizar un respaldo físico, ejecuta:

`pg_basebackup -h localhost -D /var/lib/postgresql/respaldos -Ft -z -P -U replicador.`

Lo cual genera los siguientes archivos.

- `base.tar.gz`: contiene una copia completa y consistente del directorio de datos de tu base de datos (excluyendo los archivos WAL activos en el momento del backup, que están en `pg_wal.tar.gz`).
- `pg_wal.tar.gz`: contiene los archivos del WAL necesarios para que la base de datos se recupere y alcance un estado consistente al iniciar después de la restauración.
- `backup_manifest`: contiene metadatos sobre el backup, la lista de archivos incluidos, las sumas de verificación y la información del punto de control (`checkpoint`) del backup, no se extrae directamente en el directorio de datos para el inicio del servidor.

Para restaurar el "clúster", ejecuta:

`tar -xzf base.tar.gz -C /var/lib/postgresql/16/main`

Restaura los archivos WAL.

`tar -xzf pg_wal.tar.gz -C /var/lib/postgresql/16/main/pg_wal`

- Asegúrate de tener el `restore_command` bien definido.
- Opcional: si hay `PITR` colocar el archivo `recovery.signal` en el nuevo `$PGDATA`.

Actualiza el propietario y los permisos del directorio `main`.

