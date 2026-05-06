# 3.3 Análisis de Ventas

## Técnicas de Análisis de Ventas con SQL

### Análisis de Tendencias

#### Tendencia Mensual con Variación
```sql
WITH ventas_mensuales AS (
    SELECT 
        DATE_FORMAT(fecha_emision, '%Y-%m') AS mes,
        SUM(total) AS ingresos,
        COUNT(*) AS transacciones
    FROM facturas
    WHERE estado = 'activa'
      AND fecha_emision >= DATE_SUB(CURRENT_DATE, INTERVAL 24 MONTH)
    GROUP BY DATE_FORMAT(fecha_emision, '%Y-%m')
)
SELECT 
    mes,
    ingresos,
    transacciones,
    LAG(ingresos) OVER (ORDER BY mes) AS mes_anterior,
    ROUND((ingresos - LAG(ingresos) OVER (ORDER BY mes)) / NULLIF(LAG(ingresos) OVER (ORDER BY mes), 0) * 100, 2) AS crecimiento_mensual,
    ROUND(AVG(ingresos) OVER (ORDER BY mes ROWS BETWEEN 2 PRECEDING AND CURRENT ROW), 2) AS media_movil_3m,
    ROUND(ingresos - AVG(ingresos) OVER (ORDER BY mes ROWS BETWEEN 2 PRECEDING AND CURRENT ROW), 2) AS desviacion_media_movil
FROM ventas_mensuales
ORDER BY mes DESC;
```

#### Comparación Año vs Año
```sql
SELECT 
    MONTH(fecha_emision) AS mes,
    SUM(CASE WHEN YEAR(fecha_emision) = YEAR(CURRENT_DATE) THEN total ELSE 0 END) AS ingresos_actual,
    SUM(CASE WHEN YEAR(fecha_emision) = YEAR(CURRENT_DATE) - 1 THEN total ELSE 0 END) AS ingresos_anterior,
    ROUND(
        (SUM(CASE WHEN YEAR(fecha_emision) = YEAR(CURRENT_DATE) THEN total ELSE 0 END) -
         SUM(CASE WHEN YEAR(fecha_emision) = YEAR(CURRENT_DATE) - 1 THEN total ELSE 0 END))
        / NULLIF(SUM(CASE WHEN YEAR(fecha_emision) = YEAR(CURRENT_DATE) - 1 THEN total ELSE 0 END), 0) * 100, 2
    ) AS crecimiento_yoy
FROM facturas
WHERE YEAR(fecha_emision) IN (YEAR(CURRENT_DATE), YEAR(CURRENT_DATE) - 1)
  AND estado = 'activa'
GROUP BY MONTH(fecha_emision)
ORDER BY mes;
```

### Segmentación de Clientes (RFM)

RFM (Recency, Frequency, Monetary) es una técnica para segmentar clientes:

```sql
WITH rfm AS (
    SELECT 
        c.id_cliente,
        c.nombre,
        DATEDIFF(CURRENT_DATE, MAX(f.fecha_emision)) AS recencia,
        COUNT(DISTINCT f.id_factura) AS frecuencia,
        SUM(f.total) AS monetario,
        NTILE(5) OVER (ORDER BY DATEDIFF(CURRENT_DATE, MAX(f.fecha_emision)) ASC) AS r_score,
        NTILE(5) OVER (ORDER BY COUNT(DISTINCT f.id_factura) DESC) AS f_score,
        NTILE(5) OVER (ORDER BY SUM(f.total) DESC) AS m_score
    FROM clientes c
    JOIN facturas f ON c.id_cliente = f.id_cliente AND f.estado = 'activa'
    GROUP BY c.id_cliente
)
SELECT 
    id_cliente,
    nombre,
    recencia,
    frecuencia,
    monetario,
    r_score, f_score, m_score,
    CONCAT(r_score, f_score, m_score) AS rfm_celda,
    CASE 
        WHEN r_score >= 4 AND f_score >= 4 AND m_score >= 4 THEN 'Campeones'
        WHEN r_score >= 4 AND f_score >= 3 AND m_score >= 3 THEN 'Clientes Leales'
        WHEN r_score >= 4 AND f_score <= 2 AND m_score <= 2 THEN 'Nuevos'
        WHEN r_score >= 3 AND f_score >= 3 AND m_score >= 3 THEN 'Potenciales'
        WHEN r_score <= 2 AND f_score >= 3 AND m_score >= 3 THEN 'En Riesgo'
        WHEN r_score <= 2 AND f_score <= 2 AND m_score >= 3 THEN 'Necesitan Atención'
        WHEN r_score <= 2 AND f_score <= 2 AND m_score <= 2 THEN 'Perdidos'
        ELSE 'Otros'
    END AS segmento
FROM rfm
ORDER BY monetario DESC;
```

