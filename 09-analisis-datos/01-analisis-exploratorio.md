# 9.1 Análisis Exploratorio de Datos con SQL

## Introducción al EDA

El Análisis Exploratorio de Datos (EDA) es el proceso de examinar datos para descubrir patrones, anomalías, relaciones y verificar supuestos antes de realizar análisis más formales.

## 1. Perfil de Datos (Data Profiling)

### Estadísticas Descriptivas
```sql
-- Resumen completo de una tabla
SELECT 
    'Facturas' AS tabla,
    COUNT(*) AS total_registros,
    COUNT(DISTINCT id_cliente) AS clientes_unicos,
    MIN(total) AS venta_minima,
    MAX(total) AS venta_maxima,
    ROUND(AVG(total), 2) AS venta_promedio,
    ROUND(AVG(NULLIF(total, 0)), 2) AS promedio_sin_ceros,
    ROUND(STDDEV(total), 2) AS desviacion_estandar,
    ROUND(VARIANCE(total), 2) AS varianza,
    SUM(CASE WHEN total IS NULL THEN 1 ELSE 0 END) AS nulos,
    SUM(CASE WHEN total = 0 THEN 1 ELSE 0 END) AS ceros
FROM facturas;

-- Percentiles
SELECT 
    ROUND(AVG(CASE WHEN percentil = 1 THEN total END), 2) AS p25,
    ROUND(AVG(CASE WHEN percentil = 2 THEN total END), 2) AS p50,
    ROUND(AVG(CASE WHEN percentil = 3 THEN total END), 2) AS p75,
    ROUND(AVG(CASE WHEN percentil = 4 THEN total END), 2) AS p95,
    ROUND(AVG(CASE WHEN percentil = 5 THEN total END), 2) AS p99
FROM (
    SELECT 
        total,
        NTILE(100) OVER (ORDER BY total) AS percentil
    FROM facturas WHERE estado = 'activa'
) AS sub
WHERE percentil IN (25, 50, 75, 95, 99);
```

### Perfil por Columna
```sql
-- Análisis de calidad de datos por tabla
SELECT 
    'id_cliente' AS columna,
    COUNT(*) AS total,
    COUNT(DISTINCT id_cliente) AS unicos,
    SUM(CASE WHEN id_cliente IS NULL THEN 1 ELSE 0 END) AS nulos,
    ROUND(SUM(CASE WHEN id_cliente IS NULL THEN 1 ELSE 0 END) / COUNT(*) * 100, 2) AS pct_nulos,
    NULL AS min,
    NULL AS max,
    NULL AS promedio
FROM facturas

UNION ALL

SELECT 
    'total',
    COUNT(*),
    COUNT(DISTINCT total),
    SUM(CASE WHEN total IS NULL THEN 1 ELSE 0 END),
    ROUND(SUM(CASE WHEN total IS NULL THEN 1 ELSE 0 END) / COUNT(*) * 100, 2),
    MIN(total),
    MAX(total),
    ROUND(AVG(total), 2)
FROM facturas

UNION ALL

SELECT 
    'estado',
    COUNT(*),
    COUNT(DISTINCT estado),
    SUM(CASE WHEN estado IS NULL THEN 1 ELSE 0 END),
    ROUND(SUM(CASE WHEN estado IS NULL THEN 1 ELSE 0 END) / COUNT(*) * 100, 2),
    NULL, NULL, NULL
FROM facturas;
```

## 2. Análisis de Distribuciones

### Distribución de Ventas
```sql
-- Histograma de montos de venta
SELECT 
    CASE 
        WHEN total BETWEEN 0 AND 100 THEN '0-100'
        WHEN total BETWEEN 101 AND 500 THEN '101-500'
        WHEN total BETWEEN 501 AND 1000 THEN '501-1,000'
        WHEN total BETWEEN 1001 AND 5000 THEN '1,001-5,000'
        WHEN total BETWEEN 5001 AND 10000 THEN '5,001-10,000'
        WHEN total BETWEEN 10001 AND 50000 THEN '10,001-50,000'
        ELSE '50,000+'
    END AS rango,
    COUNT(*) AS frecuencia,
    LPAD('', COUNT(*) / 10, '*') AS barra
FROM facturas
WHERE estado = 'activa'
GROUP BY rango
ORDER BY MIN(total);
```

