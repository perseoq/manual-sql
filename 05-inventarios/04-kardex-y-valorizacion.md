# 5.4 Kardex y Valorización

## Introducción al Kardex

El Kardex es un registro detallado de todas las entradas y salidas de cada producto, mostrando el saldo actualizado y el costo unitario según el método de valorización.

## Métodos de Valorización

### 1. PP (Promedio Ponderado)
```sql
-- Calcular costo promedio ponderado
CREATE PROCEDURE sp_actualizar_costo_promedio(
    IN p_id_producto INT,
    IN p_id_almacen INT,
    IN p_cantidad_nueva DECIMAL(12,4),
    IN p_costo_nuevo DECIMAL(12,4)
)
BEGIN
    DECLARE v_stock_actual DECIMAL(12,4);
    DECLARE v_costo_actual DECIMAL(12,4);
    DECLARE v_nuevo_costo DECIMAL(12,4);
    
    SELECT stock_actual, costo_promedio 
    INTO v_stock_actual, v_costo_actual
    FROM productos_stock
    WHERE id_producto = p_id_producto AND id_almacen = p_id_almacen;
    
    -- Fórmula del promedio ponderado
    SET v_nuevo_costo = (v_costo_actual * v_stock_actual + p_costo_nuevo * p_cantidad_nueva) 
                       / NULLIF(v_stock_actual + p_cantidad_nueva, 0);
    
    UPDATE productos_stock
    SET costo_promedio = v_nuevo_costo
    WHERE id_producto = p_id_producto AND id_almacen = p_id_almacen;
END //
```

### 2. PEPS (Primeras Entradas, Primeras Salidas)
```sql
-- Kardex PEPS: Las primeras unidades que entraron son las primeras en salir
CREATE VIEW vista_kardex_peps AS
WITH entradas_acumuladas AS (
    SELECT 
        m.id_movimiento,
        m.id_producto,
        m.fecha_movimiento,
        m.tipo_movimiento,
        CASE WHEN m.cantidad > 0 THEN m.cantidad ELSE 0 END AS entrada,
        m.costo_unitario,
        SUM(CASE WHEN m.cantidad > 0 THEN m.cantidad ELSE 0 END) OVER (
            PARTITION BY m.id_producto 
            ORDER BY m.fecha_movimiento, m.id_movimiento
        ) AS entradas_acum,
        SUM(CASE WHEN m.cantidad < 0 THEN ABS(m.cantidad) ELSE 0 END) OVER (
            PARTITION BY m.id_producto 
            ORDER BY m.fecha_movimiento, m.id_movimiento
        ) AS salidas_acum
    FROM inventario_movimientos m
    WHERE m.tipo_movimiento IN ('entrada_compra', 'salida_venta', 'inventario_inicial')
)
SELECT 
    ea.id_producto,
    p.nombre_comercial,
    ea.fecha_movimiento,
    ea.tipo_movimiento,
    ea.entrada,
    ea.costo_unitario AS costo_unitario_entrada,
    CASE WHEN ea.entrada = 0 THEN 
        -- Calcular costo PEPS para salidas
        (SELECT ea2.costo_unitario 
         FROM entradas_acumuladas ea2 
         WHERE ea2.id_producto = ea.id_producto
           AND ea2.entradas_acum > ea.salidas_acum - ABS(
               CASE WHEN ea.entrada = 0 THEN 
                   (SELECT SUM(CASE WHEN m3.cantidad < 0 THEN ABS(m3.cantidad) ELSE 0 END)
                    FROM inventario_movimientos m3
                    WHERE m3.id_producto = ea.id_producto 
                      AND m3.fecha_movimiento <= ea.fecha_movimiento
                      AND m3.tipo_movimiento = 'salida_venta')
               ELSE 0 END
           )
         ORDER BY ea2.entradas_acum
         LIMIT 1)
    ELSE NULL END AS costo_unitario_salida_peps
FROM entradas_acumuladas ea
JOIN productos p ON ea.id_producto = p.id_producto
ORDER BY ea.id_producto, ea.fecha_movimiento;
```

### 3. UEPS (Últimas Entradas, Primeras Salidas)
```sql
-- Similar a PEPS pero ordenando por fecha descendente para las salidas
-- (No recomendado fiscalmente en muchos países, pero útil para análisis interno)
```

## Kardex Completo con Todos los Métodos

