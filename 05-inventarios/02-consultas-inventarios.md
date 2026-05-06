# 5.2 Consultas de Inventarios

## Consultas de Stock

### Stock Actual por Producto
```sql
SELECT 
    p.sku,
    p.nombre_comercial,
    c.nombre AS categoria,
    ps.stock_actual,
    ps.stock_minimo,
    ps.stock_maximo,
    ps.punto_reorden,
    ps.stock_disponible,
    ps.costo_promedio,
    ROUND(ps.stock_actual * ps.costo_promedio, 2) AS valor_inventario
FROM productos p
JOIN categorias c ON p.id_categoria = c.id_categoria
LEFT JOIN productos_stock ps ON p.id_producto = ps.id_producto
WHERE p.activo = TRUE
ORDER BY p.nombre_comercial;
```

### Stock por Almacén
```sql
SELECT 
    a.nombre AS almacen,
    p.sku,
    p.nombre_comercial,
    ps.stock_actual,
    ps.stock_minimo,
    ps.punto_reorden
FROM almacenes a
JOIN productos_stock ps ON a.id_almacen = ps.id_almacen
JOIN productos p ON ps.id_producto = p.id_producto
WHERE p.activo = TRUE
ORDER BY a.nombre, p.nombre_comercial;
```

### Productos con Stock Bajo
```sql
SELECT 
    p.sku,
    p.nombre_comercial,
    c.nombre AS categoria,
    ps.stock_actual,
    ps.stock_minimo,
    ps.punto_reorden,
    ps.stock_actual - ps.punto_reorden AS diferencia,
    CASE 
        WHEN ps.stock_actual <= 0 THEN '🔴 SIN STOCK'
        WHEN ps.stock_actual <= ps.punto_reorden THEN '🟡 REORDENAR'
        WHEN ps.stock_actual <= ps.stock_minimo THEN '🟠 MÍNIMO'
        ELSE '🟢 OK'
    END AS indicador
FROM productos p
JOIN categorias c ON p.id_categoria = c.id_categoria
JOIN productos_stock ps ON p.id_producto = ps.id_producto
WHERE p.activo = TRUE
  AND ps.stock_actual <= ps.stock_minimo
ORDER BY ps.stock_actual ASC;
```

### Productos Sin Movimiento (Lento Movimiento)
```sql
SELECT 
    p.sku,
    p.nombre_comercial,
    c.nombre AS categoria,
    ps.stock_actual,
    ps.ultima_actualizacion,
    DATEDIFF(CURRENT_DATE, ps.fecha_ultima_venta) AS dias_sin_venta,
    DATEDIFF(CURRENT_DATE, ps.fecha_ultima_compra) AS dias_sin_compra,
    ps.stock_actual * ps.costo_promedio AS valor_inventario
FROM productos p
JOIN categorias c ON p.id_categoria = c.id_categoria
JOIN productos_stock ps ON p.id_producto = ps.id_producto
WHERE p.activo = TRUE
  AND ps.stock_actual > 0
  AND (
    ps.fecha_ultima_venta IS NULL 
    OR ps.fecha_ultima_venta < DATE_SUB(CURRENT_DATE, INTERVAL 90 DAY)
  )
ORDER BY dias_sin_venta DESC;
```

## Movimientos de Inventario

### Últimos Movimientos
```sql
SELECT 
    m.id_movimiento,
    DATE_FORMAT(m.fecha_movimiento, '%d/%m/%Y %H:%i') AS fecha,
    p.nombre_comercial AS producto,
    a.nombre AS almacen,
    tm.nombre AS tipo_movimiento,
    CASE WHEN m.cantidad > 0 THEN m.cantidad ELSE 0 END AS entrada,
    CASE WHEN m.cantidad < 0 THEN ABS(m.cantidad) ELSE 0 END AS salida,
    m.costo_unitario,
    m.cantidad * m.costo_unitario AS costo_total,
    m.existencia_antes,
    m.existencia_despues,
    m.referencia_tipo,
    m.referencia_id,
    m.usuario
FROM inventario_movimientos m
JOIN productos p ON m.id_producto = p.id_producto
LEFT JOIN almacenes a ON m.id_almacen = a.id_almacen
LEFT JOIN tipos_movimiento tm ON m.tipo_movimiento = tm.codigo
ORDER BY m.fecha_movimiento DESC
LIMIT 100;
```

### Kardex de un Producto
```sql
SELECT 
    DATE_FORMAT(m.fecha_movimiento, '%d/%m/%Y') AS fecha,
    m.tipo_movimiento,
    m.referencia_tipo,
    m.referencia_id,
    CASE WHEN m.cantidad > 0 THEN m.cantidad ELSE 0 END AS entrada,
    CASE WHEN m.cantidad < 0 THEN ABS(m.cantidad) ELSE 0 END AS salida,
    m.costo_unitario,
    m.cantidad * m.costo_unitario AS costo_movimiento,
    m.existencia_antes,
    m.existencia_despues,
    m.usuario,
    m.notas
FROM inventario_movimientos m
WHERE m.id_producto = 1
ORDER BY m.fecha_movimiento ASC;
```

### Movimientos por Tipo
```sql
SELECT 
    tm.nombre AS tipo_movimiento,
    DATE_FORMAT(m.fecha_movimiento, '%Y-%m') AS mes,
    COUNT(*) AS movimientos,
    SUM(ABS(m.cantidad)) AS unidades_afectadas,
    SUM(ABS(m.cantidad) * m.costo_unitario) AS costo_afectado
FROM inventario_movimientos m
JOIN tipos_movimiento tm ON m.tipo_movimiento = tm.codigo
WHERE m.fecha_movimiento >= DATE_SUB(CURRENT_DATE, INTERVAL 6 MONTH)
GROUP BY tm.codigo, DATE_FORMAT(m.fecha_movimiento, '%Y-%m')
ORDER BY tm.nombre, mes;
```

