# 9.4 Dashboards con SQL

## Introducción

Los dashboards permiten visualizar KPIs y métricas de negocio en tiempo real. SQL es la base para extraer y preparar los datos.

## 1. Dashboard de Ventas

### KPI Cards
```sql
-- Consulta principal para dashboard de ventas
SELECT 
    -- Hoy
    (SELECT COUNT(*) FROM facturas WHERE DATE(fecha_emision) = CURRENT_DATE AND estado = 'activa') AS facturas_hoy,
    (SELECT COALESCE(SUM(total), 0) FROM facturas WHERE DATE(fecha_emision) = CURRENT_DATE AND estado = 'activa') AS ingresos_hoy,
    (SELECT COUNT(DISTINCT id_cliente) FROM facturas WHERE DATE(fecha_emision) = CURRENT_DATE AND estado = 'activa') AS clientes_hoy,
    (SELECT COALESCE(AVG(total), 0) FROM facturas WHERE DATE(fecha_emision) = CURRENT_DATE AND estado = 'activa') AS ticket_promedio_hoy,
    
    -- Este mes
    (SELECT COALESCE(SUM(total), 0) FROM facturas WHERE MONTH(fecha_emision) = MONTH(CURRENT_DATE) AND YEAR(fecha_emision) = YEAR(CURRENT_DATE) AND estado = 'activa') AS ingresos_mes,
    (SELECT COALESCE(SUM(total), 0) FROM facturas WHERE MONTH(fecha_emision) = MONTH(CURRENT_DATE) - 1 AND YEAR(fecha_emision) = YEAR(CURRENT_DATE) AND estado = 'activa') AS ingresos_mes_anterior,
    
    -- Variaciones
    ROUND(
        ((SELECT COALESCE(SUM(total), 0) FROM facturas WHERE DATE(fecha_emision) = CURRENT_DATE AND estado = 'activa') -
         (SELECT COALESCE(SUM(total), 0) FROM facturas WHERE DATE(fecha_emision) = DATE_SUB(CURRENT_DATE, INTERVAL 1 DAY) AND estado = 'activa'))
        / NULLIF((SELECT COALESCE(SUM(total), 0) FROM facturas WHERE DATE(fecha_emision) = DATE_SUB(CURRENT_DATE, INTERVAL 1 DAY) AND estado = 'activa'), 0) * 100, 2
    ) AS variacion_diaria;
```

### Gráfico de Ventas por Día (Últimos 30 Días)
```sql
SELECT 
    DATE(fecha_emision) AS dia,
    DAYNAME(fecha_emision) AS dia_semana,
    COUNT(*) AS facturas,
    SUM(total) AS ingresos,
    AVG(total) AS ticket_promedio,
    COUNT(DISTINCT id_cliente) AS clientes_unicos
FROM facturas
WHERE estado = 'activa'
  AND fecha_emision >= DATE_SUB(CURRENT_DATE, INTERVAL 30 DAY)
GROUP BY DATE(fecha_emision)
ORDER BY dia;
```

### Gráfico de Ventas por Hora (Hoy)
```sql
SELECT 
    HOUR(fecha_emision) AS hora,
    COUNT(*) AS transacciones,
    SUM(total) AS ingresos,
    AVG(total) AS ticket_promedio
FROM facturas
WHERE DATE(fecha_emision) = CURRENT_DATE AND estado = 'activa'
GROUP BY HOUR(fecha_emision)
ORDER BY hora;
```

## 2. Dashboard de Productos

### Top Productos Vendidos
```sql
SELECT 
    p.nombre_comercial,
    c.nombre AS categoria,
    SUM(fd.cantidad) AS unidades_vendidas,
    SUM(fd.total) AS ingresos,
    COUNT(DISTINCT f.id_factura) AS veces_vendido,
    ROUND(AVG(fd.precio_unitario), 2) AS precio_promedio
FROM productos p
JOIN categorias c ON p.id_categoria = c.id_categoria
JOIN facturas_detalle fd ON p.id_producto = fd.id_producto
JOIN facturas f ON fd.id_factura = f.id_factura AND f.estado = 'activa'
WHERE f.fecha_emision >= DATE_SUB(CURRENT_DATE, INTERVAL 30 DAY)
GROUP BY p.id_producto
ORDER BY ingresos DESC
LIMIT 10;
```

