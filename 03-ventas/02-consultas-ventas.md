# 3.2 Consultas de Ventas

## Consultas Operativas del Día a Día

### Facturación del Día
```sql
-- Resumen de ventas del día actual
SELECT 
    DATE_FORMAT(fecha_emision, '%d/%m/%Y') AS fecha,
    COUNT(*) AS total_facturas,
    COUNT(DISTINCT id_cliente) AS clientes_atendidos,
    SUM(subtotal) AS subtotal,
    SUM(iva) AS iva,
    SUM(total) AS total,
    AVG(total) AS ticket_promedio
FROM facturas
WHERE DATE(fecha_emision) = CURRENT_DATE
  AND estado = 'activa';
```

### Ventas por Vendedor (Hoy)
```sql
SELECT 
    v.nombre_vendedor,
    COUNT(f.id_factura) AS facturas,
    SUM(f.total) AS monto_vendido,
    ROUND(AVG(f.total), 2) AS ticket_promedio,
    SUM(f.total) * v.comision_porcentaje / 100 AS comision_generada
FROM vendedores v
LEFT JOIN facturas f ON v.id_vendedor = f.id_vendedor 
    AND DATE(f.fecha_emision) = CURRENT_DATE
    AND f.estado = 'activa'
GROUP BY v.id_vendedor
ORDER BY monto_vendido DESC;
```

### Clientes que más compran
```sql
-- Top 10 clientes por monto de compra (últimos 30 días)
SELECT 
    c.nombre,
    c.email,
    COUNT(f.id_factura) AS compras,
    SUM(f.total) AS total_gastado,
    ROUND(AVG(f.total), 2) AS ticket_promedio,
    MAX(f.fecha_emision) AS ultima_compra
FROM clientes c
JOIN facturas f ON c.id_cliente = f.id_cliente
WHERE f.estado = 'activa'
  AND f.fecha_emision >= DATE_SUB(CURRENT_DATE, INTERVAL 30 DAY)
GROUP BY c.id_cliente
ORDER BY total_gastado DESC
LIMIT 10;
```

### Facturas Pendientes de Cobro
```sql
SELECT 
    f.folio,
    c.nombre AS cliente,
    f.fecha_emision,
    f.fecha_vencimiento,
    DATEDIFF(CURRENT_DATE, f.fecha_vencimiento) AS dias_vencido,
    f.total,
    COALESCE(SUM(fp.monto), 0) AS total_pagado,
    f.total - COALESCE(SUM(fp.monto), 0) AS saldo_pendiente
FROM facturas f
JOIN clientes c ON f.id_cliente = c.id_cliente
LEFT JOIN facturas_pagos fp ON f.id_factura = fp.id_factura AND fp.estatus = 'aplicado'
WHERE f.estado IN ('activa', 'pendiente_cobro')
  AND f.metodo_pago = 'credito'
GROUP BY f.id_factura
HAVING saldo_pendiente > 0
ORDER BY dias_vencido DESC;
```

### Historial de Compras de un Cliente
```sql
SELECT 
    f.folio,
    DATE_FORMAT(f.fecha_emision, '%d/%m/%Y') AS fecha,
    f.total,
    f.metodo_pago,
    f.estado,
    COUNT(fd.id_detalle) AS articulos,
    GROUP_CONCAT(CONCAT(fd.cantidad, 'x ', p.nombre_comercial) SEPARATOR ', ') AS productos
FROM facturas f
JOIN facturas_detalle fd ON f.id_factura = fd.id_factura
JOIN productos p ON fd.id_producto = p.id_producto
WHERE f.id_cliente = 1
GROUP BY f.id_factura
ORDER BY f.fecha_emision DESC;
```

## Análisis de Ventas

