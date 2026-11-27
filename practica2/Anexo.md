# Anexo: Investigación de Términos y Herramientas en MariaDB

## **Tecnológico Nacional de México**  
### **Instituto Tecnológico de Milpa Alta 2**

* **Carrera:** Ingeniería en Sistemas Computacionales
* **Materia:** Administración de Bases de Datos
* **Alumno:** Basurto García Carlos Yael
* **Número de Control:** 241190029
* **Docente:** Ing. Diego Alberto Bollas Paredes
* **Semestre:** Quinto
* **Fecha:** 26/11/26

---

### `mariadb-dump` (antes `mysqldump`)
Es la utilidad oficial de MariaDB para realizar **respaldos y exportaciones lógicas**. Genera un archivo de texto plano en formato SQL que contiene las instrucciones DDL (`CREATE TABLE`, `CREATE VIEW`, etc.) y DML (`INSERT INTO`) necesarias para reconstruir una o varias bases de datos. A partir de MariaDB 10.5 se introdujo el nombre `mariadb-dump` como reemplazo moderno de `mysqldump`.

### `mariadb-backup`
Herramienta oficial para realizar **respaldos físicos en caliente** (copiando directamente los archivos de datos del disco sin detener el servidor). A diferencia de `mariadb-dump`, es mucho más rápida para bases de datos de gran volumen, aunque requiere compatibilidad directa entre la versión y el motor de almacenamiento del origen y el destino.

### `mariadb-import` y `LOAD DATA INFILE`
* **`LOAD DATA INFILE`:** Sentencia SQL optimizada para importar grandes volúmenes de datos desde archivos delimitados (como CSV) directamente a una tabla a alta velocidad.
* **`mariadb-import`:** Utilidad de consola que actúa como interfaz directa para `LOAD DATA INFILE`, permitiendo cargar archivos CSV cuyo nombre coincide con el de la tabla destino.

### `mariadb-upgrade`
Herramienta administrativa utilizada tras realizar actualizaciones de versión *in-place* en MariaDB. Examina todas las tablas y bases de datos del sistema (como el esquema `mysql`) para detectar incompatibilidades y actualizar sus estructuras internas a los requerimientos de la nueva versión del servidor.

### `scp` (Secure Copy Protocol) y Tuberías (`|`)
* **`scp`:** Protocolo y comando de sistemas Unix/Linux que permite transferir archivos de forma cifrada entre dos servidores remotos utilizando SSH. Es el método estándar para mover volcados `.sql` entre servidores.
* **Tubería (`|`):** Operador del intérprete de comandos que redirige la salida estándar de un programa directamente hacia la entrada estándar de otro, permitiendo migrar una base de datos al vuelo sin almacenar un archivo `.sql` intermedio en disco.

### `--single-transaction`
Parámetro de `mariadb-dump` que ejecuta todo el volcado dentro de una sola transacción con nivel de aislamiento *REPEATABLE READ*. Aprovecha el control de concurrencia multiversión (MVCC) del motor **InnoDB** para garantizar un respaldo consistente sin bloquear el acceso de otros usuarios ni aplicaciones.

### `--routines`, `--triggers` y `--events`
Banderas de `mariadb-dump` que aseguran la exportación de la lógica almacenada en la base de datos:
* **`--routines`:** Incluye procedimientos almacenados (*stored procedures*) y funciones.
* **`--triggers`:** Incluye los disparadores asociados a las tablas (habilitado por defecto, pero se especifica por claridad).
* **`--events`:** Incluye las tareas programadas del *Event Scheduler* de MariaDB.

### `CHECKSUM TABLE`
Sentencia SQL que calcula un valor entero de suma de comprobación (tipo CRC32) a partir de las filas actuales de una o más tablas. Es fundamental en auditorías de migración para validar que el contenido migrado es idéntico al original sin tener que comparar registro por registro manualmente.

### `SHOW CREATE TABLE` y modificador `\G`
* **`SHOW CREATE TABLE`:** Devuelve la sentencia DDL exacta que define una tabla, mostrando columnas, tipos de datos, llaves primarias y foráneas, índices, motor de almacenamiento (`InnoDB`) y codificación de caracteres.
* **`\G`:** Terminador de consulta del cliente `mariadb` que muestra los resultados en formato vertical (una columna por línea) en lugar de tabular, facilitando la lectura de salidas extensas.

