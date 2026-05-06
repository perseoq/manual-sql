# 3.4 Reportes de Ventas

## Reportes Gerenciales

### Reporte Ejecutivo Diario
```sql
SELECT 
    CURRENT_DATE AS fecha_reporte,
    (SELECT COUNT(*) FROM facturas WHERE DATE(fecha_emision) = CURRENT_DATE AND estado = 'activa') AS facturas_hoy,
    (SELECT SUM(total) FROM facturas WHERE DATE(fecha_emision) = CURRENT_DATE AND estado = 'activa') AS ventas_hoy,
    (SELECT COUNT(DISTINCT id_cliente) FROM facturas WHERE DATE(fecha_emision) = CURRENT_DATE AND estado = 'activa') AS clientes_hoy,
    (SELECT COUNT(*) FROM facturas WHERE DATE(fecha_emision) = CURRENT_DATE AND estado = 'cancelada') AS cancelaciones_hoy,
    (SELECT SUM(total) FROM facturas WHERE DATE(fecha_emision) = CURRENT_DATE - INTERVAL 1 DAY AND estado = 'activa') AS ventas_ayer,
    (SELECT SUM(total) FROM facturas WHERE YEAR(fecha_emision) = YEAR(CURRENT_DATE) AND MONTH(fecha_emision) = MONTH(CURRENT_DATE) AND estado = 'activa') AS ventas_mes,
    (SELECT SUM(total) FROM facturas WHERE YEAR(fecha_emision) = YEAR(CURRENT_DATE) AND estado = 'activa') AS ventas_año;
```

### Reporte de Ventas por Sucursal
```sql
SELECT 
    s.nombre AS sucursal,
    COUNT(f.id_factura) AS facturas,
    SUM(f.total) AS ingresos,
    ROUND(AVG(f.total), 2) AS ticket_promedio,
    COUNT(DISTINCT f.id_cliente) AS clientes,
    MIN(f.fecha_emision) AS primera_venta_dia,
    MAX(f.fecha_emision) AS ultima_venta_dia
FROM sucursales s
LEFT JOIN facturas f ON s.id_sucursal = f.id_sucursal 
    AND DATE(f.fecha_emision) = CURRENT_DATE
    AND f.estado = 'activa'
GROUP BY s.id_sucursal
ORDER BY ingresos DESC;
```

### Reporte de Productos - Análisis ABC
```sql
WITH ventas_productos AS (
    SELECT 
        p.id_producto,
        p.nombre_comercial,
        c.nombre AS categoria,
        SUM(fd.cantidad) AS unidades,
        SUM(fd.total) AS ingresos,
        SUM(fd.total) / SUM(SUM(fd.total)) OVER () * 100 AS porcentaje_ingresos,
        SUM(SUM(fd.total)) OVER (ORDER BY SUM(fd.total) DESC) / SUM(SUM(fd.total)) OVER () * 100 AS porcentaje_acumulado
    FROM productos p
    JOIN categorias c ON p.id_categoria = c.id_categoria
    JOIN facturas_detalle fd ON p.id_producto = fd.id_producto
    JOIN facturas f ON fd.id_factura = f.id_factura AND f.estado = 'activa'
    WHERE f.fecha_emision >= DATE_SUB(CURRENT_DATE, INTERVAL 6 MONTH)
    GROUP BY p.id_producto
)
SELECT 
    id_producto,
    nombre_comercial,
    categoria,
    unidades,
    ROUND(ingresos, 2) AS ingresos,
    ROUND(porcentaje_ingresos, 2) AS porcentaje,
    ROUND(porcentaje_acumulado, 2) AS acumulado,
    CASE 
        WHEN porcentaje_acumulado <= 80 THEN 'A'
        WHEN porcentaje_acumulado <= 95 THEN 'B'
        ELSE 'C'
    END AS clasificacion_abc
FROM ventas_productos
ORDER BY ingresos DESC;
```