### Ventas por Periodo
```sql
-- Ventas diarias del mes actual
SELECT 
    DATE(fecha_emision) AS dia,
    DAYNAME(fecha_emision) AS dia_semana,
    COUNT(*) AS facturas,
    SUM(total) AS ingresos,
    SUM(total) - SUM(subtotal) AS iva_cobrado
FROM facturas
WHERE MONTH(fecha_emision) = MONTH(CURRENT_DATE)
  AND YEAR(fecha_emision) = YEAR(CURRENT_DATE)
  AND estado = 'activa'
GROUP BY DATE(fecha_emision)
ORDER BY dia;

-- Ventas mensuales del año
SELECT 
    MONTH(fecha_emision) AS mes_numero,
    DATE_FORMAT(fecha_emision, '%M') AS mes,
    COUNT(*) AS facturas,
    SUM(total) AS ingresos,
    SUM(total) - LAG(SUM(total)) OVER (ORDER BY MONTH(fecha_emision)) AS variacion_mensual,
    ROUND((SUM(total) - LAG(SUM(total)) OVER (ORDER BY MONTH(fecha_emision))) 
        / NULLIF(LAG(SUM(total)) OVER (ORDER BY MONTH(fecha_emision)), 0) * 100, 2) AS variacion_porcentaje
FROM facturas
WHERE YEAR(fecha_emision) = 2024
  AND estado = 'activa'
GROUP BY MONTH(fecha_emision)
ORDER BY mes_numero;
```

### Análisis por Método de Pago
```sql
SELECT 
    CASE metodo_pago
        WHEN 'efectivo' THEN 'Efectivo'
        WHEN 'tarjeta_credito' THEN 'Tarjeta de Crédito'
        WHEN 'tarjeta_debito' THEN 'Tarjeta de Débito'
        WHEN 'transferencia' THEN 'Transferencia Electrónica'
        WHEN 'credito' THEN 'Crédito'
        ELSE metodo_pago
    END AS metodo_pago,
    COUNT(*) AS transacciones,
    SUM(total) AS monto,
    ROUND(AVG(total), 2) AS promedio,
    COUNT(*) * 100.0 / SUM(COUNT(*)) OVER () AS porcentaje_transacciones,
    SUM(total) * 100.0 / SUM(SUM(total)) OVER () AS porcentaje_monto
FROM facturas
WHERE YEAR(fecha_emision) = 2024 AND estado = 'activa'
GROUP BY metodo_pago
ORDER BY monto DESC;
```

### Horas Pico de Ventas
```sql
SELECT 
    HOUR(fecha_emision) AS hora,
    COUNT(*) AS transacciones,
    SUM(total) AS ingresos,
    ROUND(AVG(total), 2) AS ticket_promedio
FROM facturas
WHERE YEAR(fecha_emision) = 2024 AND estado = 'activa'
GROUP BY HOUR(fecha_emision)
ORDER BY hora;
```

## Análisis de Productos

### Productos Más Vendidos
```sql
SELECT 
    p.nombre_comercial,
    c.nombre AS categoria,
    SUM(fd.cantidad) AS unidades_vendidas,
    SUM(fd.total) AS ingresos,
    ROUND(SUM(fd.total) / NULLIF(SUM(fd.cantidad), 0), 2) AS precio_promedio,
    COUNT(DISTINCT f.id_factura) AS facturas_que_lo_incluyen
FROM productos p
JOIN categorias c ON p.id_categoria = c.id_categoria
JOIN facturas_detalle fd ON p.id_producto = fd.id_producto
JOIN facturas f ON fd.id_factura = f.id_factura AND f.estado = 'activa'
GROUP BY p.id_producto
ORDER BY ingresos DESC
LIMIT 20;
```

### Productos con Menor Rotación
```sql
SELECT 
    p.nombre_comercial,
    c.nombre AS categoria,
    p.stock_actual,
    COALESCE(SUM(fd.cantidad), 0) AS unidades_vendidas_6meses,
    p.stock_actual / NULLIF(SUM(fd.cantidad), 0) AS meses_de_inventario
FROM productos p
JOIN categorias c ON p.id_categoria = c.id_categoria
LEFT JOIN facturas_detalle fd ON p.id_producto = fd.id_producto
LEFT JOIN facturas f ON fd.id_factura = f.id_factura 
    AND f.estado = 'activa'
    AND f.fecha_emision >= DATE_SUB(CURRENT_DATE, INTERVAL 6 MONTH)
GROUP BY p.id_producto
HAVING COALESCE(SUM(fd.cantidad), 0) = 0 OR meses_de_inventario > 3
ORDER BY COALESCE(SUM(fd.cantidad), 0) ASC;
```

