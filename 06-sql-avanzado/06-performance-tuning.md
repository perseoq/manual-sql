# 6.6 Performance Tuning

## Identificación de Cuellos de Botella

### Slow Query Log
```ini
# MySQL my.cnf
slow_query_log = 1
slow_query_log_file = /var/log/mysql/slow-queries.log
long_query_time = 2
log_queries_not_using_indexes = 1
```

```sql
-- Ver consultas lentas
SELECT * FROM mysql.slow_log ORDER BY start_time DESC LIMIT 10;
```

### pg_stat_statements (PostgreSQL)
```sql
CREATE EXTENSION pg_stat_statements;

SELECT 
    query,
    calls,
    total_time / calls AS avg_time_ms,
    rows / calls AS avg_rows,
    shared_blks_hit / NULLIF(shared_blks_read + shared_blks_hit, 0) * 100 AS cache_hit_ratio
FROM pg_stat_statements
ORDER BY total_time DESC
LIMIT 10;
```

## EXPLAIN y Planes de Ejecución

### Usar EXPLAIN
```sql
EXPLAIN SELECT 
    c.nombre, 
    SUM(f.total) AS total_compras
FROM clientes c
JOIN facturas f ON c.id_cliente = f.id_cliente
WHERE f.fecha_emision >= '2024-01-01'
GROUP BY c.id_cliente
ORDER BY total_compras DESC
LIMIT 10;
```

### EXPLAIN ANALYZE (ejecución real)
```sql
-- MySQL
EXPLAIN ANALYZE SELECT ...;

-- PostgreSQL
EXPLAIN (ANALYZE, BUFFERS) SELECT ...;
```

## Estrategias de Indexación

### Identificar Índices Faltantes
```sql
-- MySQL: Consultas sin índice
SELECT * FROM mysql.slow_log 
WHERE sql_text NOT LIKE '%mysql%'
LIMIT 100;

-- PostgreSQL: Índices recomendados
SELECT 
    schemaname,
    tablename,
    indexname,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch
FROM pg_stat_user_indexes
ORDER BY idx_scan ASC
LIMIT 20;
```

### Índices Cubrientes (Covering Index)
```sql
-- Índice que contiene TODAS las columnas necesarias para la consulta
-- La consulta se responde completamente desde el índice (sin tocar la tabla)

CREATE INDEX idx_cubriente_ventas ON facturas(fecha_emision, id_cliente, total, estado);

-- Esta consulta solo usa el índice:
SELECT fecha_emision, id_cliente, total 
FROM facturas 
WHERE fecha_emision >= '2024-01-01' AND estado = 'activa';
```

## Optimización de JOINs

### Usar STRAIGHT_JOIN (MySQL)
```sql
-- Forzar orden de JOINs
SELECT STRAIGHT_JOIN 
    c.nombre, f.total
FROM clientes c
JOIN facturas f ON c.id_cliente = f.id_cliente
WHERE f.fecha_emision >= '2024-01-01';
```

### JOIN vs EXISTS
```sql
-- Para encontrar registros existentes, EXISTS suele ser más rápido
-- Lento:
SELECT DISTINCT c.*
FROM clientes c
JOIN facturas f ON c.id_cliente = f.id_cliente;

-- Rápido:
SELECT c.*
FROM clientes c
WHERE EXISTS (SELECT 1 FROM facturas f WHERE f.id_cliente = c.id_cliente);
```

## Optimización de Subconsultas

```sql
-- En lugar de subconsulta correlacionada:
SELECT p.*, 
    (SELECT AVG(precio_venta) FROM productos p2 WHERE p2.id_categoria = p.id_categoria) AS promedio_categoria
FROM productos p;

-- Usar JOIN con GROUP BY:
SELECT p.*, pc.promedio_categoria
FROM productos p
JOIN (
    SELECT id_categoria, AVG(precio_venta) AS promedio_categoria
    FROM productos
    GROUP BY id_categoria
) pc ON p.id_categoria = pc.id_categoria;
```

## Optimización de GROUP BY y ORDER BY

```sql
-- Índice compuesto que cubra GROUP BY y ORDER BY
CREATE INDEX idx_grupo_orden ON facturas(fecha_emision, id_cliente, total);

-- La siguiente consulta se beneficia:
SELECT 
    DATE(fecha_emision) AS dia,
    id_cliente,
    SUM(total) AS total
FROM facturas
WHERE fecha_emision >= '2024-01-01'
GROUP BY DATE(fecha_emision), id_cliente
ORDER BY dia, id_cliente;
```

## Uso de Tablas Temporales

```sql
-- Para consultas muy pesadas, usar tabla temporal
CREATE TEMPORARY TABLE temp_ventas_mensuales AS
SELECT 
    DATE_FORMAT(fecha_emision, '%Y-%m') AS mes,
    id_cliente,
    SUM(total) AS total_mes
FROM facturas
WHERE YEAR(fecha_emision) = 2024 AND estado = 'activa'
GROUP BY DATE_FORMAT(fecha_emision, '%Y-%m'), id_cliente;

-- Luego consultar la tabla temporal
SELECT * FROM temp_ventas_mensuales WHERE total_mes > 10000;
```

## Particionamiento

