# 5.3 Análisis de Inventarios

## Rotación de Inventarios

### Rotación por Producto
```sql
SELECT 
    p.id_producto,
    p.sku,
    p.nombre_comercial,
    c.nombre AS categoria,
    ps.stock_actual,
    ps.costo_promedio,
    COALESCE(SUM(fd.cantidad), 0) AS unidades_vendidas_anuales,
    ps.stock_actual * ps.costo_promedio AS valor_inventario,
    ROUND(
        COALESCE(SUM(fd.cantidad), 0) / NULLIF(ps.stock_actual, 0), 2
    ) AS rotacion,
    ROUND(
        COALESCE(SUM(fd.cantidad), 0) / NULLIF(ps.stock_actual, 0) * ps.costo_promedio, 2
    ) AS rotacion_valorizada,
    CASE 
        WHEN COALESCE(SUM(fd.cantidad), 0) = 0 THEN 'SIN VENTAS'
        WHEN ROUND(COALESCE(SUM(fd.cantidad), 0) / NULLIF(ps.stock_actual, 0), 2) >= 6 THEN 'ALTA ROTACIÓN'
        WHEN ROUND(COALESCE(SUM(fd.cantidad), 0) / NULLIF(ps.stock_actual, 0), 2) >= 3 THEN 'MEDIA ROTACIÓN'
        ELSE 'BAJA ROTACIÓN'
    END AS clasificacion_rotacion
FROM productos p
JOIN categorias c ON p.id_categoria = c.id_categoria
JOIN productos_stock ps ON p.id_producto = ps.id_producto
LEFT JOIN facturas_detalle fd ON p.id_producto = fd.id_producto
LEFT JOIN facturas f ON fd.id_factura = f.id_factura 
    AND f.estado = 'activa'
    AND f.fecha_emision >= DATE_SUB(CURRENT_DATE, INTERVAL 12 MONTH)
GROUP BY p.id_producto
ORDER BY rotacion DESC;
```

### Días de Inventario
```sql
WITH ventas_diarias_promedio AS (
    SELECT 
        fd.id_producto,
        SUM(fd.cantidad) / 365 AS ventas_diarias_prom
    FROM facturas_detalle fd
    JOIN facturas f ON fd.id_factura = f.id_factura
    WHERE f.estado = 'activa'
      AND f.fecha_emision >= DATE_SUB(CURRENT_DATE, INTERVAL 12 MONTH)
    GROUP BY fd.id_producto
)
SELECT 
    p.sku,
    p.nombre_comercial,
    ps.stock_actual,
    ROUND(vdp.ventas_diarias_prom, 2) AS ventas_diarias_promedio,
    ROUND(ps.stock_actual / NULLIF(vdp.ventas_diarias_prom, 0), 0) AS dias_inventario,
    CASE 
        WHEN ps.stock_actual > 0 AND (ps.stock_actual / NULLIF(vdp.ventas_diarias_prom, 0)) > 90 THEN 'EXCESO'
        WHEN ps.stock_actual > 0 AND (ps.stock_actual / NULLIF(vdp.ventas_diarias_prom, 0)) > 30 THEN 'NORMAL'
        WHEN ps.stock_actual > 0 THEN 'BAJO'
        ELSE 'SIN STOCK'
    END AS estado_inventario
FROM productos p
JOIN productos_stock ps ON p.id_producto = ps.id_producto
LEFT JOIN ventas_diarias_promedio vdp ON p.id_producto = vdp.id_producto
WHERE p.activo = TRUE
ORDER BY dias_inventario DESC;
```

## Análisis ABC (Valor de Inventario)