### Análisis de Margen por Producto
```sql
SELECT 
    p.nombre_comercial,
    p.precio_compra,
    p.precio_venta,
    p.precio_venta - p.precio_compra AS margen_bruto,
    ROUND((p.precio_venta - p.precio_compra) / p.precio_venta * 100, 2) AS margen_porcentaje,
    SUM(fd.cantidad) AS unidades_vendidas,
    SUM(fd.cantidad * (p.precio_venta - p.precio_compra)) AS margen_total
FROM productos p
JOIN facturas_detalle fd ON p.id_producto = fd.id_producto
JOIN facturas f ON fd.id_factura = f.id_factura AND f.estado = 'activa'
GROUP BY p.id_producto
ORDER BY margen_total DESC;
```

## Dashboard de Ventas

```sql
-- Dashboard ejecutivo completo
SELECT 
    'Ventas Totales' AS metrica,
    SUM(total) AS valor
FROM facturas 
WHERE DATE(fecha_emision) = CURRENT_DATE AND estado = 'activa'

UNION ALL

SELECT 
    'Facturas Emitidas',
    COUNT(*)
FROM facturas 
WHERE DATE(fecha_emision) = CURRENT_DATE AND estado = 'activa'

UNION ALL

SELECT 
    'Ticket Promedio',
    ROUND(AVG(total), 2)
FROM facturas 
WHERE DATE(fecha_emision) = CURRENT_DATE AND estado = 'activa'

UNION ALL

SELECT 
    'Clientes Atendidos',
    COUNT(DISTINCT id_cliente)
FROM facturas 
WHERE DATE(fecha_emision) = CURRENT_DATE AND estado = 'activa'

UNION ALL

SELECT 
    'Productos Vendidos',
    SUM(fd.cantidad)
FROM facturas f
JOIN facturas_detalle fd ON f.id_factura = fd.id_factura
WHERE DATE(f.fecha_emision) = CURRENT_DATE AND f.estado = 'activa'

UNION ALL

SELECT 
    'Devoluciones del Día',
    COALESCE(SUM(total), 0)
FROM devoluciones
WHERE DATE(fecha_devolucion) = CURRENT_DATE AND estado = 'procesada';
```

### Comparativa vs Período Anterior
```sql
WITH ventas_periodo AS (
    SELECT 
        CASE 
            WHEN fecha_emision >= DATE_SUB(CURRENT_DATE, INTERVAL 30 DAY) THEN 'periodo_actual'
            WHEN fecha_emision >= DATE_SUB(CURRENT_DATE, INTERVAL 60 DAY) 
                 AND fecha_emision < DATE_SUB(CURRENT_DATE, INTERVAL 30 DAY) THEN 'periodo_anterior'
        END AS periodo,
        SUM(total) AS ventas,
        COUNT(*) AS facturas
    FROM facturas
    WHERE estado = 'activa'
      AND fecha_emision >= DATE_SUB(CURRENT_DATE, INTERVAL 60 DAY)
    GROUP BY periodo
)
SELECT 
    'Ventas' AS concepto,
    MAX(CASE WHEN periodo = 'periodo_anterior' THEN ventas END) AS periodo_anterior,
    MAX(CASE WHEN periodo = 'periodo_actual' THEN ventas END) AS periodo_actual,
    MAX(CASE WHEN periodo = 'periodo_actual' THEN ventas END) - 
    MAX(CASE WHEN periodo = 'periodo_anterior' THEN ventas END) AS variacion,
    ROUND(
        (MAX(CASE WHEN periodo = 'periodo_actual' THEN ventas END) - 
         MAX(CASE WHEN periodo = 'periodo_anterior' THEN ventas END)) 
        / NULLIF(MAX(CASE WHEN periodo = 'periodo_anterior' THEN ventas END), 0) * 100, 2
    ) AS variacion_porcentaje
FROM ventas_periodo;
```