### `FLUSH TABLES WITH READ LOCK`
Sentencia administrativa que cierra todas las tablas abiertas y aplica un bloqueo global de solo lectura en el servidor. Se utiliza durante la ventana de corte (*cutover*) de una migración para detener cualquier escritura en el origen y garantizar que no se pierdan transacciones de último minuto antes del cambio a producción.

### `ANALYZE TABLE` y `OPTIMIZE TABLE`
* **`ANALYZE TABLE`:** Recalcula y actualiza las estadísticas de distribución de claves e índices de una tabla para que el optimizador de consultas de MariaDB elija planes de ejecución eficientes tras una migración.
* **`OPTIMIZE TABLE`:** Reorganiza el almacenamiento físico de los datos e índices para recuperar espacio no utilizado y reducir la fragmentación.

### Migración homogénea vs. heterogénea
* **Homogénea:** El origen y el destino utilizan el mismo gestor de bases de datos (por ejemplo, de MariaDB a MariaDB). Presenta menor complejidad y mayor compatibilidad de tipos de datos y sintaxis.
* **Heterogénea:** Implica cambiar entre distintos motores de bases de datos (por ejemplo, de PostgreSQL, Oracle o SQL Server hacia MariaDB). Requiere conversión de tipos de datos, adaptación de esquemas y reescritura de lógica almacenada.

### Estrategias de migración y Plan de Rollback
* **Big Bang:** Se detiene el sistema en una ventana de mantenimiento, se migra toda la base de datos en un solo bloque y se redirige la aplicación al destino.
* **Con replicación:** Se configura el destino como réplica en tiempo real del origen; cuando ambos están sincronizados, se realiza un corte rápido con mínimo tiempo de inactividad.
* **Por fases:** Se trasladan módulos o tablas de forma gradual.
* **Plan de Rollback (Reversión):** Conjunto de pasos predefinidos para regresar la operación al servidor original en caso de que la migración presente fallas críticas durante el corte.

### Integridad referencial (`FOREIGN KEY` y `ERROR 1452`)
Es la propiedad que garantiza que las relaciones entre tablas permanezcan consistentes, impidiendo que existan registros "huérfanos" que apunten a identificadores inexistentes en la tabla padre. En MariaDB, cuando se viola una restricción de llave foránea al insertar o actualizar datos, el motor rechaza la operación emitiendo el **`ERROR 1452 (23000): Cannot add or update child row`**.

### `DEFINER`
Atributo de seguridad que indica qué cuenta de usuario es propietaria de una vista, procedimiento almacenado, función o disparador, y bajo qué privilegios se ejecuta. Durante una migración, si el usuario indicado en el `DEFINER` no existe en el servidor destino, la importación puede fallar con un error de permisos (`ERROR 1227`), por lo que se recomienda crear previamente los usuarios o depurar la cláusula `DEFINER` del archivo `.sql`.

### Charset y Collation (`utf8mb4`)
* **Charset (Juego de caracteres):** Define el conjunto de símbolos y su codificación binaria. En MariaDB se recomienda **`utf8mb4`**, ya que soporta de forma completa Unicode (incluyendo acentos, la letra `ñ` y emojis de 4 bytes), evitando errores de caracteres corruptos (como `Ã±`).
* **Collation (Cotejamiento):** Define las reglas para comparar y ordenar cadenas de texto (por ejemplo, `utf8mb4_uca1400_ai_ci` o `utf8mb4_general_ci`, donde `ai` significa insensible a acentos y `ci` insensible a mayúsculas/minúsculas).

### `sql_mode` y `secure_file_priv`
* **`sql_mode`:** Variable de configuración que controla el nivel de estrictez con el que MariaDB valida la sintaxis SQL y los datos (por ejemplo, rechazar fechas inválidas como `0000-00-00` o truncamientos silenciosos).
* **`secure_file_priv`:** Variable de seguridad que restringe el directorio específico del sistema operativo desde el cual el servidor tiene permiso para leer o escribir archivos mediante sentencias como `LOAD DATA INFILE` o `SELECT ... INTO OUTFILE`.

### Binlog (Registro binario)
Archivo de registro donde MariaDB guarda de forma secuencial todos los eventos que modifican datos o estructuras de la base de datos. Es un componente esencial tanto para la **recuperación a un punto en el tiempo (Point-in-Time Recovery)** como para configurar la **replicación** en migraciones con tiempo de inactividad mínimo.

---