### Productos con Bajo Rendimiento
```sql
SELECT 
    p.nombre_comercial,
    p.stock_actual,
    COALESCE(SUM(fd.cantidad), 0) AS vendidos_30dias,
    DATEDIFF(CURRENT_DATE, COALESCE(MAX(f.fecha_emision), p.fecha_creacion)) AS dias_sin_venta,
    p.stock_actual * p.costo_promedio AS valor_inventario,
    CASE 
        WHEN COALESCE(SUM(fd.cantidad), 0) = 0 THEN 'SIN VENTAS'
        WHEN p.stock_actual / NULLIF(SUM(fd.cantidad), 0) > 12 THEN 'SOBRE STOCK'
        ELSE 'NORMAL'
    END AS estado
FROM productos p
LEFT JOIN facturas_detalle fd ON p.id_producto = fd.id_producto
LEFT JOIN facturas f ON fd.id_factura = f.id_factura AND f.estado = 'activa'
    AND f.fecha_emision >= DATE_SUB(CURRENT_DATE, INTERVAL 30 DAY)
WHERE p.activo = TRUE
GROUP BY p.id_producto
HAVING vendidos_30dias = 0 OR (p.stock_actual / NULLIF(vendidos_30dias, 0) > 12)
ORDER BY valor_inventario DESC;
```

## 3. Dashboard de Clientes

### Distribución de Clientes por Segmento
```sql
WITH rfm AS (
    SELECT 
        c.id_cliente,
        DATEDIFF(CURRENT_DATE, MAX(f.fecha_emision)) AS recencia,
        COUNT(DISTINCT f.id_factura) AS frecuencia,
        SUM(f.total) AS monetario,
        NTILE(5) OVER (ORDER BY DATEDIFF(CURRENT_DATE, MAX(f.fecha_emision)) ASC) AS r,
        NTILE(5) OVER (ORDER BY COUNT(DISTINCT f.id_factura) DESC) AS f,
        NTILE(5) OVER (ORDER BY SUM(f.total) DESC) AS m
    FROM clientes c
    JOIN facturas f ON c.id_cliente = f.id_cliente AND f.estado = 'activa'
    GROUP BY c.id_cliente
)
SELECT 
    CASE 
        WHEN r >= 4 AND f >= 4 AND m >= 4 THEN 'Campeones'
        WHEN r >= 4 AND f >= 3 AND m >= 3 THEN 'Leales'
        WHEN r >= 4 AND f <= 2 AND m <= 2 THEN 'Nuevos'
        WHEN r <= 2 AND f >= 3 AND m >= 3 THEN 'En Riesgo'
        WHEN r <= 2 AND f <= 2 AND m <= 2 THEN 'Perdidos'
        ELSE 'Regular'
    END AS segmento,
    COUNT(*) AS cantidad,
    ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER (), 1) AS porcentaje
FROM rfm
GROUP BY segmento
ORDER BY cantidad DESC;
```

### Nuevos Clientes por Mes
```sql
SELECT 
    DATE_FORMAT(f.fecha_emision, '%Y-%m') AS mes,
    COUNT(DISTINCT f.id_cliente) AS nuevos_clientes
FROM facturas f
WHERE f.fecha_emision = (
    SELECT MIN(f2.fecha_emision) 
    FROM facturas f2 
    WHERE f2.id_cliente = f.id_cliente AND f2.estado = 'activa'
)
GROUP BY DATE_FORMAT(f.fecha_emision, '%Y-%m')
ORDER BY mes;
```

## 4. Dashboard de Inventario