```sql
WITH valor_productos AS (
    SELECT 
        p.id_producto,
        p.sku,
        p.nombre_comercial,
        c.nombre AS categoria,
        ps.stock_actual,
        ps.costo_promedio,
        ps.stock_actual * ps.costo_promedio AS valor_total,
        ROW_NUMBER() OVER (ORDER BY ps.stock_actual * ps.costo_promedio DESC) AS ranking,
        SUM(ps.stock_actual * ps.costo_promedio) OVER () AS valor_total_inventario
    FROM productos p
    JOIN categorias c ON p.id_categoria = c.id_categoria
    JOIN productos_stock ps ON p.id_producto = ps.id_producto
    WHERE p.activo = TRUE AND ps.stock_actual > 0
)
SELECT 
    ranking,
    sku,
    nombre_comercial,
    categoria,
    stock_actual,
    ROUND(costo_promedio, 2) AS costo_unitario,
    ROUND(valor_total, 2) AS valor_inventario,
    ROUND(valor_total / valor_total_inventario * 100, 2) AS porcentaje_valor,
    ROUND(SUM(valor_total) OVER (ORDER BY ranking) / valor_total_inventario * 100, 2) AS porcentaje_acumulado,
    CASE 
        WHEN SUM(valor_total) OVER (ORDER BY ranking) / valor_total_inventario * 100 <= 80 THEN 'A'
        WHEN SUM(valor_total) OVER (ORDER BY ranking) / valor_total_inventario * 100 <= 95 THEN 'B'
        ELSE 'C'
    END AS clasificacion_abc
FROM valor_productos
ORDER BY ranking;
```

## Análisis de Obsolescencia

```sql
SELECT 
    p.sku,
    p.nombre_comercial,
    c.nombre AS categoria,
    ps.stock_actual,
    ps.costo_promedio,
    ps.stock_actual * ps.costo_promedio AS valor_inventario,
    ps.fecha_ultima_venta,
    DATEDIFF(CURRENT_DATE, ps.fecha_ultima_venta) AS dias_ultima_venta,
    ps.fecha_ultima_compra,
    DATEDIFF(CURRENT_DATE, ps.fecha_ultima_compra) AS dias_ultima_compra,
    CASE 
        WHEN ps.stock_actual > 0 AND ps.fecha_ultima_venta IS NULL THEN 'SIN VENTAS NUNCA'
        WHEN DATEDIFF(CURRENT_DATE, ps.fecha_ultima_venta) > 365 THEN 'OBSOLETO'
        WHEN DATEDIFF(CURRENT_DATE, ps.fecha_ultima_venta) > 180 THEN 'LENTA ROTACIÓN'
        WHEN DATEDIFF(CURRENT_DATE, ps.fecha_ultima_venta) > 90 THEN 'ROTACIÓN BAJA'
        ELSE 'NORMAL'
    END AS estado_obsolescencia
FROM productos p
JOIN categorias c ON p.id_categoria = c.id_categoria
JOIN productos_stock ps ON p.id_producto = ps.id_producto
WHERE p.activo = TRUE AND ps.stock_actual > 0
HAVING estado_obsolescencia IN ('SIN VENTAS NUNCA', 'OBSOLETO', 'LENTA ROTACIÓN')
ORDER BY dias_ultima_venta DESC;
```

## Análisis de Mermas y Ajustes

```sql
SELECT 
    DATE_FORMAT(m.fecha_movimiento, '%Y-%m') AS mes,
    m.tipo_movimiento,
    c.nombre AS categoria,
    COUNT(*) AS eventos,
    SUM(ABS(m.cantidad)) AS unidades_afectadas,
    SUM(ABS(m.cantidad) * m.costo_unitario) AS valor_afectado,
    AVG(ABS(m.cantidad)) AS promedio_por_evento,
    MAX(ABS(m.cantidad)) AS maximo_por_evento
FROM inventario_movimientos m
JOIN productos p ON m.id_producto = p.id_producto
JOIN categorias c ON p.id_categoria = c.id_categoria
WHERE m.tipo_movimiento IN ('merma', 'ajuste_salida', 'ajuste_entrada')
  AND m.fecha_movimiento >= DATE_SUB(CURRENT_DATE, INTERVAL 12 MONTH)
GROUP BY DATE_FORMAT(m.fecha_movimiento, '%Y-%m'), m.tipo_movimiento, c.id_categoria
ORDER BY mes DESC, valor_afectado DESC;
```

## Proyección de Inventario