## Análisis de Inventario

### Exactitud del Inventario (Conteo Cíclico)
```sql
SELECT 
    p.sku,
    p.nombre_comercial,
    cc.cantidad_sistema,
    cc.cantidad_fisica,
    cc.diferencia,
    ROUND(cc.diferencia / NULLIF(cc.cantidad_sistema, 0) * 100, 2) AS porcentaje_error,
    cc.valor_diferencia,
    cc.estado,
    cc.fecha_conteo,
    cc.usuario_conteo
FROM conteos_ciclicos cc
JOIN productos p ON cc.id_producto = p.id_producto
ORDER BY ABS(cc.diferencia) DESC
LIMIT 50;
```

### Valor del Inventario Actual
```sql
SELECT 
    'Costo Total' AS tipo_valor,
    ROUND(SUM(ps.stock_actual * ps.costo_promedio), 2) AS valor
FROM productos_stock ps
JOIN productos p ON ps.id_producto = p.id_producto
WHERE p.activo = TRUE

UNION ALL

SELECT 
    'Venta Total',
    ROUND(SUM(ps.stock_actual * p.precio_venta), 2)
FROM productos_stock ps
JOIN productos p ON ps.id_producto = p.id_producto
WHERE p.activo = TRUE

UNION ALL

SELECT 
    'Margen Potencial',
    ROUND(SUM(ps.stock_actual * (p.precio_venta - ps.costo_promedio)), 2)
FROM productos_stock ps
JOIN productos p ON ps.id_producto = p.id_producto
WHERE p.activo = TRUE

UNION ALL

SELECT 
    'Productos con Stock',
    COUNT(DISTINCT id_producto)
FROM productos_stock
WHERE stock_actual > 0

UNION ALL

SELECT 
    'Unidades Totales',
    SUM(stock_actual)
FROM productos_stock;
```

### Conteo Cíclico por Categoría
```sql
SELECT 
    c.nombre AS categoria,
    COUNT(cc.id_conteo) AS conteos_realizados,
    SUM(CASE WHEN cc.diferencia = 0 THEN 1 ELSE 0 END) AS exactos,
    SUM(CASE WHEN cc.diferencia <> 0 THEN 1 ELSE 0 END) AS con_diferencia,
    ROUND(AVG(ABS(cc.diferencia) / NULLIF(cc.cantidad_sistema, 0) * 100), 2) AS error_promedio_porcentaje,
    ROUND(SUM(ABS(cc.valor_diferencia)), 2) AS valor_total_diferencias
FROM conteos_ciclicos cc
JOIN productos p ON cc.id_producto = p.id_producto
JOIN categorias c ON p.id_categoria = c.id_categoria
GROUP BY c.id_categoria;
```

### Transferencias entre Almacenes
```sql
SELECT 
    t.folio,
    ao.nombre AS almacen_origen,
    ad.nombre AS almacen_destino,
    t.fecha_creacion,
    t.fecha_envio,
    t.fecha_recepcion,
    t.estado,
    COUNT(td.id_detalle) AS productos,
    SUM(td.cantidad_enviada) AS unidades_enviadas,
    SUM(td.cantidad_recibida) AS unidades_recibidas,
    DATEDIFF(COALESCE(t.fecha_recepcion, NOW()), t.fecha_envio) AS dias_transito
FROM transferencias t
JOIN almacenes ao ON t.id_almacen_origen = ao.id_almacen
JOIN almacenes ad ON t.id_almacen_destino = ad.id_almacen
JOIN transferencias_detalle td ON t.id_transferencia = td.id_transferencia
WHERE t.estado NOT IN ('borrador', 'cancelada')
GROUP BY t.id_transferencia
ORDER BY t.fecha_creacion DESC;
```

## Dashboard de Inventario

```sql
-- KPIs de inventario
SELECT 
    'Valor del Inventario' AS kpi,
    CONCAT('$', FORMAT(SUM(ps.stock_actual * ps.costo_promedio), 2)) AS valor
FROM productos_stock ps

UNION ALL

SELECT 
    'Total Unidades',
    FORMAT(SUM(ps.stock_actual), 0)
FROM productos_stock ps

UNION ALL

SELECT 
    'Productos con Stock',
    COUNT(DISTINCT id_producto)
FROM productos_stock
WHERE stock_actual > 0

UNION ALL

SELECT 
    'Productos Stock Cero',
    COUNT(DISTINCT id_producto)
FROM productos_stock
WHERE stock_actual = 0

UNION ALL

SELECT 
    'Productos para Reordenar',
    COUNT(*)
FROM productos_stock
WHERE stock_actual <= punto_reorden AND stock_actual > 0

UNION ALL

SELECT 
    'Exactitud del Inventario',
    CONCAT(
        ROUND(
            SUM(CASE WHEN cc.diferencia = 0 THEN 1 ELSE 0 END) 
            / NULLIF(COUNT(*), 0) * 100, 2
        ), '%'
    )
FROM conteos_ciclicos
WHERE DATE(fecha_conteo) >= DATE_SUB(CURRENT_DATE, INTERVAL 30 DAY);
```

---

## Anterior: [01 Modelo Datos Inventarios](../05-inventarios/01-modelo-datos-inventarios.md)
## Siguiente: [03 Analisis Inventarios](../05-inventarios/03-analisis-inventarios.md)

[Volver al índice](../README.md)