### Resumen de Inventario
```sql
SELECT 
    'Valor Total Inventario' AS kpi,
    CONCAT('$', FORMAT(SUM(stock_actual * costo_promedio), 2)) AS valor
FROM productos_stock
WHERE stock_actual > 0

UNION ALL

SELECT 
    'Total Unidades',
    FORMAT(SUM(stock_actual), 0)
FROM productos_stock

UNION ALL

SELECT 
    'Productos con Stock',
    FORMAT(COUNT(*), 0)
FROM productos_stock
WHERE stock_actual > 0

UNION ALL

SELECT 
    'Productos Stock Cero',
    FORMAT(COUNT(*), 0)
FROM productos_stock
WHERE stock_actual = 0

UNION ALL

SELECT 
    'Productos para Reordenar',
    FORMAT(COUNT(*), 0)
FROM productos_stock
WHERE stock_actual <= punto_reorden AND stock_actual > 0;
```

### Rotación de Inventario por Categoría
```sql
SELECT 
    c.nombre AS categoria,
    SUM(ps.stock_actual) AS stock_actual,
    SUM(COALESCE(v.vendidos, 0)) AS vendidos_anio,
    ROUND(SUM(COALESCE(v.vendidos, 0)) / NULLIF(SUM(ps.stock_actual), 0), 2) AS rotacion,
    CASE 
        WHEN SUM(COALESCE(v.vendidos, 0)) = 0 THEN 'SIN MOVIMIENTO'
        WHEN ROUND(SUM(COALESCE(v.vendidos, 0)) / NULLIF(SUM(ps.stock_actual), 0), 2) >= 6 THEN 'ALTA'
        WHEN ROUND(SUM(COALESCE(v.vendidos, 0)) / NULLIF(SUM(ps.stock_actual), 0), 2) >= 3 THEN 'MEDIA'
        ELSE 'BAJA'
    END AS nivel_rotacion
FROM categorias c
JOIN productos p ON c.id_categoria = p.id_categoria
JOIN productos_stock ps ON p.id_producto = ps.id_producto
LEFT JOIN (
    SELECT fd.id_producto, SUM(fd.cantidad) AS vendidos
    FROM facturas_detalle fd
    JOIN facturas f ON fd.id_factura = f.id_factura
    WHERE f.estado = 'activa' AND f.fecha_emision >= DATE_SUB(CURRENT_DATE, INTERVAL 12 MONTH)
    GROUP BY fd.id_producto
) v ON p.id_producto = v.id_producto
GROUP BY c.id_categoria
ORDER BY rotacion DESC;
```

## 5. Dashboard de Compras

### Órdenes de Compra por Estado
```sql
SELECT 
    estado,
    COUNT(*) AS cantidad,
    SUM(total) AS monto_total,
    ROUND(AVG(total), 2) AS monto_promedio,
    MIN(fecha_emision) AS primera,
    MAX(fecha_emision) AS ultima
FROM ordenes_compra
GROUP BY estado
ORDER BY FIELD(estado, 'borrador', 'pendiente_autorizacion', 'autorizada', 
               'enviada', 'confirmada', 'recibida_parcial', 'recibida_total', 'cancelada');
```

### Gasto Mensual en Compras
```sql
SELECT 
    DATE_FORMAT(fecha_emision, '%Y-%m') AS mes,
    COUNT(*) AS ordenes,
    SUM(total) AS gasto_total,
    AVG(total) AS promedio_por_orden,
    COUNT(DISTINCT id_proveedor) AS proveedores
FROM ordenes_compra
WHERE estado NOT IN ('borrador', 'cancelada')
  AND fecha_emision >= DATE_SUB(CURRENT_DATE, INTERVAL 12 MONTH)
GROUP BY DATE_FORMAT(fecha_emision, '%Y-%m')
ORDER BY mes;
```

## 6. Dashboard Ejecutivo Completo