### Análisis de Cohortes

```sql
-- Cohortes basadas en mes de primera compra
WITH primer_compra AS (
    SELECT 
        id_cliente,
        DATE_FORMAT(MIN(fecha_emision), '%Y-%m') AS cohorte
    FROM facturas
    WHERE estado = 'activa'
    GROUP BY id_cliente
),
compras AS (
    SELECT 
        pc.id_cliente,
        pc.cohorte,
        DATE_FORMAT(f.fecha_emision, '%Y-%m') AS mes_compra,
        TIMESTAMPDIFF(MONTH, CONCAT(pc.cohorte, '-01'), f.fecha_emision) AS mes_desde_cohorte,
        f.total
    FROM primer_compra pc
    JOIN facturas f ON pc.id_cliente = f.id_cliente AND f.estado = 'activa'
)
SELECT 
    cohorte,
    mes_desde_cohorte,
    COUNT(DISTINCT id_cliente) AS clientes_activos,
    SUM(total) AS ingresos,
    ROUND(COUNT(DISTINCT id_cliente) / NULLIF(FIRST_VALUE(COUNT(DISTINCT id_cliente)) OVER (
        PARTITION BY cohorte ORDER BY mes_desde_cohorte), 0) * 100, 2) AS retencion
FROM compras
WHERE mes_desde_cohorte >= 0
GROUP BY cohorte, mes_desde_cohorte
ORDER BY cohorte, mes_desde_cohorte;
```

### Análisis de Canasta de Compra (Market Basket)

```sql
-- Productos que frecuentemente se compran juntos
SELECT 
    p1.nombre_comercial AS producto_A,
    p2.nombre_comercial AS producto_B,
    COUNT(DISTINCT f1.id_factura) AS veces_juntos,
    (SELECT COUNT(DISTINCT id_factura) FROM facturas_detalle WHERE id_producto = p1.id_producto) AS veces_comprado_A,
    ROUND(
        COUNT(DISTINCT f1.id_factura) * 100.0 / 
        NULLIF((SELECT COUNT(DISTINCT id_factura) FROM facturas_detalle WHERE id_producto = p1.id_producto), 0), 2
    ) AS probabilidad_B_con_A
FROM facturas_detalle fd1
JOIN facturas f1 ON fd1.id_factura = f1.id_factura
JOIN productos p1 ON fd1.id_producto = p1.id_producto
JOIN facturas_detalle fd2 ON f1.id_factura = fd2.id_factura AND fd2.id_producto > fd1.id_producto
JOIN productos p2 ON fd2.id_producto = p2.id_producto
GROUP BY p1.id_producto, p2.id_producto
HAVING COUNT(DISTINCT f1.id_factura) >= 3
ORDER BY veces_juntos DESC
LIMIT 20;
```

### Detección de Anomalías en Ventas

```sql
-- Identificar días con ventas anormalmente altas o bajas (desviación > 2 sigma)
WITH stats_ventas AS (
    SELECT 
        DATE(fecha_emision) AS dia,
        SUM(total) AS ventas_dia
    FROM facturas
    WHERE estado = 'activa'
      AND fecha_emision >= DATE_SUB(CURRENT_DATE, INTERVAL 90 DAY)
    GROUP BY DATE(fecha_emision)
),
medidas AS (
    SELECT 
        AVG(ventas_dia) AS media,
        STDDEV(ventas_dia) AS desviacion
    FROM stats_ventas
)
SELECT 
    s.dia,
    DAYNAME(s.dia) AS dia_semana,
    s.ventas_dia,
    ROUND(m.media, 2) AS media_ventas,
    ROUND((s.ventas_dia - m.media) / NULLIF(m.desviacion, 0), 2) AS z_score,
    CASE 
        WHEN s.ventas_dia > m.media + 2 * m.desviacion THEN 'ANOMALÍA ALTA ⚠️'
        WHEN s.ventas_dia < m.media - 2 * m.desviacion THEN 'ANOMALÍA BAJA ⚠️'
        ELSE 'Normal'
    END AS estado
FROM stats_ventas s
CROSS JOIN medidas m
WHERE ABS((s.ventas_dia - m.media) / NULLIF(m.desviacion, 0)) > 2
ORDER BY ABS(s.ventas_dia - m.media) DESC;
```

