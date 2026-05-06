# 4.2 Consultas de Compras

## Consultas Operativas

### Órdenes de Compra En Proceso
```sql
SELECT 
    oc.folio,
    p.nombre_comercial AS proveedor,
    oc.fecha_emision,
    oc.fecha_estimada_entrega,
    DATEDIFF(oc.fecha_estimada_entrega, CURRENT_DATE) AS dias_restantes,
    oc.total,
    oc.estado,
    COUNT(ocd.id_detalle) AS productos,
    SUM(ocd.cantidad_solicitada) AS unidades_solicitadas,
    SUM(ocd.cantidad_recibida) AS unidades_recibidas,
    ROUND(SUM(ocd.cantidad_recibida) / NULLIF(SUM(ocd.cantidad_solicitada), 0) * 100, 2) AS porcentaje_recibido
FROM ordenes_compra oc
JOIN proveedores p ON oc.id_proveedor = p.id_proveedor
JOIN ordenes_compra_detalle ocd ON oc.id_orden = ocd.id_orden
WHERE oc.estado IN ('autorizada', 'enviada', 'confirmada', 'recibida_parcial')
GROUP BY oc.id_orden
ORDER BY oc.fecha_estimada_entrega ASC;
```

### Órdenes Atrasadas
```sql
SELECT 
    oc.folio,
    p.nombre_comercial AS proveedor,
    oc.fecha_emision,
    oc.fecha_estimada_entrega,
    DATEDIFF(CURRENT_DATE, oc.fecha_estimada_entrega) AS dias_atraso,
    oc.total,
    oc.estado,
    pc.nombre AS contacto,
    pc.telefono,
    pc.email
FROM ordenes_compra oc
JOIN proveedores p ON oc.id_proveedor = p.id_proveedor
LEFT JOIN proveedores_contactos pc ON p.id_proveedor = pc.id_proveedor AND pc.es_principal = TRUE
WHERE oc.fecha_estimada_entrega < CURRENT_DATE
  AND oc.estado IN ('enviada', 'confirmada', 'recibida_parcial')
ORDER BY dias_atraso DESC;
```

### Detalle de una Orden de Compra
```sql
SELECT 
    ocd.id_detalle,
    pr.sku,
    pr.nombre_comercial AS producto,
    ocd.cantidad_solicitada,
    ocd.cantidad_recibida,
    ocd.cantidad_pendiente,
    ocd.precio_unitario,
    ocd.descuento,
    ocd.subtotal,
    ocd.iva,
    ocd.total
FROM ordenes_compra_detalle ocd
JOIN productos pr ON ocd.id_producto = pr.id_producto
WHERE ocd.id_orden = 1001
ORDER BY ocd.id_detalle;
```

### Historial de Compras por Producto
```sql
SELECT 
    p.nombre_comercial,
    oc.folio AS orden_compra,
    oc.fecha_emision,
    ocd.cantidad_solicitada,
    ocd.cantidad_recibida,
    ocd.precio_unitario,
    ocd.total,
    pr.nombre_comercial AS proveedor,
    oc.estado
FROM ordenes_compra_detalle ocd
JOIN ordenes_compra oc ON ocd.id_orden = oc.id_orden
JOIN productos p ON ocd.id_producto = p.id_producto
JOIN proveedores pr ON oc.id_proveedor = pr.id_proveedor
WHERE p.id_producto = 1
ORDER BY oc.fecha_emision DESC;
```

### Evaluación de Proveedores
```sql
SELECT 
    p.nombre_comercial,
    COUNT(oc.id_orden) AS ordenes_totales,
    SUM(CASE WHEN oc.estado = 'recibida_total' THEN 1 ELSE 0 END) AS ordenes_completadas,
    ROUND(AVG(DATEDIFF(oc.fecha_real_entrega, oc.fecha_estimada_entrega))) AS desviacion_entrega_promedio,
    ROUND(AVG(oc.total), 2) AS monto_promedio_orden,
    SUM(oc.total) AS monto_total_compras,
    MIN(oc.fecha_emision) AS primera_compra,
    MAX(oc.fecha_emision) AS ultima_compra
FROM proveedores p
LEFT JOIN ordenes_compra oc ON p.id_proveedor = oc.id_proveedor
    AND oc.estado NOT IN ('borrador', 'cancelada')
GROUP BY p.id_proveedor
ORDER BY monto_total_compras DESC;
```

### Productos por Proveedor
```sql
SELECT 
    pr.nombre_comercial AS proveedor,
    p.sku,
    p.nombre_comercial AS producto,
    pp.codigo_proveedor,
    pp.precio_compra,
    pp.plazo_entrega,
    COALESCE((
        SELECT AVG(ocd.precio_unitario) 
        FROM ordenes_compra_detalle ocd
        JOIN ordenes_compra oc ON ocd.id_orden = oc.id_orden
        WHERE ocd.id_producto = p.id_producto AND oc.id_proveedor = pr.id_proveedor
    ), 0) AS precio_promedio_historico,
    (pp.precio_compra - COALESCE((
        SELECT AVG(ocd.precio_unitario) 
        FROM ordenes_compra_detalle ocd
        JOIN ordenes_compra oc ON ocd.id_orden = oc.id_orden
        WHERE ocd.id_producto = p.id_producto AND oc.id_proveedor = pr.id_proveedor
    ), 0)) / NULLIF(COALESCE((
        SELECT AVG(ocd.precio_unitario) 
        FROM ordenes_compra_detalle ocd
        JOIN ordenes_compra oc ON ocd.id_orden = oc.id_orden
        WHERE ocd.id_producto = p.id_producto AND oc.id_proveedor = pr.id_proveedor
    ), 0), 0) * 100 AS variacion_precio
FROM proveedores pr
JOIN proveedores_productos pp ON pr.id_proveedor = pp.id_proveedor
JOIN productos p ON pp.id_producto = p.id_producto
WHERE pr.activo = TRUE
ORDER BY pr.nombre_comercial, p.nombre_comercial;
```