### Reporte de Devoluciones
```sql
SELECT 
    d.folio AS folio_devolucion,
    f.folio AS factura_original,
    c.nombre AS cliente,
    d.fecha_devolucion,
    d.tipo,
    d.total,
    d.motivo,
    d.estado,
    DATEDIFF(d.fecha_devolucion, f.fecha_emision) AS dias_desde_venta
FROM devoluciones d
JOIN facturas f ON d.id_factura_original = f.id_factura
JOIN clientes c ON f.id_cliente = c.id_cliente
WHERE d.fecha_devolucion >= DATE_SUB(CURRENT_DATE, INTERVAL 30 DAY)
ORDER BY d.fecha_devolucion DESC;
```

### Reporte de Cuentas por Cobrar
```sql
SELECT 
    c.nombre AS cliente,
    c.credito_limite,
    c.saldo_actual,
    c.credito_limite - c.saldo_actual AS credito_disponible,
    COUNT(CASE WHEN f.estado IN ('activa', 'pendiente_cobro') THEN f.id_factura END) AS facturas_pendientes,
    SUM(CASE WHEN f.estado IN ('activa', 'pendiente_cobro') THEN f.total END) AS total_pendiente,
    MIN(CASE WHEN f.estado IN ('activa', 'pendiente_cobro') THEN f.fecha_vencimiento END) AS proximo_vencimiento,
    SUM(CASE WHEN f.fecha_vencimiento < CURRENT_DATE AND f.estado IN ('activa', 'pendiente_cobro') THEN f.total ELSE 0 END) AS vencido
FROM clientes c
LEFT JOIN facturas f ON c.id_cliente = f.id_cliente
WHERE c.activo = TRUE
GROUP BY c.id_cliente
HAVING total_pendiente > 0 OR vencido > 0
ORDER BY vencido DESC;
```

### Reporte de Rentabilidad por Cliente
```sql
SELECT 
    c.nombre AS cliente,
    COUNT(f.id_factura) AS compras,
    SUM(f.total) AS ingresos,
    SUM(fd.cantidad * (fd.precio_unitario - p.precio_compra)) AS costo_ventas,
    SUM(f.total) - SUM(fd.cantidad * (fd.precio_unitario - p.precio_compra)) AS margen_bruto,
    ROUND((SUM(f.total) - SUM(fd.cantidad * (fd.precio_unitario - p.precio_compra))) / NULLIF(SUM(f.total), 0) * 100, 2) AS margen_porcentaje
FROM clientes c
JOIN facturas f ON c.id_cliente = f.id_cliente AND f.estado = 'activa'
JOIN facturas_detalle fd ON f.id_factura = fd.id_factura
JOIN productos p ON fd.id_producto = p.id_producto
GROUP BY c.id_cliente
ORDER BY margen_bruto DESC;
```

### Reporte de Metas vs Realidad
```sql
SELECT 
    v.nombre_vendedor,
    v.meta_mensual,
    COALESCE(SUM(f.total), 0) AS ventas_reales,
    ROUND(COALESCE(SUM(f.total), 0) / v.meta_mensual * 100, 2) AS cumplimiento_porcentaje,
    v.meta_mensual - COALESCE(SUM(f.total), 0) AS faltante,
    CASE 
        WHEN COALESCE(SUM(f.total), 0) >= v.meta_mensual THEN 'META ALCANZADA ✅'
        ELSE 'FALTANTE ⚠️'
    END AS estado_meta
FROM vendedores v
LEFT JOIN facturas f ON v.id_vendedor = f.id_vendedor 
    AND f.estado = 'activa'
    AND MONTH(f.fecha_emision) = MONTH(CURRENT_DATE)
    AND YEAR(f.fecha_emision) = YEAR(CURRENT_DATE)
GROUP BY v.id_vendedor
ORDER BY cumplimiento_porcentaje DESC;
```

### Reporte de Estacionalidad
```sql
SELECT 
    DAYNAME(fecha_emision) AS dia_semana,
    HOUR(fecha_emision) AS hora,
    COUNT(*) AS transacciones,
    SUM(total) AS ingresos,
    AVG(total) AS ticket_promedio,
    COUNT(DISTINCT id_cliente) AS clientes_unicos
FROM facturas
WHERE YEAR(fecha_emision) = 2024
  AND estado = 'activa'
GROUP BY DAYNAME(fecha_emision), HOUR(fecha_emision)
ORDER BY 
    FIELD(DAYNAME(fecha_emision), 'Monday', 'Tuesday', 'Wednesday', 'Thursday', 'Friday', 'Saturday', 'Sunday'),
    hora;
```