```sql
-- Un solo query para el dashboard ejecutivo
SELECT 'VENTAS' AS seccion, 
       CONCAT('$', FORMAT(SUM(total), 2)) AS valor
FROM facturas 
WHERE DATE(fecha_emision) = CURRENT_DATE AND estado = 'activa'

UNION ALL

SELECT 'VENTAS_MES', 
       CONCAT('$', FORMAT(SUM(total), 2))
FROM facturas 
WHERE MONTH(fecha_emision) = MONTH(CURRENT_DATE) 
  AND YEAR(fecha_emision) = YEAR(CURRENT_DATE) 
  AND estado = 'activa'

UNION ALL

SELECT 'TICKET_PROMEDIO', 
       CONCAT('$', FORMAT(AVG(total), 2))
FROM facturas 
WHERE DATE(fecha_emision) = CURRENT_DATE AND estado = 'activa'

UNION ALL

SELECT 'VENTAS_ANO', 
       CONCAT('$', FORMAT(SUM(total), 2))
FROM facturas 
WHERE YEAR(fecha_emision) = YEAR(CURRENT_DATE) AND estado = 'activa'

UNION ALL

SELECT 'CLIENTES_ACTIVOS', 
       FORMAT(COUNT(DISTINCT id_cliente), 0)
FROM facturas 
WHERE DATE(fecha_emision) = CURRENT_DATE AND estado = 'activa'

UNION ALL

SELECT 'PRODUCTOS_VENDIDOS', 
       FORMAT(SUM(fd.cantidad), 0)
FROM facturas f
JOIN facturas_detalle fd ON f.id_factura = fd.id_factura
WHERE DATE(f.fecha_emision) = CURRENT_DATE AND f.estado = 'activa'

UNION ALL

SELECT 'VALOR_INVENTARIO', 
       CONCAT('$', FORMAT(SUM(ps.stock_actual * ps.costo_promedio), 2))
FROM productos_stock ps

UNION ALL

SELECT 'PRODUCTOS_STOCK_BAJO', 
       FORMAT(COUNT(*), 0)
FROM productos_stock
WHERE stock_actual <= punto_reorden AND stock_actual > 0

UNION ALL

SELECT 'CUENTAS_COBRAR', 
       CONCAT('$', FORMAT(SUM(total), 2))
FROM facturas 
WHERE metodo_pago = 'credito' AND estado = 'activa' AND fecha_vencimiento < CURRENT_DATE;
```

## Integración con Herramientas de BI

### Metabase
```sql
-- Consulta nativa para tarjeta de Metabase
SELECT 
    DATE(fecha_emision) AS dia,
    SUM(total) AS ingresos,
    COUNT(*) AS transacciones
FROM facturas
WHERE estado = 'activa'
  AND fecha_emision >= DATE_SUB(CURRENT_DATE, INTERVAL 30 DAY)
GROUP BY DATE(fecha_emision)
ORDER BY dia;
```

### Power BI / Tableau
```sql
-- Vista para conectar con Power BI
CREATE VIEW vw_dashboard_ventas AS
SELECT 
    f.id_factura,
    f.folio,
    f.fecha_emision,
    DATE_FORMAT(f.fecha_emision, '%Y-%m') AS año_mes,
    DATE_FORMAT(f.fecha_emision, '%Y-%m-%d') AS fecha,
    DAYNAME(f.fecha_emision) AS dia_semana,
    HOUR(f.fecha_emision) AS hora,
    c.id_cliente,
    c.nombre AS cliente,
    c.rfc,
    v.id_vendedor,
    v.nombre_vendedor AS vendedor,
    s.nombre AS sucursal,
    f.metodo_pago,
    f.estado,
    f.subtotal,
    f.descuento,
    f.iva,
    f.total,
    fd.id_producto,
    p.nombre_comercial AS producto,
    cat.nombre AS categoria,
    fd.cantidad,
    fd.precio_unitario,
    fd.total AS importe_linea
FROM facturas f
JOIN clientes c ON f.id_cliente = c.id_cliente
LEFT JOIN vendedores v ON f.id_vendedor = v.id_vendedor
LEFT JOIN sucursales s ON f.id_sucursal = s.id_sucursal
JOIN facturas_detalle fd ON f.id_factura = fd.id_factura
JOIN productos p ON fd.id_producto = p.id_producto
JOIN categorias cat ON p.id_categoria = cat.id_categoria;
```

---

## Anterior: [03 Segmentacion Clientes](../09-analisis-datos/03-segmentacion-clientes.md)
## Siguiente: [01 Glosario](../10-apendices/01-glosario.md)

[Volver al índice](../README.md)
