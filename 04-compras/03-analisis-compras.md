# 4.3 Análisis de Compras

## Análisis de Eficiencia de Compras

### Tiempo de Entrega por Proveedor
```sql
SELECT 
    p.nombre_comercial AS proveedor,
    COUNT(oc.id_orden) AS ordenes_recibidas,
    ROUND(AVG(DATEDIFF(oc.fecha_real_entrega, oc.fecha_emision)), 1) AS dias_promedio_entrega,
    ROUND(AVG(oc.plazo_entrega_promedio), 1) AS plazo_prometido,
    ROUND(AVG(DATEDIFF(oc.fecha_real_entrega, oc.fecha_estimada_entrega)), 1) AS desviacion_promedio,
    SUM(CASE WHEN oc.fecha_real_entrega <= oc.fecha_estimada_entrega THEN 1 ELSE 0 END) AS entregas_a_tiempo,
    ROUND(
        SUM(CASE WHEN oc.fecha_real_entrega <= oc.fecha_estimada_entrega THEN 1 ELSE 0 END) 
        / NULLIF(COUNT(oc.id_orden), 0) * 100, 2
    ) AS cumplimiento_porcentaje
FROM proveedores p
JOIN ordenes_compra oc ON p.id_proveedor = oc.id_proveedor
WHERE oc.estado IN ('recibida_parcial', 'recibida_total', 'cerrada')
  AND oc.fecha_real_entrega IS NOT NULL
GROUP BY p.id_proveedor
ORDER BY cumplimiento_porcentaje DESC;
```

### Variación de Precios por Producto
```sql
WITH precios_historicos AS (
    SELECT 
        p.id_producto,
        p.nombre_comercial,
        pr.nombre_comercial AS proveedor,
        ocd.precio_unitario,
        oc.fecha_emision,
        LAG(ocd.precio_unitario) OVER (
            PARTITION BY p.id_producto, pr.id_proveedor 
            ORDER BY oc.fecha_emision
        ) AS precio_anterior,
        ROUND(
            (ocd.precio_unitario - LAG(ocd.precio_unitario) OVER (
                PARTITION BY p.id_producto, pr.id_proveedor 
                ORDER BY oc.fecha_emision
            )) / NULLIF(LAG(ocd.precio_unitario) OVER (
                PARTITION BY p.id_producto, pr.id_proveedor 
                ORDER BY oc.fecha_emision
            ), 0) * 100, 2
        ) AS variacion_porcentaje
    FROM ordenes_compra_detalle ocd
    JOIN ordenes_compra oc ON ocd.id_orden = oc.id_orden
    JOIN productos p ON ocd.id_producto = p.id_producto
    JOIN proveedores pr ON oc.id_proveedor = pr.id_proveedor
    WHERE oc.estado NOT IN ('borrador', 'cancelada')
)
SELECT 
    nombre_comercial,
    proveedor,
    precio_unitario AS ultimo_precio,
    precio_anterior,
    variacion_porcentaje,
    CASE 
        WHEN variacion_porcentaje > 10 THEN 'INCREMENTO ALTO ⚠️'
        WHEN variacion_porcentaje > 5 THEN 'INCREMENTO MODERADO'
        WHEN variacion_porcentaje < -5 THEN 'DECREMENTO'
        ELSE 'ESTABLE'
    END AS tendencia
FROM precios_historicos
WHERE precio_anterior IS NOT NULL
ORDER BY ABS(variacion_porcentaje) DESC
LIMIT 20;
```

### Análisis de Estacionalidad en Compras
```sql
SELECT 
    MONTH(fecha_emision) AS mes,
    COUNT(*) AS ordenes,
    SUM(total) AS gasto_total,
    AVG(total) AS gasto_promedio,
    COUNT(DISTINCT id_proveedor) AS proveedores_activos,
    ROUND(SUM(total) / SUM(SUM(total)) OVER () * 100, 2) AS porcentaje_anual
FROM ordenes_compra
WHERE YEAR(fecha_emision) = 2024
  AND estado NOT IN ('borrador', 'cancelada')
GROUP BY MONTH(fecha_emision)
ORDER BY mes;
```

### Análisis ABC de Proveedores
```sql
WITH gasto_proveedores AS (
    SELECT 
        p.id_proveedor,
        p.nombre_comercial,
        SUM(oc.total) AS gasto_total,
        SUM(oc.total) / SUM(SUM(oc.total)) OVER () * 100 AS porcentaje_gasto,
        SUM(SUM(oc.total)) OVER (ORDER BY SUM(oc.total) DESC) / SUM(SUM(oc.total)) OVER () * 100 AS porcentaje_acumulado
    FROM proveedores p
    JOIN ordenes_compra oc ON p.id_proveedor = oc.id_proveedor
    WHERE oc.estado NOT IN ('borrador', 'cancelada')
      AND oc.fecha_emision >= DATE_SUB(CURRENT_DATE, INTERVAL 12 MONTH)
    GROUP BY p.id_proveedor
)
SELECT 
    nombre_comercial,
    ROUND(gasto_total, 2) AS gasto_total,
    ROUND(porcentaje_gasto, 2) AS porcentaje,
    ROUND(porcentaje_acumulado, 2) AS acumulado,
    CASE 
        WHEN porcentaje_acumulado <= 80 THEN 'A (Crítico)'
        WHEN porcentaje_acumulado <= 95 THEN 'B (Importante)'
        ELSE 'C (Ocasional)'
    END AS clasificacion
FROM gasto_proveedores
ORDER BY gasto_total DESC;
```