### Forecast con Media Móvil y Tendencia

```sql
WITH ventas_diarias AS (
    SELECT 
        DATE(fecha_emision) AS dia,
        SUM(total) AS ventas
    FROM facturas
    WHERE estado = 'activa'
      AND fecha_emision >= DATE_SUB(CURRENT_DATE, INTERVAL 60 DAY)
    GROUP BY DATE(fecha_emision)
),
media_movil AS (
    SELECT 
        dia,
        ventas,
        AVG(ventas) OVER (ORDER BY dia ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS media_movil_7d
    FROM ventas_diarias
)
SELECT 
    dia,
    ventas,
    ROUND(media_movil_7d, 2) AS media_movil_7dias,
    ROUND(ventas - media_movil_7d, 2) AS ruido,
    ROUND(media_movil_7d * 1.10, 2) AS proyeccion_optimista,
    ROUND(media_movil_7d * 0.90, 2) AS proyeccion_pesimista,
    CASE 
        WHEN ROW_NUMBER() OVER (ORDER BY dia) = 1 THEN NULL
        ELSE ROUND(ventas - LAG(ventas) OVER (ORDER BY dia), 2)
    END AS cambio_diario
FROM media_movil
ORDER BY dia DESC;
```

### Análisis de Elasticidad de Precio

```sql
-- Correlación entre cambios de precio y volumen de ventas
SELECT 
    p.nombre_comercial,
    p.precio_venta AS precio_actual,
    AVG(fd.precio_unitario) AS precio_promedio_historico,
    SUM(fd.cantidad) / COUNT(DISTINCT DATE_FORMAT(f.fecha_emision, '%Y-%m')) AS unidades_promedio_mensual,
    ROUND(
        (SUM(fd.cantidad) / COUNT(DISTINCT DATE_FORMAT(f.fecha_emision, '%Y-%m'))) * p.precio_venta, 2
    ) AS ingresos_estimados_mensuales
FROM productos p
JOIN facturas_detalle fd ON p.id_producto = fd.id_producto
JOIN facturas f ON fd.id_factura = f.id_factura AND f.estado = 'activa'
GROUP BY p.id_producto
ORDER BY ingresos_estimados_mensuales DESC;
```

### KPIs de Ventas (Tablero Completo)

```sql
WITH metricas_hoy AS (
    SELECT 
        DATE(fecha_emision) AS dia,
        COUNT(DISTINCT id_factura) AS facturas,
        COUNT(DISTINCT id_cliente) AS clientes,
        SUM(total) AS ingresos,
        COUNT(DISTINCT CASE WHEN es_nuevo THEN id_cliente END) AS nuevos_clientes
    FROM facturas f
    LEFT JOIN (
        SELECT id_cliente, MIN(fecha_emision) AS primera_compra
        FROM facturas WHERE estado = 'activa' GROUP BY id_cliente
    ) pc ON f.id_cliente = pc.id_cliente AND f.fecha_emision = pc.primera_compra
    WHERE DATE(fecha_emision) = CURRENT_DATE AND estado = 'activa'
    GROUP BY DATE(fecha_emision)
),
metricas_mes AS (
    SELECT 
        'Mensual' AS periodo,
        COUNT(*) AS facturas_mes,
        COUNT(DISTINCT id_cliente) AS clientes_mes,
        SUM(total) AS ingresos_mes
    FROM facturas
    WHERE MONTH(fecha_emision) = MONTH(CURRENT_DATE)
      AND YEAR(fecha_emision) = YEAR(CURRENT_DATE)
      AND estado = 'activa'
)
SELECT 
    'Ingresos Hoy' AS kpi, ingresos FROM metricas_hoy
UNION ALL
SELECT 'Facturas Hoy', facturas FROM metricas_hoy
UNION ALL
SELECT 'Clientes Hoy', clientes FROM metricas_hoy
UNION ALL
SELECT 'Ticket Promedio Hoy', ROUND(ingresos / NULLIF(facturas, 0), 2) FROM metricas_hoy
UNION ALL
SELECT 'Ingresos del Mes', ingresos_mes FROM metricas_mes
UNION ALL
SELECT 'Facturas del Mes', facturas_mes FROM metricas_mes
UNION ALL
SELECT 'Clientes del Mes', clientes_mes FROM metricas_mes;
```

---

## Anterior: [02 Consultas Ventas](../03-ventas/02-consultas-ventas.md)
## Siguiente: [04 Reportes Ventas](../03-ventas/04-reportes-ventas.md)

[Volver al índice](../README.md)
