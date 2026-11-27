# Anexo: Investigación de Términos y Herramientas de Recuperación en MariaDB

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

### Propiedades ACID (Atomicidad y Durabilidad)
Son el conjunto de garantías que aseguran el procesamiento confiable de transacciones en una base de datos relacional. En materia de recuperación destacan dos:
* **Atomicidad:** Garantiza que una transacción se ejecute como una unidad indivisible ("todo o nada"); si ocurre un fallo a mitad de la operación, todos los cambios parciales se revierten.
* **Durabilidad:** Asegura que una vez que una transacción ha sido confirmada (`COMMIT`), sus efectos persisten de forma permanente incluso si ocurre un corte eléctrico o una caída abrupta del servidor.

### Redo Log y Write-Ahead Logging (WAL)
El **Redo Log** es un registro secuencial en disco donde el motor InnoDB escribe los cambios realizados por las transacciones **antes** de modificar las páginas de datos definitivas (técnica conocida como *Write-Ahead Logging*). Si el servidor se cae antes de escribir en disco los datos definitivos, al reiniciar utiliza el Redo Log para **rehacer (REDO)** las transacciones que ya habían hecho `COMMIT`.

### Undo Log
Es la estructura interna donde InnoDB almacena los valores anteriores de los registros modificados por transacciones activas. Cumple dos funciones: permite **deshacer (UNDO)** transacciones incompletas o canceladas (`ROLLBACK`) durante una recuperación tras caída, y sostiene las lecturas consistentes multiversión (MVCC).

### Checkpoint
Es un punto de sincronización en el cual InnoDB confirma que todas las páginas modificadas en memoria hasta cierto instante ya han sido escritas permanentemente en los archivos de datos del disco. Durante una recuperación tras caída (*crash recovery*), el motor solo necesita procesar el Redo Log a partir del último checkpoint, acelerando considerablemente el arranque.

### Doublewrite Buffer
Área de almacenamiento intermedia de InnoDB donde se escriben las páginas de datos antes de escribirlas en su ubicación final en el *tablespace*. Su propósito es proteger la base de datos contra **páginas parcialmente escritas ("páginas rotas")** en caso de un corte repentino de energía durante una operación de escritura en disco.

### Crash Recovery (Recuperación automática tras caída)
Proceso automático que ejecuta InnoDB al iniciar el servicio de MariaDB después de un cierre inesperado. Analiza el Redo Log desde el último checkpoint, reaplica las transacciones confirmadas que faltaban en disco (fase **REDO**) y revierte las transacciones que quedaron abiertas sin `COMMIT` mediante el Undo Log (fase **UNDO**).

### Binary Log (`binlog`)
Archivo de registro a nivel de servidor donde MariaDB almacena de forma cronológica todos los eventos que alteran los datos o la estructura de las bases de datos (`INSERT`, `UPDATE`, `DELETE`, `CREATE`, `DROP`, etc.). Es indispensable tanto para la **replicación** como para la **Recuperación a un Punto en el Tiempo (PITR)**.

### PITR (*Point-in-Time Recovery*)
Técnica de recuperación que combina un **respaldo completo** con la reproducción selectiva de los archivos **binlog** posteriores. Permite restaurar una base de datos hasta un segundo o posición exacta, rescatando transacciones válidas recientes y deteniendo la restauración justo antes de un error humano o lógico (como un `DROP TABLE` o un `DELETE` sin `WHERE`).

### `mariadb-binlog` (antes `mysqlbinlog`)
Utilidad de línea de comandos diseñada para leer, decodificar y extraer eventos de los archivos binarios del servidor en formato SQL legible. Admite filtros precisos como `--start-position`, `--stop-position`, `--start-datetime` y `--stop-datetime` para reproducir únicamente el fragmento deseado del historial de cambios.

### `--flush-logs` y `--master-data=2`
Opciones clave de `mariadb-dump` orientadas a PITR:
* **`--flush-logs`:** Cierra el archivo de registro binario actual y fuerza a MariaDB a abrir uno nuevo al momento de iniciar el respaldo, separando limpiamente los eventos previos de los posteriores.
* **`--master-data=2`:** Registra como comentario dentro del archivo `.sql` el nombre del binlog activo y su posición exacta (`CHANGE MASTER TO MASTER_LOG_FILE=..., MASTER_LOG_POS=...`) al instante del volcado.

### `innodb_flush_log_at_trx_commit` y `sync_binlog`
Parámetros críticos de durabilidad en el archivo `50-server.cnf`:
* **`innodb_flush_log_at_trx_commit = 1`:** Obliga a que el buffer del Redo Log se escriba y sincronice físicamente en disco en cada `COMMIT`, garantizando durabilidad total según ACID.
* **`sync_binlog = 1`:** Fuerza la sincronización en disco del registro binario tras cada transacción confirmada, evitando la pérdida de eventos del binlog ante una caída del sistema operativo.

### `mariadb-backup` (Respaldo físico e incremental)
Herramienta oficial de MariaDB para ejecutar **respaldos físicos en caliente** copiando directamente los archivos de datos sin detener el servidor. Soporta **respaldos incrementales** (copiar únicamente las páginas modificadas desde el último respaldo completo o incremental) y requiere una fase de preparación (`--prepare`) para aplicar el Redo Log antes de restaurar mediante `--copy-back`.

### `innodb_force_recovery`
Parámetro de emergencia (con niveles del `1` al `6`) que permite forzar el arranque de MariaDB cuando existe corrupción grave en las tablas o archivos de InnoDB. Al activarlo, el servidor opera únicamente para rescate de datos (permitiendo ejecutar `mariadb-dump`) y nunca debe mantenerse encendido en operación normal.

### Replicación, Failover, GTID y Galera Cluster
* **Replicación:** Mecanismo mediante el cual uno o más servidores (**réplicas**) reciben y aplican de forma continua los cambios registrados en el binlog de un servidor principal (**primario**).
* **GTID (*Global Transaction ID*):** Identificador único global asignado a cada transacción replicada, lo que simplifica la sincronización y conmutación entre servidores.
* **Failover:** Procedimiento (manual o automático) mediante el cual una réplica asume el rol de servidor principal cuando el primario falla (`STOP SLAVE; RESET SLAVE ALL; SET GLOBAL read_only = 0;`).
* **MariaDB Galera Cluster:** Solución de alta disponibilidad basada en replicación síncrona multi-maestro donde todos los nodos pueden aceptar lecturas y escrituras simultáneamente.

### RPO (*Recovery Point Objective*)
Métrica que define la cantidad máxima tolerable de pérdida de datos medida en tiempo ante un incidente (por ejemplo, si el último respaldo fue hace 24 horas sin binlog, el RPO es de 24 horas; con PITR y binlog continuo, el RPO puede acercarse a cero).

### RTO (*Recovery Time Objective*)
Métrica que establece el tiempo máximo aceptable que una base de datos o servicio puede permanecer fuera de línea tras un fallo antes de afectar gravemente la operación del negocio.

### Regla 3-2-1 de respaldos
Estándar de buenas prácticas para la protección de información que establece mantener al menos:
* **3** copias de los datos (la base en producción y dos respaldos).
* En **2** tipos de medios o dispositivos de almacenamiento distintos.
* Con al menos **1** copia almacenada fuera del sitio físico (*off-site* o en la nube) para sobrevivir a desastres locales.