### Reporte de Tendencias (12 Meses)
```sql
SELECT 
    DATE_FORMAT(fecha_emision, '%Y-%m') AS periodo,
    COUNT(*) AS facturas,
    SUM(total) AS ingresos,
    SUM(total) - LAG(SUM(total)) OVER (ORDER BY DATE_FORMAT(fecha_emision, '%Y-%m')) AS diferencia_mensual,
    ROUND((SUM(total) - LAG(SUM(total)) OVER (ORDER BY DATE_FORMAT(fecha_emision, '%Y-%m'))) 
        / NULLIF(LAG(SUM(total)) OVER (ORDER BY DATE_FORMAT(fecha_emision, '%Y-%m')), 0) * 100, 2) AS variacion_porcentual,
    SUM(SUM(total)) OVER (ORDER BY DATE_FORMAT(fecha_emision, '%Y-%m')) AS ingresos_acumulados,
    COUNT(*) - LAG(COUNT(*)) OVER (ORDER BY DATE_FORMAT(fecha_emision, '%Y-%m')) AS diferencia_facturas
FROM facturas
WHERE YEAR(fecha_emision) = 2024
  AND estado = 'activa'
GROUP BY DATE_FORMAT(fecha_emision, '%Y-%m')
ORDER BY periodo;
```

### Reporte de Nuevos vs Recurrentes
```sql
WITH primera_compra AS (
    SELECT id_cliente, MIN(fecha_emision) AS primera_vez
    FROM facturas WHERE estado = 'activa' GROUP BY id_cliente
)
SELECT 
    DATE_FORMAT(f.fecha_emision, '%Y-%m') AS mes,
    COUNT(DISTINCT CASE WHEN f.fecha_emision = pc.primera_vez THEN f.id_cliente END) AS nuevos_clientes,
    COUNT(DISTINCT CASE WHEN f.fecha_emision > pc.primera_vez THEN f.id_cliente END) AS recurrentes,
    COUNT(DISTINCT f.id_cliente) AS total_clientes,
    ROUND(COUNT(DISTINCT CASE WHEN f.fecha_emision > pc.primera_vez THEN f.id_cliente END) 
        / NULLIF(COUNT(DISTINCT f.id_cliente), 0) * 100, 2) AS porcentaje_recurrentes
FROM facturas f
LEFT JOIN primera_compra pc ON f.id_cliente = pc.id_cliente
WHERE f.estado = 'activa'
GROUP BY DATE_FORMAT(f.fecha_emision, '%Y-%m')
ORDER BY mes;
```

### Reporte de Tasa de Conversión (Cotización → Venta)
```sql
SELECT 
    v.nombre_vendedor,
    COUNT(c.id_cotizacion) AS cotizaciones_realizadas,
    SUM(CASE WHEN c.estado = 'aceptada' THEN 1 ELSE 0 END) AS cotizaciones_aceptadas,
    SUM(CASE WHEN c.estado = 'convertida' THEN 1 ELSE 0 END) AS cotizaciones_convertidas,
    ROUND(SUM(CASE WHEN c.estado IN ('aceptada', 'convertida') THEN 1 ELSE 0 END) 
        / NULLIF(COUNT(c.id_cotizacion), 0) * 100, 2) AS tasa_conversion,
    ROUND(AVG(CASE WHEN c.estado IN ('aceptada', 'convertida') THEN c.total END), 2) AS ticket_promedio_convertido
FROM vendedores v
LEFT JOIN cotizaciones c ON v.id_vendedor = c.id_vendedor
    AND c.fecha_emision >= DATE_SUB(CURRENT_DATE, INTERVAL 30 DAY)
GROUP BY v.id_vendedor
ORDER BY tasa_conversion DESC;
```

---

## Anterior: [03 Analisis Ventas](../03-ventas/03-analisis-ventas.md)
## Siguiente: [01 Modelo Datos Compras](../04-compras/01-modelo-datos-compras.md)

[Volver al índice](../README.md)