## Análisis de Compras

### Gastos por Período
```sql
SELECT 
    DATE_FORMAT(oc.fecha_emision, '%Y-%m') AS mes,
    COUNT(oc.id_orden) AS ordenes,
    COUNT(DISTINCT oc.id_proveedor) AS proveedores_activos,
    SUM(oc.subtotal) AS compras_brutas,
    SUM(oc.gastos_envio) AS gastos_envio,
    SUM(oc.iva) AS iva_acreditable,
    SUM(oc.total) AS compras_totales,
    ROUND(AVG(oc.total), 2) AS orden_promedio
FROM ordenes_compra oc
WHERE oc.estado NOT IN ('borrador', 'cancelada')
  AND oc.fecha_emision >= DATE_SUB(CURRENT_DATE, INTERVAL 12 MONTH)
GROUP BY DATE_FORMAT(oc.fecha_emision, '%Y-%m')
ORDER BY mes DESC;
```

### Análisis de Precios (Mejor Cotización)
```sql
WITH precios_proveedores AS (
    SELECT 
        p.id_producto,
        p.nombre_comercial,
        pp.id_proveedor,
        pr.nombre_comercial AS proveedor,
        pp.precio_compra,
        pp.plazo_entrega,
        ROW_NUMBER() OVER (PARTITION BY p.id_producto ORDER BY pp.precio_compra ASC) AS ranking_precio
    FROM productos p
    JOIN proveedores_productos pp ON p.id_producto = pp.id_producto
    JOIN proveedores pr ON pp.id_proveedor = pr.id_proveedor
    WHERE pr.activo = TRUE
)
SELECT 
    nombre_comercial AS producto,
    MAX(CASE WHEN ranking_precio = 1 THEN proveedor END) AS mejor_proveedor_precio,
    MAX(CASE WHEN ranking_precio = 1 THEN precio_compra END) AS mejor_precio,
    MAX(CASE WHEN ranking_precio = 2 THEN precio_compra END) AS segundo_mejor_precio,
    MAX(CASE WHEN ranking_precio = 1 THEN plazo_entrega END) AS plazo_entrega_mejor,
    COUNT(*) AS total_proveedores
FROM precios_proveedores
GROUP BY id_producto
ORDER BY producto;
```

### Rotación de Proveedores
```sql
SELECT 
    p.nombre_comercial AS proveedor,
    DATE_FORMAT(oc.fecha_emision, '%Y-%m') AS mes_compra,
    COUNT(oc.id_orden) AS ordenes,
    SUM(oc.total) AS monto,
    ROW_NUMBER() OVER (PARTITION BY p.id_proveedor ORDER BY DATE_FORMAT(oc.fecha_emision, '%Y-%m') DESC) AS meses_atras
FROM proveedores p
JOIN ordenes_compra oc ON p.id_proveedor = oc.id_proveedor
WHERE oc.estado NOT IN ('borrador', 'cancelada')
  AND oc.fecha_emision >= DATE_SUB(CURRENT_DATE, INTERVAL 12 MONTH)
GROUP BY p.id_proveedor, DATE_FORMAT(oc.fecha_emision, '%Y-%m')
ORDER BY proveedor, mes_compra;
```

## Dashboard de Compras

```sql
-- KPIs de Compras
SELECT 
    'Órdenes Abiertas' AS indicador,
    COUNT(*) AS valor
FROM ordenes_compra
WHERE estado IN ('autorizada', 'enviada', 'confirmada')

UNION ALL

SELECT 
    'Órdenes en Riesgo (atrasadas)',
    COUNT(*)
FROM ordenes_compra
WHERE estado IN ('enviada', 'confirmada')
  AND fecha_estimada_entrega < CURRENT_DATE

UNION ALL

SELECT 
    'Proveedores Activos',
    COUNT(*)
FROM proveedores
WHERE activo = TRUE

UNION ALL

SELECT 
    'Gasto del Mes',
    COALESCE(SUM(total), 0)
FROM ordenes_compra
WHERE MONTH(fecha_emision) = MONTH(CURRENT_DATE)
  AND YEAR(fecha_emision) = YEAR(CURRENT_DATE)
  AND estado NOT IN ('borrador', 'cancelada')

UNION ALL

SELECT 
    'Productos para Reordenar',
    COUNT(*)
FROM vista_punto_reorden
WHERE prioridad IN ('URGENTE', 'REORDENAR');
```

---

## Anterior: [01 Modelo Datos Compras](../04-compras/01-modelo-datos-compras.md)
## Siguiente: [03 Analisis Compras](../04-compras/03-analisis-compras.md)

[Volver al índice](../README.md)
