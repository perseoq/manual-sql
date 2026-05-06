# 10.1 Glosario de Términos SQL y Seguridad

## A
**ACID**: Conjunto de propiedades de las transacciones: Atomicidad, Consistencia, Aislamiento, Durabilidad.

**Agregación**: Función que calcula un valor a partir de un conjunto de filas (SUM, COUNT, AVG, etc.).

**Alias**: Nombre temporal dado a una columna o tabla dentro de una consulta (AS).

**ALTER**: Comando DDL para modificar la estructura de una tabla existente.

**ANALYZE**: Comando que actualiza las estadísticas de una tabla para el optimizador de consultas.

**AND**: Operador lógico que retorna TRUE si todas las condiciones son verdaderas.

**AUTO_INCREMENT**: Atributo de MySQL que genera automáticamente valores numéricos secuenciales.

**AVG**: Función de agregación que calcula el promedio de valores.

## B
**BACKUP**: Copia de seguridad de la base de datos.

**BETWEEN**: Operador que selecciona valores dentro de un rango (inclusivo).

**B-Tree**: Estructura de índice balanceado, la más común en bases de datos relacionales.

**Blind SQL Injection**: Técnica de inyección SQL donde no se ven los resultados directamente, se infieren.

## C
**CASCADE**: Acción referencial que propaga cambios (ON DELETE CASCADE).

**CASE**: Expresión condicional en SQL (similar a if-then-else).

**CHECK**: Restricción que valida datos en una columna.

**CLV (Customer Lifetime Value)**: Valor estimado de un cliente durante toda la relación comercial.

**CLUSTERED INDEX**: Índice que determina el orden físico de los datos en disco (SQL Server).

**COALESCE**: Función que retorna el primer valor no nulo de una lista.

**COMMIT**: Confirma una transacción, haciendo los cambios permanentes.

**COUNT**: Función de agregación que cuenta filas.

**CREATE**: Comando DDL para crear objetos (tablas, vistas, índices, etc.).

**CROSS JOIN**: Producto cartesiano de dos tablas.

**CTE (Common Table Expression)**: Tabla temporal definida con WITH para usar dentro de una consulta.

**CURSOR**: Estructura que permite recorrer filas una por una en procedural SQL.

## D
**DATABASE**: Conjunto organizado de datos almacenados y relacionados.

**DCL (Data Control Language)**: GRANT, REVOKE - control de acceso.

**DDL (Data Definition Language)**: CREATE, ALTER, DROP - definición de estructura.

**Deadlock**: Situación donde dos transacciones se bloquean mutuamente.

**DECIMAL**: Tipo de dato numérico exacto con precisión y escala definidas.

**DELETE**: Comando DML para eliminar filas de una tabla.

**DEFAULT**: Valor por defecto para una columna.

**DISTINCT**: Palabra clave que elimina duplicados de los resultados.

**DML (Data Manipulation Language)**: SELECT, INSERT, UPDATE, DELETE - manipulación de datos.

**DROP**: Comando DDL para eliminar objetos de la base de datos.

## E
**Entity**: Entidad, objeto o concepto del mundo real representado en una tabla.

**ENUM**: Tipo de dato que permite seleccionar un valor de una lista predefinida (MySQL).

**Error-based SQL Injection**: Técnica que aprovecha mensajes de error para extraer información.

**EXISTS**: Operador que verifica si una subconsulta retorna al menos una fila.

**EXPLAIN**: Comando que muestra el plan de ejecución de una consulta.

## F
**FOREIGN KEY (FK)**: Clave foránea que referencia la clave primaria de otra tabla.

**FROM**: Cláusula que especifica la(s) tabla(s) de origen en una consulta.

**FULL JOIN**: Combinación que retorna todas las filas de ambas tablas.

**FUNCTION**: Bloque de código SQL que retorna un valor.

## G
**GRANT**: Comando DCL que otorga permisos a un usuario.

**GROUP BY**: Cláusula que agrupa filas con el mismo valor en columnas especificadas.

**GROUP_CONCAT**: Función que concatena valores de un grupo en una sola cadena (MySQL).

## H
**Hardening**: Proceso de asegurar un sistema reduciendo su superficie de ataque.

**HAVING**: Cláusula que filtra grupos después de GROUP BY.

**Hashing**: Transformación irreversible de datos (usado para contraseñas).

## I
**IN**: Operador que verifica si un valor está en una lista.

**INDEX**: Estructura que acelera la búsqueda de datos en una tabla.

**INNER JOIN**: Combinación que retorna solo filas con correspondencia en ambas tablas.

**INSERT**: Comando DML para agregar nuevas filas a una tabla.

**IS NULL**: Operador que verifica si un valor es nulo.

**ISOLATION LEVEL**: Nivel de aislamiento de una transacción (READ UNCOMMITTED, READ COMMITTED, REPEATABLE READ, SERIALIZABLE).

## J
**JOIN**: Operación que combina filas de dos o más tablas basada en una condición.

**JSON**: Formato de datos ligero para intercambio de información.

## K
**KPI (Key Performance Indicator)**: Indicador clave de rendimiento.

**Kardex**: Registro detallado de entradas y salidas de inventario.

## L
**LAG**: Función de ventana que accede a la fila anterior.

**LEAD**: Función de ventana que accede a la fila siguiente.

**LEFT JOIN**: Combinación que retorna todas las filas de la tabla izquierda.

**LIKE**: Operador de búsqueda por patrón con comodines (% y _).

**LIMIT**: Cláusula que limita el número de filas retornadas.