### Distribución de Productos por Precio
```sql
SELECT 
    CASE 
        WHEN precio_venta < 100 THEN 'Económico (<$100)'
        WHEN precio_venta < 500 THEN 'Barato ($100-$500)'
        WHEN precio_venta < 2000 THEN 'Medio ($501-$2,000)'
        WHEN precio_venta < 10000 THEN 'Premium ($2,001-$10,000)'
        ELSE 'Lujo (>$10,000)'
    END AS categoria_precio,
    COUNT(*) AS productos,
    ROUND(AVG(precio_venta), 2) AS precio_promedio,
    MIN(precio_venta) AS precio_min,
    MAX(precio_venta) AS precio_max
FROM productos
WHERE activo = TRUE
GROUP BY categoria_precio
ORDER BY AVG(precio_venta);
```

## 3. Detección de Valores Atípicos (Outliers)

### Método IQR (Rango Intercuartil)
```sql
WITH stats AS (
    SELECT 
        AVG(total) AS media,
        STDDEV(total) AS desv,
        COUNT(*) AS n
    FROM facturas WHERE estado = 'activa'
)
SELECT 
    f.id_factura,
    f.folio,
    f.total,
    ROUND((f.total - s.media) / s.desv, 2) AS z_score,
    CASE 
        WHEN ABS((f.total - s.media) / s.desv) > 3 THEN 'OUTLIER ⚠️'
        ELSE 'Normal'
    END AS clasificacion
FROM facturas f
CROSS JOIN stats s
WHERE f.estado = 'activa'
ORDER BY ABS(z_score) DESC
LIMIT 20;
```

### Detección de Anomalías por Cliente
```sql
WITH compras_cliente AS (
    SELECT 
        c.id_cliente,
        c.nombre,
        COUNT(f.id_factura) AS num_compras,
        AVG(f.total) AS ticket_promedio,
        STDDEV(f.total) AS desv_ticket,
        MAX(f.total) AS max_compra
    FROM clientes c
    JOIN facturas f ON c.id_cliente = f.id_cliente AND f.estado = 'activa'
    GROUP BY c.id_cliente
    HAVING COUNT(f.id_factura) > 2
)
SELECT 
    nombre,
    num_compras,
    ROUND(ticket_promedio, 2) AS ticket_promedio,
    ROUND(max_compra, 2) AS max_compra,
    ROUND(desv_ticket, 2) AS desviacion,
    ROUND(max_compra / NULLIF(ticket_promedio, 0), 2) AS veces_promedio,
    CASE 
        WHEN max_compra > ticket_promedio + 3 * desv_ticket THEN 'POSIBLE FRAUDE ⚠️'
        ELSE 'Normal'
    END AS alerta
FROM compras_cliente
ORDER BY veces_promedio DESC
LIMIT 20;
```

## 4. Análisis de Valores Nulos

```sql
-- Cobertura de datos por tabla
SELECT 
    'clientes' AS tabla,
    ROUND(SUM(CASE WHEN nombre IS NOT NULL THEN 1 ELSE 0 END) / COUNT(*) * 100, 2) AS pct_nombre,
    ROUND(SUM(CASE WHEN email IS NOT NULL THEN 1 ELSE 0 END) / COUNT(*) * 100, 2) AS pct_email,
    ROUND(SUM(CASE WHEN telefono IS NOT NULL THEN 1 ELSE 0 END) / COUNT(*) * 100, 2) AS pct_telefono,
    ROUND(SUM(CASE WHEN rfc IS NOT NULL THEN 1 ELSE 0 END) / COUNT(*) * 100, 2) AS pct_rfc,
    ROUND(SUM(CASE WHEN direccion IS NOT NULL THEN 1 ELSE 0 END) / COUNT(*) * 100, 2) AS pct_direccion
FROM clientes

UNION ALL

SELECT 
    'facturas',
    ROUND(SUM(CASE WHEN id_cliente IS NOT NULL THEN 1 ELSE 0 END) / COUNT(*) * 100, 2),
    ROUND(SUM(CASE WHEN total IS NOT NULL THEN 1 ELSE 0 END) / COUNT(*) * 100, 2),
    ROUND(SUM(CASE WHEN iva IS NOT NULL THEN 1 ELSE 0 END) / COUNT(*) * 100, 2),
    ROUND(SUM(CASE WHEN metodo_pago IS NOT NULL THEN 1 ELSE 0 END) / COUNT(*) * 100, 2),
    ROUND(SUM(CASE WHEN fecha_emision IS NOT NULL THEN 1 ELSE 0 END) / COUNT(*) * 100, 2)
FROM facturas;
```

## 5. Análisis de Duplicados