```sql
CREATE VIEW vista_kardex_completo AS
SELECT 
    m.fecha_movimiento,
    m.tipo_movimiento,
    p.sku,
    p.nombre_comercial,
    a.nombre AS almacen,
    CASE WHEN m.cantidad > 0 THEN m.cantidad ELSE 0 END AS entrada,
    CASE WHEN m.cantidad < 0 THEN ABS(m.cantidad) ELSE 0 END AS salida,
    m.costo_unitario,
    m.cantidad * m.costo_unitario AS costo_movimiento,
    -- Saldo en cantidades (acumulado)
    SUM(m.cantidad) OVER (
        PARTITION BY m.id_producto, m.id_almacen 
        ORDER BY m.fecha_movimiento, m.id_movimiento
    ) AS saldo_cantidad,
    -- Saldo en valores (acumulado usando promedio)
    SUM(m.cantidad * m.costo_unitario) OVER (
        PARTITION BY m.id_producto, m.id_almacen 
        ORDER BY m.fecha_movimiento, m.id_movimiento
    ) AS saldo_valor,
    -- Costo promedio (saldo valor / saldo cantidad)
    CASE 
        WHEN SUM(m.cantidad) OVER (
            PARTITION BY m.id_producto, m.id_almacen 
            ORDER BY m.fecha_movimiento, m.id_movimiento
        ) > 0 
        THEN ROUND(
            SUM(m.cantidad * m.costo_unitario) OVER (
                PARTITION BY m.id_producto, m.id_almacen 
                ORDER BY m.fecha_movimiento, m.id_movimiento
            ) / NULLIF(SUM(m.cantidad) OVER (
                PARTITION BY m.id_producto, m.id_almacen 
                ORDER BY m.fecha_movimiento, m.id_movimiento
            ), 0), 4
        )
        ELSE 0
    END AS costo_promedio_momento,
    m.referencia_tipo,
    m.referencia_id,
    m.usuario
FROM inventario_movimientos m
JOIN productos p ON m.id_producto = p.id_producto
LEFT JOIN almacenes a ON m.id_almacen = a.id_almacen
ORDER BY p.nombre_comercial, m.fecha_movimiento;

-- Consultar kardex de un producto específico
SELECT * FROM vista_kardex_completo
WHERE sku = 'LAPTOP-HP-001'
ORDER BY fecha_movimiento;
```

## Reporte de Valorización de Inventario

```sql
CREATE PROCEDURE sp_reporte_valorizacion_inventario(
    IN p_fecha_corte DATE
)
BEGIN
    -- Obtener el costo de cada producto a la fecha de corte
    -- usando el costo promedio actual
    SELECT 
        c.nombre AS categoria,
        p.sku,
        p.nombre_comercial,
        p.unidad_medida,
        ps.stock_actual,
        ps.costo_promedio,
        ps.stock_actual * ps.costo_promedio AS valor_total_costo,
        p.precio_venta,
        ps.stock_actual * p.precio_venta AS valor_total_venta,
        ps.stock_actual * (p.precio_venta - ps.costo_promedio) AS utilidad_potencial,
        ROUND((p.precio_venta - ps.costo_promedio) / NULLIF(p.precio_venta, 0) * 100, 2) AS margen_porcentaje
    FROM productos p
    JOIN categorias c ON p.id_categoria = c.id_categoria
    JOIN productos_stock ps ON p.id_producto = ps.id_producto
    WHERE p.activo = TRUE
      AND ps.stock_actual > 0
    ORDER BY c.nombre, p.nombre_comercial;
    
    -- Totales
    SELECT 
        'Resumen de Valorización' AS reporte,
        COUNT(*) AS productos_con_stock,
        SUM(stock_actual) AS total_unidades,
        ROUND(SUM(stock_actual * costo_promedio), 2) AS valor_costo_total,
        ROUND(SUM(stock_actual * precio_venta), 2) AS valor_venta_total,
        ROUND(SUM(stock_actual * (precio_venta - costo_promedio)), 2) AS utilidad_potencial_total
    FROM productos p
    JOIN productos_stock ps ON p.id_producto = ps.id_producto
    WHERE p.activo = TRUE AND ps.stock_actual > 0;
END //
```

## Ajustes de Inventario (Conteo Físico)

