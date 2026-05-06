# 10.2 Hojas de Trucos (Cheat Sheets)

## Sintaxis SQL Rápida

### SELECT
```sql
SELECT [DISTINCT] columnas
FROM tabla
[JOIN tabla2 ON condicion]
[WHERE condicion]
[GROUP BY columnas]
[HAVING condicion]
[ORDER BY columna [ASC|DESC]]
[LIMIT n [OFFSET m]];
```

### Tipos de JOIN
```sql
INNER JOIN  → Solo registros que coinciden
LEFT JOIN   → Todos los de izquierda + coincidencias
RIGHT JOIN  → Todos los de derecha + coincidencias
FULL JOIN   → Todos los registros (no MySQL)
CROSS JOIN  → Producto cartesiano
```

### Funciones de Agregación
```sql
COUNT(*)        → Contar filas
COUNT(DISTINCT) → Contar valores únicos
SUM(col)        → Sumar valores
AVG(col)        → Promedio
MIN(col)        → Valor mínimo
MAX(col)        → Valor máximo
STDDEV(col)     → Desviación estándar
VARIANCE(col)   → Varianza
GROUP_CONCAT(col) → Concatenar valores (MySQL)
```

### Funciones de Ventana
```sql
ROW_NUMBER() OVER (PARTITION BY col ORDER BY col)  → Numeración
RANK()       OVER (PARTITION BY col ORDER BY col)  → Ranking con huecos
DENSE_RANK() OVER (PARTITION BY col ORDER BY col)  → Ranking sin huecos
NTILE(n)     OVER (ORDER BY col)                   → Dividir en n grupos
LAG(col, n)  OVER (ORDER BY col)                   → Valor n filas atrás
LEAD(col, n) OVER (ORDER BY col)                   → Valor n filas adelante
FIRST_VALUE(col) OVER (ORDER BY col)               → Primer valor
LAST_VALUE(col)  OVER (ORDER BY col)               → Último valor
SUM(col)    OVER (PARTITION BY col ORDER BY col)    → Suma acumulada
AVG(col)    OVER (ORDER BY col ROWS BETWEEN n PRECEDING AND CURRENT ROW) → Media móvil
```

## Comparación entre Motores

### Limit / Paginación
```sql
-- MySQL / MariaDB / PostgreSQL / SQLite
SELECT * FROM productos LIMIT 10 OFFSET 20;
SELECT * FROM productos LIMIT 20, 10;  -- MySQL (offset, limit)

-- SQL Server / Oracle 12+
SELECT * FROM productos ORDER BY id OFFSET 20 ROWS FETCH NEXT 10 ROWS ONLY;

-- Oracle (antes de 12c)
SELECT * FROM (SELECT rownum rn, p.* FROM productos p) WHERE rn BETWEEN 21 AND 30;
```

### Concatenación
```sql
-- MySQL
SELECT CONCAT(nombre, ' - ', email) FROM clientes;

-- PostgreSQL, SQL Server
SELECT nombre || ' - ' || email FROM clientes;

-- SQL Server
SELECT CONCAT(nombre, ' - ', email) FROM clientes;
```

### Fecha Actual
```sql
-- MySQL, MariaDB
SELECT NOW(), CURDATE(), CURTIME(), CURRENT_TIMESTAMP;

-- PostgreSQL
SELECT NOW(), CURRENT_DATE, CURRENT_TIME, CURRENT_TIMESTAMP;

-- SQL Server
SELECT GETDATE(), SYSDATETIME(), CURRENT_TIMESTAMP;
```

## Palabras Reservadas Comunes
```
ADD, ALL, ALTER, AND, ANY, AS, ASC, AUTO_INCREMENT,
BACKUP, BETWEEN, BY,
CASCADE, CASE, CHANGE, CHECK, COLUMN, COMMIT, CONSTRAINT, CREATE, CROSS, CURRENT_DATE,
DATABASE, DECIMAL, DEFAULT, DELETE, DESC, DISTINCT, DROP,
ELSE, END, ENGINE, ESCAPE, EVENT, EXCEPT, EXEC, EXISTS, EXPLAIN,
FALSE, FETCH, FOREIGN, FROM, FULL, FUNCTION,
GRANT, GROUP, GROUP_CONCAT,
HAVING,
IF, IN, INDEX, INNER, INSERT, INTERSECT, INTO, IS,
JOIN,
KEY,
LANGUAGE, LEFT, LIKE, LIMIT, LOCK, LOGS,
MATCHED, MAX, MERGE, MIN, MODIFY,
NATURAL, NO, NOT, NULL, NULLIF,
OF, ON, ONLY, OPEN, OPTION, OR, ORDER, OUTER, OUTPUT, OVER,
PARTITION, PLAN, PRECISION, PRIMARY, PROCEDURE, PUBLIC,
RANGE, READ, RECONFIGURE, REFERENCES, REPLICATION, RESTORE, RESTRICT, RETURN, REVOKE, RIGHT, ROLLBACK, ROW, ROWS,
SAVE, SCHEMA, SELECT, SEQUENCE, SERIALIZABLE, SESSION, SET, SHOW, SHUTDOWN, SOME, START, STATISTICS, SYSTEM,
TABLE, TEMP, TEMPORARY, THEN, TO, TOP, TRANSACTION, TRIGGER, TRUE, TRUNCATE,
UNION, UNIQUE, UPDATE, USE, USER, USING,
VALUE, VALUES, VARCHAR, VIEW,
WAITFOR, WHEN, WHERE, WHILE, WITH, WORK,
WRITE,
XML,
YEAR,
ZONE
```