```sql
-- Buscar clientes duplicados por email
SELECT 
    email,
    COUNT(*) AS ocurrencias,
    GROUP_CONCAT(nombre SEPARATOR ' | ') AS nombres,
    GROUP_CONCAT(id_cliente SEPARATOR ', ') AS ids
FROM clientes
WHERE email IS NOT NULL
GROUP BY email
HAVING COUNT(*) > 1;

-- Buscar facturas duplicadas
SELECT 
    folio,
    COUNT(*) AS ocurrencias
FROM facturas
GROUP BY folio
HAVING COUNT(*) > 1;

-- Buscar productos con mismo SKU
SELECT 
    sku,
    COUNT(*) AS ocurrencias,
    GROUP_CONCAT(nombre_comercial SEPARATOR ' | ') AS nombres
FROM productos
GROUP BY sku
HAVING COUNT(*) > 1;
```

## 6. Análisis de Relaciones

### Correlación entre Variables
```sql
-- Relación entre frecuencia de compra y monto
SELECT 
    CASE 
        WHEN num_compras BETWEEN 1 AND 2 THEN '1-2 compras'
        WHEN num_compras BETWEEN 3 AND 5 THEN '3-5 compras'
        WHEN num_compras BETWEEN 6 AND 10 THEN '6-10 compras'
        WHEN num_compras BETWEEN 11 AND 20 THEN '11-20 compras'
        ELSE '20+ compras'
    END AS segmento_frecuencia,
    COUNT(*) AS clientes,
    ROUND(AVG(monto_total), 2) AS gasto_promedio,
    ROUND(AVG(ticket_promedio), 2) AS ticket_promedio,
    ROUND(AVG(antiguedad_dias), 0) AS antiguedad_promedio
FROM (
    SELECT 
        c.id_cliente,
        COUNT(f.id_factura) AS num_compras,
        SUM(f.total) AS monto_total,
        AVG(f.total) AS ticket_promedio,
        DATEDIFF(CURRENT_DATE, MIN(f.fecha_emision)) AS antiguedad_dias
    FROM clientes c
    JOIN facturas f ON c.id_cliente = f.id_cliente AND f.estado = 'activa'
    GROUP BY c.id_cliente
) AS sub
GROUP BY segmento_frecuencia
ORDER BY MIN(num_compras);
```

### Productos que se Compran Juntos
```sql
SELECT 
    p1.nombre_comercial AS producto_a,
    p2.nombre_comercial AS producto_b,
    COUNT(*) AS veces_juntos,
    ROUND(COUNT(*) * 100.0 / (
        SELECT COUNT(*) FROM facturas_detalle fd3
        JOIN facturas f3 ON fd3.id_factura = f3.id_factura
        WHERE fd3.id_producto = p1.id_producto AND f3.estado = 'activa'
    ), 2) AS confianza
FROM facturas_detalle fd1
JOIN facturas f1 ON fd1.id_factura = f1.id_factura AND f1.estado = 'activa'
JOIN productos p1 ON fd1.id_producto = p1.id_producto
JOIN facturas_detalle fd2 ON f1.id_factura = fd2.id_factura 
    AND fd2.id_producto > fd1.id_producto
JOIN productos p2 ON fd2.id_producto = p2.id_producto
GROUP BY p1.id_producto, p2.id_producto
HAVING COUNT(*) > 5
ORDER BY veces_juntos DESC
LIMIT 20;
```

## 7. Reporte de Calidad de Datos

```sql
-- Dashboard de calidad
SELECT 'Calidad de Datos - Sistema de Ventas' AS reporte;
SELECT 
    'Total de registros' AS metrica, COUNT(*) AS valor FROM facturas
UNION ALL
SELECT 'Registros con NULL', SUM(CASE WHEN total IS NULL THEN 1 ELSE 0 END) FROM facturas
UNION ALL
SELECT 'Registros duplicados', COUNT(*) - COUNT(DISTINCT folio) FROM facturas
UNION ALL
SELECT 'Clientes sin email', COUNT(*) FROM clientes WHERE email IS NULL
UNION ALL
SELECT 'Productos sin stock', COUNT(*) FROM productos_stock WHERE stock_actual = 0
UNION ALL
SELECT 'Facturas sin detalle', COUNT(*) FROM facturas f WHERE NOT EXISTS 
    (SELECT 1 FROM facturas_detalle fd WHERE fd.id_factura = f.id_factura);
```

---

## Anterior: [04 Backup Y Recovery](../08-seguridad/04-backup-y-recovery.md)
## Siguiente: [02 Series Temporales](../09-analisis-datos/02-series-temporales.md)

[Volver al índice](../README.md)
