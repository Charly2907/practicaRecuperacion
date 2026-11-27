# Cuestionario: Práctica 2 - Migración de una base de datos en MariaDB

## **Intituto Tecnológico Nacional de México**  
### **Instituto Tecnológico de Milpa Alta 2**

* **Carrera:** Ingeniería en Sistemas Computacionales
* **Materia:** Administración de Bases de Datos
* **Alumno:** Basurto García Carlos Yael
* **Número de Control:** 241190029
* **Docente:** Ing. Diego Alberto Bollas Paredes
* **Semestre:** Quinto
* **Fecha:** 26/11/26

---

## Preguntas y Respuestas

### 1. ¿Por qué se hace un respaldo antes de migrar?

Realizar un respaldo previo es una regla obligatoria en la administración de bases de datos porque constituye la base del **plan de reversión (rollback)**. Durante un proceso de migración pueden ocurrir fallos inesperados, como cortes de energía, errores de compatibilidad, corrupción de archivos o ejecución accidental de sentencias destructivas. Contar con un respaldo completo y verificado garantiza que, ante cualquier incidente, se pueda restaurar la base de datos de origen a su estado exacto anterior al inicio de la operación, evitando la pérdida definitiva de información crítica.

---

### 2. ¿Para qué sirve la opción `--single-transaction`?

La opción `--single-transaction` instruye a `mariadb-dump` para que inicie una única transacción (en modo de aislamiento *repeatable read*) antes de comenzar a volcar los datos. Esto permite obtener una **instantánea consistente (snapshot)** de toda la base de datos en un punto exacto en el tiempo para tablas que utilizan motores transaccionales como **InnoDB**, sin necesidad de bloquear las tablas contra lecturas o escrituras durante el respaldo. De esta manera, se evita que cambios concurrentes generen inconsistencias entre tablas relacionadas mientras se realiza la exportación.

---

### 3. ¿Qué diferencia hay entre exportar con y sin `--databases`?

* **Con `--databases`:** El archivo `.sql` generado incluye explícitamente las sentencias `CREATE DATABASE` y `USE nombre_base_datos;`. Esto resulta útil para respaldos de recuperación (como `respaldo_previo.sql`), ya que al restaurarlo recrea automáticamente el esquema con su nombre original. Sin embargo, dificulta importar los datos en una base con un nombre diferente sin editar el archivo.
* **Sin `--databases`:** El archivo `.sql` contiene únicamente las sentencias de creación de tablas, vistas, rutinas e inserción de datos, omitiendo `CREATE DATABASE` y `USE`. Esto permite crear manualmente una base de datos destino con cualquier nombre (por ejemplo, `biblioteca_nueva`) e importar el volcado directamente en ella especificando el nombre del esquema al invocar el cliente `mariadb`.

---

### 4. ¿Qué dos pruebas permiten confirmar que los datos migraron sin pérdidas?

Las dos pruebas principales para verificar de forma objetiva que no hubo pérdida ni alteración de datos son:

1. **Conteo de filas (`SELECT COUNT(*) FROM tabla;`):** Permite comprobar que la cantidad exacta de registros en cada tabla de la base de datos de destino coincide con la registrada previamente en el origen.
2. **Suma de comprobación de tablas (`CHECKSUM TABLE tabla;`):** Calcula un valor numérico único (una huella digital) basado en el contenido completo de cada tabla. Si los valores de `Checksum` en origen y destino son idénticos (como ocurrió con `4192191419` en `autores` y `1759993422` en `libros`), se confirma con certeza técnica que los datos son exactamente los mismos bit a bit.

---

### 5. Si el destino estuviera en otro servidor, ¿qué paso adicional sería necesario?

Sería necesario **transferir el archivo `.sql` exportado desde el servidor de origen hacia el servidor de destino a través de la red**, por ejemplo utilizando una herramienta de copia segura como **`scp`** (`scp biblioteca_migracion.sql usuario@servidor_destino:/tmp/`) antes de ejecutar la importación en el nuevo servidor. 

Como alternativa sin archivo intermedio, se puede conectar directamente el flujo de salida de `mariadb-dump` al cliente `mariadb` del servidor remoto mediante una **tubería (`|`)** especificando el host destino (`-h servidor_destino`), asegurando previamente que el puerto `3306` y las reglas de firewall permitan la conexión. Además, en migraciones entre servidores distintos también se deben migrar los **usuarios y permisos** (`CREATE USER` y `SHOW GRANTS`), ya que no se incluyen en el volcado de una sola base de datos.