## Índices - Cuándo Usarlos

### Crear Índice
```sql
CREATE INDEX idx_nombre ON tabla(columna);
CREATE UNIQUE INDEX idx_unique ON tabla(columna);
CREATE INDEX idx_compuesto ON tabla(col1, col2);
CREATE FULLTEXT INDEX idx_texto ON tabla(columna);
CREATE INDEX idx_parcial ON tabla(columna) WHERE condicion;  -- PostgreSQL
```

### Cuándo Indexar
```
✅ Columnas en WHERE frecuente
✅ Columnas en JOIN (FOREIGN KEY)
✅ Columnas en ORDER BY
✅ Columnas en GROUP BY
✅ Columnas con alta cardinalidad (muchos valores distintos)
✅ Tablas grandes (> 10,000 filas)

❌ Tablas pequeñas
❌ Columnas con pocos valores (booleanos, estados pequeños)
❌ Columnas que se modifican frecuentemente
❌ Tablas con muchas escrituras y pocas lecturas
```

## SQL Injection - Payloads Comunes

### Detección
```sql
'           → Error de sintaxis
' OR 1=1 -- → TRUE (debe funcionar)
' OR 1=2 -- → FALSE
' OR SLEEP(5) -- → Retardo de 5 segundos
```

### Extracción de Datos
```sql
' UNION SELECT database(), user(), version() --
' UNION SELECT table_name, NULL FROM information_schema.tables --
' UNION SELECT column_name, data_type FROM information_schema.columns WHERE table_name='usuarios' --
```

### Blind SQL
```sql
' AND SUBSTRING(database(),1,1)='s' --
' OR IF(1=1, SLEEP(3), 0) --
```

## Formatos de Fecha

```sql
-- MySQL
DATE_FORMAT(fecha, '%d/%m/%Y')     → 15/01/2024
DATE_FORMAT(fecha, '%Y-%m-%d')     → 2024-01-15
DATE_FORMAT(fecha, '%M %d, %Y')    → January 15, 2024
DATE_FORMAT(fecha, '%W')           → Monday
DATE_FORMAT(fecha, '%H:%i:%s')     → 14:30:00

-- PostgreSQL
TO_CHAR(fecha, 'DD/MM/YYYY')       → 15/01/2024
TO_CHAR(fecha, 'Month DD, YYYY')   → January 15, 2024

-- SQL Server
FORMAT(fecha, 'dd/MM/yyyy')        → 15/01/2024
FORMAT(fecha, 'dddd')              → Monday
```

## Expresiones Regulares en SQL

```sql
-- MySQL
SELECT * FROM productos WHERE nombre_comercial REGEXP '^[A-Z].*[0-9]$';
SELECT * FROM clientes WHERE email REGEXP '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$';

-- PostgreSQL
SELECT * FROM productos WHERE nombre_comercial ~ '^[A-Z].*[0-9]$';

-- SQL Server
SELECT * FROM productos WHERE nombre_comercial LIKE '[A-Z]%[0-9]';
```

## Resolución de Problemas

### Consultas Lentas
```sql
-- 1. Verificar índices
EXPLAIN SELECT ...;

-- 2. Actualizar estadísticas
ANALYZE TABLE facturas;

-- 3. Revisar slow query log
-- 4. Simplificar JOINs y subconsultas
```

### Bloqueos
```sql
SHOW PROCESSLIST;
SHOW FULL PROCESSLIST;
KILL CONNECTION id;

SELECT * FROM information_schema.INNODB_TRX;
SELECT * FROM information_schema.INNODB_LOCKS;
SELECT * FROM information_schema.INNODB_LOCK_WAITS;
```

## Backup y Restauración Rápida

```bash
# MySQL
mysqldump -u user -p db > backup.sql
mysql -u user -p db < backup.sql

# PostgreSQL
pg_dump -U user db > backup.sql
psql -U user db < backup.sql

# SQL Server
sqlcmd -S server -U SA -Q "BACKUP DATABASE db TO DISK='file.bak'"
sqlcmd -S server -U SA -Q "RESTORE DATABASE db FROM DISK='file.bak'"
```

---

## Anterior: [01 Glosario](../10-apendices/01-glosario.md)
## Siguiente: [03 Recursos Adicionales](../10-apendices/03-recursos-adicionales.md)

[Volver al índice](../README.md)