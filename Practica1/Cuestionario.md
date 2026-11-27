# Cuestionario: Práctica 1 - Recuperación de una base de datos en MariaDB

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

## Preguntas y Respuestas

### 1. ¿Qué datos se habrían perdido si solo hubieras usado el respaldo de la Parte A?

Se habrían perdido **todos los cambios e inserciones realizados después del momento en que se generó el respaldo**. En el escenario de esta práctica, se habrían perdido definitivamente los dos registros agregados en la Parte B (**Pedro Nunez** y **Laura Vega**), quedando únicamente los 5 clientes iniciales que existían cuando se ejecutó `mariadb-dump`.

---

### 2. ¿Para qué sirve la opción `--master-data=2`?

La opción `--master-data=2` indica a `mariadb-dump` que escriba en la cabecera del archivo de respaldo `.sql` —en formato de **comentario SQL** (`-- CHANGE MASTER TO ...`)— el nombre exacto del archivo de registro binario (`MASTER_LOG_FILE`) y la posición exacta (`MASTER_LOG_POS`) en la que se encontraba el servidor al iniciar el respaldo. 

Esta coordenada es fundamental para la **Recuperación a un Punto en el Tiempo (PITR)**, ya que permite saber con precisión desde qué punto exacto del binlog se deben reaplicar las transacciones con `mariadb-binlog --start-position`, evitando omitir datos o duplicar operaciones que ya estaban incluidas en el respaldo.

---

### 3. ¿Por qué es necesario activar el binlog antes de que ocurra el error?

Porque el registro binario (*binlog*) únicamente graba de forma secuencial las operaciones de modificación (`INSERT`, `UPDATE`, `DELETE` y sentencias DDL) **a partir del momento en que se habilita en la configuración y se reinicia el servicio**. No tiene capacidad retroactiva; si el binlog estaba desactivado cuando se insertaron los datos nuevos o cuando ocurrió el error, MariaDB no habrá guardado ningún historial de esas transacciones y será imposible realizar una recuperación a un punto en el tiempo (PITR).

---

### 4. ¿Por qué se esperaron 5 segundos antes y después de anotar la hora?

Se esperaron 5 segundos antes y después de ejecutar `SELECT NOW()` para crear un **margen de tiempo limpio y seguro** entre la última transacción válida (el `INSERT` de Pedro y Laura) y la transacción destructiva (el `DELETE FROM clientes`). 

Como la bandera `--stop-datetime` de `mariadb-binlog` trabaja con una granularidad de segundos, si ambas operaciones ocurrieran dentro del mismo segundo, al filtrar por fecha y hora podría incluirse accidentalmente el `DELETE` o excluirse el `INSERT`. Dejar ese intervalo garantiza que la marca de tiempo anotada caiga estrictamente después de las inserciones válidas y antes del borrado accidental.