```sql
CREATE PROCEDURE sp_aplicar_ajuste_inventario(
    IN p_id_producto INT,
    IN p_id_almacen INT,
    IN p_cantidad_fisica DECIMAL(12,4),
    IN p_usuario VARCHAR(100)
)
BEGIN
    DECLARE v_cantidad_sistema DECIMAL(12,4);
    DECLARE v_diferencia DECIMAL(12,4);
    DECLARE v_costo DECIMAL(12,4);
    
    SELECT stock_actual, costo_promedio 
    INTO v_cantidad_sistema, v_costo
    FROM productos_stock
    WHERE id_producto = p_id_producto AND id_almacen = p_id_almacen;
    
    SET v_diferencia = p_cantidad_fisica - v_cantidad_sistema;
    
    -- Registrar el movimiento de ajuste
    INSERT INTO inventario_movimientos (
        id_producto, id_almacen, tipo_movimiento, cantidad, 
        costo_unitario, existencia_antes, referencia_tipo, usuario, notas
    ) VALUES (
        p_id_producto, p_id_almacen,
        CASE WHEN v_diferencia > 0 THEN 'ajuste_entrada' ELSE 'ajuste_salida' END,
        v_diferencia, v_costo, v_cantidad_sistema,
        'conteo_fisico', p_usuario,
        CONCAT('Ajuste por conteo físico. Sistema: ', v_cantidad_sistema, 
               ', Físico: ', p_cantidad_fisica, ', Diferencia: ', v_diferencia)
    );
END //
```

## Reporte de Costo de Ventas

```sql
CREATE VIEW vista_costo_ventas AS
SELECT 
    DATE_FORMAT(f.fecha_emision, '%Y-%m') AS periodo,
    c.nombre AS categoria,
    p.nombre_comercial AS producto,
    SUM(fd.cantidad) AS unidades_vendidas,
    ps.costo_promedio,
    SUM(fd.cantidad * ps.costo_promedio) AS costo_ventas,
    SUM(fd.total) AS ingresos_ventas,
    SUM(fd.total) - SUM(fd.cantidad * ps.costo_promedio) AS margen_bruto,
    ROUND((SUM(fd.total) - SUM(fd.cantidad * ps.costo_promedio)) / NULLIF(SUM(fd.total), 0) * 100, 2) AS margen_porcentaje
FROM facturas f
JOIN facturas_detalle fd ON f.id_factura = fd.id_factura
JOIN productos p ON fd.id_producto = p.id_producto
JOIN categorias c ON p.id_categoria = c.id_categoria
JOIN productos_stock ps ON p.id_producto = ps.id_producto
WHERE f.estado = 'activa'
  AND f.fecha_emision >= DATE_SUB(CURRENT_DATE, INTERVAL 12 MONTH)
GROUP BY DATE_FORMAT(f.fecha_emision, '%Y-%m'), c.id_categoria, p.id_producto
ORDER BY periodo DESC, costo_ventas DESC;
```

## Movimiento de Valorización (Ejemplo Completo)

```sql
-- Ejemplo: Compra de 10 unidades a $500 y venta posterior
-- 1. Registrar entrada por compra
INSERT INTO inventario_movimientos (id_producto, id_almacen, tipo_movimiento, cantidad, costo_unitario, existencia_antes, usuario)
VALUES (1, 1, 'entrada_compra', 10, 500.00, 15, 'admin');

-- 2. El trigger actualiza:
--    stock_actual = 25 (15 + 10)
--    costo_promedio = (17500*15 + 500*10) / 25 = 10700

-- 3. Venta de 3 unidades
INSERT INTO inventario_movimientos (id_producto, id_almacen, tipo_movimiento, cantidad, costo_unitario, existencia_antes, usuario)
VALUES (1, 1, 'salida_venta', -3, NULL, 25, 'admin');

-- 4. Ver el kardex
SELECT 
    fecha_movimiento,
    tipo_movimiento,
    CASE WHEN cantidad > 0 THEN cantidad ELSE 0 END AS entrada,
    CASE WHEN cantidad < 0 THEN ABS(cantidad) ELSE 0 END AS salida,
    costo_unitario,
    cantidad * costo_unitario AS costo_movimiento,
    existencia_antes,
    existencia_despues,
    ROUND(existencia_despues * (
        SELECT costo_promedio FROM productos_stock 
        WHERE id_producto = m.id_producto AND id_almacen = m.id_almacen
    ), 2) AS valor_inventario_promedio
FROM inventario_movimientos m
WHERE id_producto = 1 AND id_almacen = 1
ORDER BY fecha_movimiento;
```

---

## Anterior: [03 Analisis Inventarios](../05-inventarios/03-analisis-inventarios.md)
## Siguiente: [01 Window Functions](../06-sql-avanzado/01-window-functions.md)

[Volver al índice](../README.md)