### Particionar por Rango
```sql
CREATE TABLE facturas_part (
    id_factura INT,
    folio VARCHAR(30),
    fecha_emision DATE,
    total DECIMAL(12,2)
) PARTITION BY RANGE (YEAR(fecha_emision)) (
    PARTITION p2022 VALUES LESS THAN (2023),
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION p2024 VALUES LESS THAN (2025),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);

-- La consulta solo escanea la partición relevante
SELECT * FROM facturas_part 
WHERE fecha_emision BETWEEN '2024-01-01' AND '2024-12-31';
```

## Configuración del Servidor

### MySQL - Parámetros Clave
```ini
[mysqld]
# Memoria
innodb_buffer_pool_size = 4G          # 70-80% de RAM
innodb_log_file_size = 512M
innodb_flush_log_at_trx_commit = 2    # Balance rendimiento/seguridad

# Conexiones
max_connections = 500
thread_cache_size = 50

# Cache de consultas (MySQL 5.7, no en 8.0)
query_cache_type = 0

# Temp tables
tmp_table_size = 64M
max_heap_table_size = 64M

# Sorting
sort_buffer_size = 2M
read_buffer_size = 2M
read_rnd_buffer_size = 4M

# Logs
slow_query_log = 1
long_query_time = 2
```

### PostgreSQL - Parámetros Clave
```ini
shared_buffers = 2GB
effective_cache_size = 6GB
work_mem = 64MB
maintenance_work_mem = 512MB
wal_buffers = 16MB
random_page_cost = 1.1  # Para SSD
```

## Monitoreo de Performance

### Consultas Más Ejecutadas
```sql
-- MySQL
SELECT 
    digest_text,
    count_star,
    sum_timer_wait / 1000000000 AS total_time_sec,
    avg_timer_wait / 1000000000 AS avg_time_sec
FROM performance_schema.events_statements_summary_by_digest
ORDER BY sum_timer_wait DESC
LIMIT 10;
```

### Tamaño de Tablas
```sql
SELECT 
    table_schema AS base_datos,
    table_name AS tabla,
    ROUND((data_length + index_length) / 1024 / 1024, 2) AS tamaño_mb,
    ROUND(data_length / 1024 / 1024, 2) AS datos_mb,
    ROUND(index_length / 1024 / 1024, 2) AS indices_mb,
    table_rows AS filas
FROM information_schema.TABLES
WHERE table_schema = 'sistema_ventas'
ORDER BY tamaño_mb DESC;
```

### Cache Hit Ratio
```sql
-- MySQL
SELECT 
    (SUM(innodb_buffer_pool_read_requests) - SUM(innodb_buffer_pool_reads)) 
    / SUM(innodb_buffer_pool_read_requests) * 100 AS cache_hit_ratio
FROM performance_schema.global_status
WHERE variable_name IN ('Innodb_buffer_pool_read_requests', 'Innodb_buffer_pool_reads');

-- Objetivo: > 99%
```

## Anti-Patrones (Lo que NO Hacer)

### 1. SELECT * en Producción
```sql
-- MAL
SELECT * FROM facturas WHERE id_cliente = 100;

-- BIEN
SELECT id_factura, folio, total, fecha_emision FROM facturas WHERE id_cliente = 100;
```

### 2. Funciones en Columnas Indexadas
```sql
-- MAL (no usa índice)
SELECT * FROM facturas WHERE YEAR(fecha_emision) = 2024;

-- BIEN (usa índice)
SELECT * FROM facturas WHERE fecha_emision >= '2024-01-01' AND fecha_emision < '2025-01-01';
```

### 3. LIKE con Comodín Inicial
```sql
-- MAL (no usa índice)
SELECT * FROM productos WHERE nombre_producto LIKE '%laptop%';

-- BIEN (usa índice si existe)
SELECT * FROM productos WHERE nombre_producto LIKE 'laptop%';
```

### 4. JOIN sin Índices
```sql
-- MAL: No hay índice en facturas.id_cliente
SELECT c.nombre, f.total
FROM clientes c
JOIN facturas f ON c.id_cliente = f.id_cliente;

-- BIEN: Crear índice
CREATE INDEX idx_facturas_cliente ON facturas(id_cliente);
```

### 5. NO Hacer en Producción
```sql
-- 1. Borrar datos sin WHERE
DELETE FROM facturas;

-- 2. Actualizar toda la tabla
UPDATE clientes SET activo = TRUE;

-- 3. DROP sin verificar
DROP TABLE facturas;

-- 4. ALTER sin planificar
ALTER TABLE facturas ADD COLUMN nuevo_campo TEXT;
```

## Checklist de Optimización

- [ ] Slow query log habilitado
- [ ] Índices en columnas de WHERE y JOIN
- [ ] Índices compuestos optimizados (orden correcto)
- [ ] Evitar SELECT *
- [ ] Funciones no bloquean índices
- [ ] LIKE no comienza con %
- [ ] JOIN tienen índices foráneos
- [ ] EXPLAIN analizado en consultas lentas
- [ ] Estadísticas actualizadas (ANALYZE)
- [ ] Particionamiento en tablas grandes
- [ ] Buffer pool configurado correctamente
- [ ] Conexiones limitadas y controladas
- [ ] Tablas temporales optimizadas
- [ ] Subconsultas convertidas a JOIN cuando posible

---

## Anterior: [05 Transacciones](../06-sql-avanzado/05-transacciones.md)
## Siguiente: [01 Que Es Sql Injection](../07-sql-injection/01-que-es-sql-injection.md)

[Volver al índice](../README.md)