**LOCK**: Mecanismo que controla el acceso concurrente a datos.

## M
**MAX**: Función de agregación que retorna el valor máximo.

**MERGE**: Operación que combina INSERT, UPDATE y DELETE (UPSERT).

**MIN**: Función de agregación que retorna el valor mínimo.

** MATERIALIZED VIEW**: Vista cuyos resultados se almacenan físicamente.

## N
**NATURAL JOIN**: JOIN que usa automáticamente columnas con el mismo nombre.

**NOCHECK**: Opción para omitir validación de restricciones.

**NON-CLUSTERED INDEX**: Índice separado del orden físico de la tabla.

**NOT NULL**: Restricción que impide valores nulos en una columna.

**NTILE**: Función de ventana que divide filas en N grupos.

**NULL**: Ausencia de valor (diferente de cero o vacío).

## O
**OFFSET**: Cláusula que salta N filas antes de retornar resultados.

**OLAP (Online Analytical Processing)**: Sistema optimizado para consultas analíticas.

**OLTP (Online Transaction Processing)**: Sistema optimizado para transacciones.

**ON**: Cláusula que especifica la condición de JOIN.

**OR**: Operador lógico que retorna TRUE si alguna condición es verdadera.

**ORDER BY**: Cláusula que ordena los resultados.

**ORM (Object-Relational Mapping)**: Técnica que mapea objetos a tablas de BD.

**OUTPUT**: Cláusula de SQL Server que retorna datos de filas afectadas.

**OVER**: Cláusula que define la ventana para funciones de ventana.

## P
**PARTITION BY**: División de filas en particiones para funciones de ventana.

**PARTITIONING**: División de tablas grandes en partes más pequeñas.

**PEPS**: Primeras Entradas, Primeras Salidas (método de valorización de inventario).

**Prepared Statement**: Consulta SQL precompilada con parámetros (previene SQL injection).

**PRIMARY KEY (PK)**: Identificador único de cada fila en una tabla.

**PROCEDURE (Stored Procedure)**: Bloque de código SQL almacenado en el servidor.

## Q
**Query**: Consulta o sentencia SQL.

## R
**RANK**: Función de ventana que asigna un rango con empates (deja huecos).

**RECURSIVE**: CTE que se llama a sí misma (WITH RECURSIVE).

**REVOKE**: Comando DCL que revoca permisos.

**RFM**: Recencia, Frecuencia, Monetario (análisis de clientes).

**RIGHT JOIN**: Combinación que retorna todas las filas de la tabla derecha.

**ROLLBACK**: Deshace una transacción, revirtiendo los cambios.

**ROW_NUMBER**: Función de ventana que asigna un número único a cada fila.

**RPO (Recovery Point Objective)**: Máxima cantidad de datos aceptable perder.

**RTO (Recovery Time Objective)**: Tiempo máximo para recuperar el servicio.

## S
**SAVEPOINT**: Punto de guardado dentro de una transacción.

**SELECT**: Comando DQL para consultar datos.

**SELF JOIN**: Una tabla unida consigo misma.

**SEQUENCE**: Objeto que genera valores numéricos secuenciales (PostgreSQL, Oracle).

**SQL**: Structured Query Language.

**SQL Injection**: Vulnerabilidad que permite inyectar código SQL malicioso.

**STDDEV**: Desviación estándar (función estadística).

**SUM**: Función de agregación que suma valores.

## T
**TABLE**: Objeto que almacena datos en filas y columnas.

**TCL (Transaction Control Language)**: COMMIT, ROLLBACK, SAVEPOINT.

**TDE (Transparent Data Encryption)**: Cifrado transparente de datos en disco.

**Temporary Table**: Tabla temporal que existe solo durante la sesión.

**Time-based Blind SQL Injection**: Técnica que usa retardos para inferir información.

**TRIGGER**: Código que se ejecuta automáticamente en respuesta a eventos.

**TRUNCATE**: Comando DDL que elimina todas las filas de una tabla (más rápido que DELETE).

## U
**UNION**: Combina resultados de múltiples consultas eliminando duplicados.

**UNION ALL**: Combina resultados sin eliminar duplicados.

**UNIQUE**: Restricción que asegura valores únicos en una columna.

**UPDATE**: Comando DML para modificar datos existentes.

**UPSERT**: INSERT + UPDATE (MERGE, ON DUPLICATE KEY UPDATE, ON CONFLICT).

**USER**: Cuenta que accede a la base de datos.

**USING**: Cláusula alternativa a ON para JOIN cuando las columnas tienen el mismo nombre.

## V
**VARCHAR**: Tipo de dato de cadena de longitud variable.

**VIEW**: Consulta almacenada que se comporta como una tabla virtual.

## W
**WAF (Web Application Firewall)**: Firewall para aplicaciones web.

**WHERE**: Cláusula que filtra filas basada en condiciones.

**Window Function**: Función que calcula sobre un conjunto de filas relacionadas.

**WITH**: Palabra clave para definir CTE.

## X
**XML**: eXtensible Markup Language, formato de datos.

**XSS (Cross-Site Scripting)**: Vulnerabilidad web relacionada (no SQL, pero común).

## Y
**YEAR**: Función que extrae el año de una fecha.

## Z
**Z-Score**: Medida estadística que indica cuántas desviaciones estándar está un valor de la media.

---

## Anterior: [04 Dashboards Sql](../09-analisis-datos/04-dashboards-sql.md)
## Siguiente: [02 Hojas De Trucos](../10-apendices/02-hojas-de-trucos.md)

[Volver al índice](../README.md)