### Ahorros por Negociación
```sql
WITH compras_recientes AS (
    SELECT 
        ocd.id_producto,
        ocd.id_orden,
        ocd.precio_unitario AS precio_pagado,
        oc.fecha_emision,
        pp.precio_compra AS precio_catalogo,
        ROUND((pp.precio_compra - ocd.precio_unitario) * ocd.cantidad_solicitada, 2) AS ahorro,
        ROUND((pp.precio_compra - ocd.precio_unitario) / NULLIF(pp.precio_compra, 0) * 100, 2) AS descuento_obtenido
    FROM ordenes_compra_detalle ocd
    JOIN ordenes_compra oc ON ocd.id_orden = oc.id_orden
    JOIN proveedores_productos pp ON ocd.id_producto = pp.id_producto 
        AND oc.id_proveedor = pp.id_proveedor
    WHERE oc.estado NOT IN ('borrador', 'cancelada')
      AND oc.fecha_emision >= DATE_SUB(CURRENT_DATE, INTERVAL 6 MONTH)
)
SELECT 
    ROUND(SUM(ahorro), 2) AS ahorro_total,
    ROUND(AVG(descuento_obtenido), 2) AS descuento_promedio,
    COUNT(*) AS lineas_negociadas,
    ROUND(SUM(ahorro) / NULLIF(COUNT(*), 0), 2) AS ahorro_promedio_por_linea
FROM compras_recientes;
```

## Proyección de Compras

### Productos para Reordenar (con demanda histórica)
```sql
WITH demanda_historica AS (
    SELECT 
        fd.id_producto,
        SUM(fd.cantidad) / 6 AS demanda_mensual_promedio
    FROM facturas_detalle fd
    JOIN facturas f ON fd.id_factura = f.id_factura
    WHERE f.estado = 'activa'
      AND f.fecha_emision >= DATE_SUB(CURRENT_DATE, INTERVAL 6 MONTH)
    GROUP BY fd.id_producto
),
stock_actual_por_producto AS (
    SELECT 
        id_producto,
        SUM(stock_actual) AS stock_total
    FROM productos_stock
    GROUP BY id_producto
)
SELECT 
    p.id_producto,
    p.sku,
    p.nombre_comercial,
    COALESCE(s.stock_total, 0) AS stock_actual,
    ROUND(dh.demanda_mensual_promedio, 2) AS demanda_mensual,
    ROUND(COALESCE(s.stock_total, 0) / NULLIF(dh.demanda_mensual_promedio, 0), 2) AS meses_de_inventario,
    ps.punto_reorden,
    CASE 
        WHEN COALESCE(s.stock_total, 0) <= ps.punto_reorden THEN 'REORDENAR'
        ELSE 'OK'
    END AS estado_reorden,
    ROUND(GREATEST(0, ps.punto_reorden - COALESCE(s.stock_total, 0) + dh.demanda_mensual_promedio), 0) AS cantidad_sugerida
FROM productos p
JOIN productos_stock ps ON p.id_producto = ps.id_producto
LEFT JOIN stock_actual_por_producto s ON p.id_producto = s.id_producto
LEFT JOIN demanda_historica dh ON p.id_producto = dh.id_producto
WHERE p.activo = TRUE AND p.controla_stock = TRUE
HAVING estado_reorden = 'REORDENAR'
ORDER BY (COALESCE(s.stock_total, 0) - ps.punto_reorden) ASC;
```

### Reporte de Eficiencia del Departamento de Compras
```sql
SELECT 
    DATE_FORMAT(oc.fecha_emision, '%Y-%m') AS mes,
    COUNT(*) AS ordenes_emitidas,
    COUNT(DISTINCT oc.id_proveedor) AS proveedores_gestionados,
    ROUND(AVG(oc.total), 2) AS promedio_por_orden,
    SUM(oc.total) AS gasto_total_gestionado,
    ROUND(AVG(DATEDIFF(oc.fecha_estimada_entrega, oc.fecha_emision)), 1) AS plazo_promedio_solicitado,
    ROUND(AVG(CASE 
        WHEN oc.fecha_real_entrega IS NOT NULL 
        THEN DATEDIFF(oc.fecha_real_entrega, oc.fecha_emision) 
        ELSE NULL 
    END), 1) AS plazo_promedio_real,
    ROUND(AVG(
        CASE WHEN oc.fecha_real_entrega <= oc.fecha_estimada_entrega THEN 1 ELSE 0 END
    ) * 100, 2) AS porcentaje_cumplimiento
FROM ordenes_compra oc
WHERE oc.estado NOT IN ('borrador', 'cancelada')
  AND oc.fecha_emision >= DATE_SUB(CURRENT_DATE, INTERVAL 12 MONTH)
GROUP BY DATE_FORMAT(oc.fecha_emision, '%Y-%m')
ORDER BY mes DESC;
```

---

## Anterior: [02 Consultas Compras](../04-compras/02-consultas-compras.md)
## Siguiente: [01 Modelo Datos Inventarios](../05-inventarios/01-modelo-datos-inventarios.md)

[Volver al índice](../README.md)