## Reporte de Comisiones

```sql
SELECT 
    v.nombre_vendedor,
    COUNT(f.id_factura) AS facturas_realizadas,
    SUM(f.total) AS monto_vendido,
    v.comision_porcentaje,
    ROUND(SUM(f.total) * v.comision_porcentaje / 100, 2) AS comision,
    v.meta_mensual,
    ROUND(SUM(f.total) / v.meta_mensual * 100, 2) AS cumplimiento_meta
FROM vendedores v
LEFT JOIN facturas f ON v.id_vendedor = f.id_vendedor 
    AND f.estado = 'activa'
    AND MONTH(f.fecha_emision) = MONTH(CURRENT_DATE)
    AND YEAR(f.fecha_emision) = YEAR(CURRENT_DATE)
GROUP BY v.id_vendedor
ORDER BY comision DESC;
```

## Ejemplo: Ciclo Completo de una Venta

```sql
-- 1. Crear cotización
INSERT INTO cotizaciones (folio, id_cliente, id_vendedor, fecha_validez, subtotal, iva, total, estado)
VALUES ('COT-A00001', 1, 1, DATE_ADD(CURRENT_DATE, INTERVAL 15 DAY), 5000, 800, 5800, 'activa');

SET @id_cotizacion = LAST_INSERT_ID();

INSERT INTO cotizaciones_detalle (id_cotizacion, id_producto, cantidad, precio_unitario, subtotal, iva, total) VALUES
    (@id_cotizacion, 1, 2, 2500, 5000, 800, 5800);

-- 2. Cliente acepta → Convertir a factura
INSERT INTO facturas (folio, id_cliente, id_vendedor, id_orden_compra, subtotal, iva, total, metodo_pago, estado)
SELECT 'F-A00001', id_cliente, id_vendedor, NULL, subtotal, iva, total, 'transferencia', 'activa'
FROM cotizaciones WHERE id_cotizacion = @id_cotizacion;

SET @id_factura = LAST_INSERT_ID();

INSERT INTO facturas_detalle (id_factura, id_producto, cantidad, precio_unitario, subtotal, iva, total)
SELECT @id_factura, id_producto, cantidad, precio_unitario, subtotal, iva, total
FROM cotizaciones_detalle WHERE id_cotizacion = @id_cotizacion;

-- 3. Actualizar cotización
UPDATE cotizaciones SET estado = 'convertida' WHERE id_cotizacion = @id_cotizacion;

-- 4. Registrar pago
INSERT INTO facturas_pagos (id_factura, monto, forma_pago, referencia)
VALUES (@id_factura, 5800, 'TRA', 'TRF-20240115-001');

-- 5. Actualizar inventario (salida de productos)
INSERT INTO inventario_movimientos (id_producto, tipo_movimiento, cantidad, costo_unitario, referencia_tipo, referencia_id)
SELECT id_producto, 'salida_venta', -cantidad, p.precio_compra, 'factura', @id_factura
FROM facturas_detalle fd
JOIN productos p ON fd.id_producto = p.id_producto
WHERE fd.id_factura = @id_factura;

UPDATE productos_stock ps
JOIN facturas_detalle fd ON ps.id_producto = fd.id_producto
SET ps.stock_actual = ps.stock_actual - fd.cantidad,
    ps.ultima_actualizacion = CURRENT_TIMESTAMP
WHERE fd.id_factura = @id_factura;
```

---

## Anterior: [01 Modelo Datos Ventas](../03-ventas/01-modelo-datos-ventas.md)
## Siguiente: [03 Analisis Ventas](../03-ventas/03-analisis-ventas.md)

[Volver al índice](../README.md)