```sql
WITH demanda AS (
    SELECT 
        fd.id_producto,
        DATE_FORMAT(f.fecha_emision, '%Y-%m') AS mes,
        SUM(fd.cantidad) AS demanda_mensual
    FROM facturas_detalle fd
    JOIN facturas f ON fd.id_factura = f.id_factura
    WHERE f.estado = 'activa'
      AND f.fecha_emision >= DATE_SUB(CURRENT_DATE, INTERVAL 6 MONTH)
    GROUP BY fd.id_producto, DATE_FORMAT(f.fecha_emision, '%Y-%m')
),
demanda_promedio AS (
    SELECT 
        id_producto,
        AVG(demanda_mensual) AS demanda_prom_mensual,
        STDDEV(demanda_mensual) AS desviacion_demanda
    FROM demanda
    GROUP BY id_producto
)
SELECT 
    p.sku,
    p.nombre_comercial,
    ps.stock_actual,
    ROUND(dp.demanda_prom_mensual, 2) AS demanda_promedio_mensual,
    ROUND(dp.desviacion_demanda, 2) AS desviacion_demanda,
    ROUND(ps.stock_actual / NULLIF(dp.demanda_prom_mensual, 0), 2) AS meses_cobertura,
    ROUND(GREATEST(0, dp.demanda_prom_mensual - ps.stock_actual), 0) AS cantidad_reorden_sugerida,
    ps.punto_reorden,
    CASE 
        WHEN ps.stock_actual <= ps.punto_reorden THEN 'COMPRAR'
        ELSE 'OK'
    END AS accion
FROM productos p
JOIN productos_stock ps ON p.id_producto = ps.id_producto
LEFT JOIN demanda_promedio dp ON p.id_producto = dp.id_producto
WHERE p.activo = TRUE
ORDER BY ps.stock_actual ASC;
```

## Dashboard de Análisis de Inventario

```sql
-- Resumen ejecutivo
SELECT 
    'Valor Total Inventario' AS metrica,
    ROUND(SUM(ps.stock_actual * ps.costo_promedio), 2) AS valor
FROM productos_stock ps
WHERE ps.stock_actual > 0

UNION ALL

SELECT 
    'Días de Inventario Promedio',
    ROUND(AVG(dias_inventario), 0)
FROM (
    SELECT 
        ps.id_producto,
        ps.stock_actual / NULLIF(SUM(fd.cantidad) / 365, 0) AS dias_inventario
    FROM productos_stock ps
    JOIN facturas_detalle fd ON ps.id_producto = fd.id_producto
    JOIN facturas f ON fd.id_factura = f.id_factura AND f.estado = 'activa'
    WHERE f.fecha_emision >= DATE_SUB(CURRENT_DATE, INTERVAL 12 MONTH)
    GROUP BY ps.id_producto
) AS sub

UNION ALL

SELECT 
    '% Valor en Clasificación A',
    CONCAT(ROUND(porcentaje_acumulado_80, 2), '%')
FROM (
    SELECT 
        SUM(CASE WHEN acumulado_porcentaje <= 80 THEN valor ELSE 0 END) / SUM(valor) * 100 AS porcentaje_acumulado_80
    FROM (
        SELECT 
            ps.id_producto,
            ps.stock_actual * ps.costo_promedio AS valor,
            SUM(ps.stock_actual * ps.costo_promedio) OVER (ORDER BY ps.stock_actual * ps.costo_promedio DESC) / SUM(ps.stock_actual * ps.costo_promedio) OVER () * 100 AS acumulado_porcentaje
        FROM productos_stock ps
        WHERE ps.stock_actual > 0
    ) AS sub2
) AS sub3

UNION ALL

SELECT 
    '% de Exactitud de Inventario',
    CONCAT(ROUND(
        SUM(CASE WHEN cc.diferencia = 0 THEN 1 ELSE 0 END) / NULLIF(COUNT(*), 0) * 100, 2
    ), '%')
FROM conteos_ciclicos cc
WHERE DATE(cc.fecha_conteo) >= DATE_SUB(CURRENT_DATE, INTERVAL 30 DAY);
```

---

## Anterior: [02 Consultas Inventarios](../05-inventarios/02-consultas-inventarios.md)
## Siguiente: [04 Kardex Y Valorizacion](../05-inventarios/04-kardex-y-valorizacion.md)

[Volver al índice](../README.md)
