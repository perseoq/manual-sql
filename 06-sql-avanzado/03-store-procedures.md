# 6.3 Stored Procedures

## Introducción

Un Stored Procedure es un conjunto de instrucciones SQL almacenadas en el servidor, que se ejecutan como una unidad. Permiten encapsular lógica de negocio, mejorar performance y centralizar operaciones.

### Ventajas
- **Rendimiento**: Compilado y optimizado en el servidor
- **Seguridad**: Control de acceso a nivel de procedimiento
- **Reutilización**: Una vez creado, se usa desde cualquier aplicación
- **Mantenibilidad**: Lógica centralizada en la base de datos
- **Reducción de tráfico**: Se envía solo la llamada, no todo el SQL

## Sintaxis Básica

### MySQL
```sql
DELIMITER //

CREATE PROCEDURE nombre_procedimiento(
    IN parametro_entrada INT,
    OUT parametro_salida VARCHAR(100),
    INOUT parametro_inout INT
)
BEGIN
    -- Cuerpo del procedimiento
    SELECT columna INTO parametro_salida FROM tabla WHERE id = parametro_entrada;
END //

DELIMITER ;
```

### PostgreSQL
```sql
CREATE OR REPLACE FUNCTION nombre_procedimiento(
    p_parametro_entrada INT,
    OUT parametro_salida VARCHAR(100)
) AS $$
BEGIN
    SELECT columna INTO parametro_salida FROM tabla WHERE id = p_parametro_entrada;
END;
$$ LANGUAGE plpgsql;
```

## Procedimientos para Ventas

### Registrar Venta Completa
```sql
DELIMITER //

CREATE PROCEDURE sp_registrar_venta(
    IN p_id_cliente INT,
    IN p_id_vendedor INT,
    IN p_id_sucursal INT,
    IN p_metodo_pago VARCHAR(20),
    IN p_productos JSON  -- Formato: [{"id":1, "cantidad":2, "precio":2500}]
)
BEGIN
    DECLARE v_id_factura INT;
    DECLARE v_subtotal DECIMAL(12,2) DEFAULT 0;
    DECLARE v_iva DECIMAL(12,2) DEFAULT 0;
    DECLARE v_total DECIMAL(12,2) DEFAULT 0;
    DECLARE v_folio VARCHAR(30);
    DECLARE v_i INT DEFAULT 0;
    DECLARE v_productos_count INT;
    DECLARE v_id_producto INT;
    DECLARE v_cantidad INT;
    DECLARE v_precio DECIMAL(12,2);
    DECLARE v_stock_actual INT;
    DECLARE exit HANDLER FOR SQLEXCEPTION
    BEGIN
        ROLLBACK;
        RESIGNAL;
    END;
    
    START TRANSACTION;
    
    -- Generar folio
    SET v_folio = generar_folio('facturas', 'A');
    
    -- Calcular totales
    SET v_productos_count = JSON_LENGTH(p_productos);
    
    WHILE v_i < v_productos_count DO
        SET v_id_producto = JSON_UNQUOTE(JSON_EXTRACT(p_productos, CONCAT('$[', v_i, '].id')));
        SET v_cantidad = JSON_UNQUOTE(JSON_EXTRACT(p_productos, CONCAT('$[', v_i, '].cantidad')));
        SET v_precio = JSON_UNQUOTE(JSON_EXTRACT(p_productos, CONCAT('$[', v_i, '].precio')));
        
        -- Verificar stock
        SELECT stock_actual INTO v_stock_actual
        FROM productos_stock WHERE id_producto = v_id_producto;
        
        IF v_stock_actual < v_cantidad THEN
            SIGNAL SQLSTATE '45000' 
            SET MESSAGE_TEXT = 'Stock insuficiente para producto';
        END IF;
        
        SET v_subtotal = v_subtotal + (v_cantidad * v_precio);
        SET v_i = v_i + 1;
    END WHILE;
    
    SET v_iva = v_subtotal * 0.16;
    SET v_total = v_subtotal + v_iva;
    
    -- Insertar factura
    INSERT INTO facturas (folio, id_cliente, id_vendedor, id_sucursal, 
        fecha_emision, subtotal, iva, total, metodo_pago, estado)
    VALUES (v_folio, p_id_cliente, p_id_vendedor, p_id_sucursal,
        NOW(), v_subtotal, v_iva, v_total, p_metodo_pago, 'activa');
    
    SET v_id_factura = LAST_INSERT_ID();
    
    -- Insertar detalle
    SET v_i = 0;
    WHILE v_i < v_productos_count DO
        SET v_id_producto = JSON_UNQUOTE(JSON_EXTRACT(p_productos, CONCAT('$[', v_i, '].id')));
        SET v_cantidad = JSON_UNQUOTE(JSON_EXTRACT(p_productos, CONCAT('$[', v_i, '].cantidad')));
        SET v_precio = JSON_UNQUOTE(JSON_EXTRACT(p_productos, CONCAT('$[', v_i, '].precio')));
        
        INSERT INTO facturas_detalle (id_factura, id_producto, cantidad, precio_unitario, subtotal, iva, total)
        VALUES (v_id_factura, v_id_producto, v_cantidad, v_precio, 
                v_cantidad * v_precio, v_cantidad * v_precio * 0.16, v_cantidad * v_precio * 1.16);
        
        -- Actualizar stock
        UPDATE productos_stock 
        SET stock_actual = stock_actual - v_cantidad
        WHERE id_producto = v_id_producto;
        
        SET v_i = v_i + 1;
    END WHILE;
    
    COMMIT;
    
    SELECT v_id_factura AS id_factura, v_folio AS folio, v_total AS total;
END //

DELIMITER ;

-- Uso:
-- CALL sp_registrar_venta(1, 1, 1, 'tarjeta_credito', 
--     '[{"id":1, "cantidad":2, "precio":2500}, {"id":2, "cantidad":5, "precio":350}]');
```

### Cancelar Factura
```sql
DELIMITER //

CREATE PROCEDURE sp_cancelar_factura(
    IN p_id_factura INT,
    IN p_motivo TEXT,
    IN p_usuario VARCHAR(100)
)
BEGIN
    DECLARE v_estado VARCHAR(20);
    DECLARE v_id_cliente INT;
    DECLARE v_total DECIMAL(12,2);
    DECLARE v_metodo_pago VARCHAR(20);
    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        ROLLBACK;
        RESIGNAL;
    END;
    
    START TRANSACTION;
    
    -- Obtener datos de la factura
    SELECT estado, id_cliente, total, metodo_pago 
    INTO v_estado, v_id_cliente, v_total, v_metodo_pago
    FROM facturas WHERE id_factura = p_id_factura
    FOR UPDATE;
    
    IF v_estado != 'activa' THEN
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'La factura no está activa';
    END IF;
    
    -- Restaurar inventario
    INSERT INTO inventario_movimientos (id_producto, tipo_movimiento, cantidad, referencia_tipo, referencia_id, usuario, notas)
    SELECT id_producto, 'devolucion_cliente', cantidad, 'cancelacion_factura', p_id_factura, p_usuario, p_motivo
    FROM facturas_detalle WHERE id_factura = p_id_factura;
    
    UPDATE productos_stock ps
    JOIN facturas_detalle fd ON ps.id_producto = fd.id_producto
    SET ps.stock_actual = ps.stock_actual + fd.cantidad
    WHERE fd.id_factura = p_id_factura;
    
    -- Si era crédito, restaurar saldo
    IF v_metodo_pago = 'credito' THEN
        UPDATE clientes SET saldo_actual = saldo_actual - v_total
        WHERE id_cliente = v_id_cliente;
    END IF;
    
    -- Cancelar factura
    UPDATE facturas SET estado = 'cancelada' WHERE id_factura = p_id_factura;
    
    COMMIT;
END //

DELIMITER ;
```

## Procedimientos para Compras

### Crear Orden de Compra desde Punto de Reorden
```sql
DELIMITER //

CREATE PROCEDURE sp_generar_orden_reorden(
    IN p_id_proveedor INT,
    IN p_id_sucursal INT,
    IN p_usuario VARCHAR(100)
)
BEGIN
    DECLARE v_id_orden INT;
    DECLARE v_folio VARCHAR(30);
    DECLARE v_subtotal DECIMAL(12,2) DEFAULT 0;
    DECLARE done INT DEFAULT FALSE;
    DECLARE v_id_producto INT;
    DECLARE v_cantidad INT;
    DECLARE v_precio DECIMAL(12,2);
    
    -- Cursor para productos que necesitan reorden
    DECLARE cur_reorden CURSOR FOR
        SELECT p.id_producto, 
               CEIL(ps.punto_reorden - ps.stock_actual + 
                    (SELECT AVG(fd.cantidad) FROM facturas_detalle fd 
                     JOIN facturas f ON fd.id_factura = f.id_factura
                     WHERE fd.id_producto = p.id_producto AND f.estado = 'activa'
                       AND f.fecha_emision >= DATE_SUB(CURRENT_DATE, INTERVAL 30 DAY)
                    )) AS cantidad_sugerida,
               pp.precio_compra
        FROM productos p
        JOIN productos_stock ps ON p.id_producto = ps.id_producto
        JOIN proveedores_productos pp ON p.id_producto = pp.id_producto AND pp.id_proveedor = p_id_proveedor
        WHERE p.activo = TRUE
          AND ps.stock_actual <= ps.punto_reorden
          AND NOT EXISTS (
              SELECT 1 FROM ordenes_compra_detalle ocd
              JOIN ordenes_compra oc ON ocd.id_orden = oc.id_orden
              WHERE ocd.id_producto = p.id_producto
                AND oc.id_proveedor = p_id_proveedor
                AND oc.estado IN ('autorizada', 'enviada', 'confirmada')
          );
    
    DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = TRUE;
    
    START TRANSACTION;
    
    SET v_folio = generar_folio('ordenes_compra', 'A');
    
    INSERT INTO ordenes_compra (folio, id_proveedor, id_sucursal, fecha_emision, 
        fecha_estimada_entrega, estado, usuario_creacion)
    VALUES (v_folio, p_id_proveedor, p_id_sucursal, NOW(), 
            DATE_ADD(CURRENT_DATE, INTERVAL 7 DAY), 'borrador', p_usuario);
    
    SET v_id_orden = LAST_INSERT_ID();
    
    OPEN cur_reorden;
    
    read_loop: LOOP
        FETCH cur_reorden INTO v_id_producto, v_cantidad, v_precio;
        IF done THEN
            LEAVE read_loop;
        END IF;
        
        INSERT INTO ordenes_compra_detalle (id_orden, id_producto, cantidad_solicitada, precio_unitario, subtotal, iva, total)
        VALUES (v_id_orden, v_id_producto, v_cantidad, v_precio,
                v_cantidad * v_precio, v_cantidad * v_precio * 0.16, v_cantidad * v_precio * 1.16);
        
        SET v_subtotal = v_subtotal + (v_cantidad * v_precio);
    END LOOP;
    
    CLOSE cur_reorden;
    
    UPDATE ordenes_compra 
    SET subtotal = v_subtotal, iva = v_subtotal * 0.16, total = v_subtotal * 1.16
    WHERE id_orden = v_id_orden;
    
    COMMIT;
    
    SELECT v_id_orden AS id_orden, v_folio AS folio, v_subtotal AS subtotal;
END //

DELIMITER ;
```

## Procedimientos para Inventarios

### Transferencia entre Almacenes
```sql
DELIMITER //

CREATE PROCEDURE sp_transferir_producto(
    IN p_id_producto INT,
    IN p_id_almacen_origen INT,
    IN p_id_almacen_destino INT,
    IN p_cantidad DECIMAL(12,4),
    IN p_usuario VARCHAR(100)
)
BEGIN
    DECLARE v_stock_origen DECIMAL(12,4);
    DECLARE v_costo DECIMAL(12,4);
    
    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        ROLLBACK;
        RESIGNAL;
    END;
    
    START TRANSACTION;
    
    -- Verificar stock en origen
    SELECT stock_actual, costo_promedio 
    INTO v_stock_origen, v_costo
    FROM productos_stock 
    WHERE id_producto = p_id_producto AND id_almacen = p_id_almacen_origen
    FOR UPDATE;
    
    IF v_stock_origen < p_cantidad THEN
        SIGNAL SQLSTATE '45000' 
        SET MESSAGE_TEXT = 'Stock insuficiente en almacén origen';
    END IF;
    
    -- Registrar salida
    INSERT INTO inventario_movimientos (id_producto, id_almacen, tipo_movimiento, cantidad, costo_unitario, existencia_antes, usuario)
    VALUES (p_id_producto, p_id_almacen_origen, 'transferencia_salida', -p_cantidad, v_costo, v_stock_origen, p_usuario);
    
    -- Registrar entrada
    INSERT INTO inventario_movimientos (id_producto, id_almacen, tipo_movimiento, cantidad, costo_unitario, existencia_antes, usuario)
    VALUES (p_id_producto, p_id_almacen_destino, 'transferencia_entrada', p_cantidad, v_costo, 
            (SELECT COALESCE(stock_actual, 0) FROM productos_stock WHERE id_producto = p_id_producto AND id_almacen = p_id_almacen_destino), 
            p_usuario);
    
    COMMIT;
END //

DELIMITER ;
```

### Reporte de Ventas por Período
```sql
DELIMITER //

CREATE PROCEDURE sp_reporte_ventas_periodo(
    IN p_fecha_inicio DATE,
    IN p_fecha_fin DATE,
    IN p_id_sucursal INT,
    IN p_id_vendedor INT
)
BEGIN
    -- Volumen de ventas
    SELECT 
        DATE_FORMAT(f.fecha_emision, '%Y-%m-%d') AS fecha,
        COUNT(DISTINCT f.id_factura) AS facturas,
        COUNT(DISTINCT f.id_cliente) AS clientes,
        SUM(f.total) AS ingresos,
        ROUND(AVG(f.total), 2) AS ticket_promedio,
        COUNT(DISTINCT fd.id_producto) AS productos_vendidos,
        SUM(fd.cantidad) AS unidades_vendidas
    FROM facturas f
    LEFT JOIN facturas_detalle fd ON f.id_factura = fd.id_factura
    WHERE f.estado = 'activa'
      AND f.fecha_emision BETWEEN p_fecha_inicio AND p_fecha_fin
      AND (p_id_sucursal IS NULL OR f.id_sucursal = p_id_sucursal)
      AND (p_id_vendedor IS NULL OR f.id_vendedor = p_id_vendedor)
    GROUP BY DATE(f.fecha_emision)
    ORDER BY fecha;
    
    -- Totales
    SELECT 
        COUNT(DISTINCT id_factura) AS total_facturas,
        SUM(total) AS total_ingresos,
        ROUND(AVG(total), 2) AS ticket_promedio,
        COUNT(DISTINCT id_cliente) AS clientes_atendidos
    FROM facturas
    WHERE estado = 'activa'
      AND fecha_emision BETWEEN p_fecha_inicio AND p_fecha_fin
      AND (p_id_sucursal IS NULL OR id_sucursal = p_id_sucursal)
      AND (p_id_vendedor IS NULL OR id_vendedor = p_id_vendedor);
END //

DELIMITER ;

-- Uso: CALL sp_reporte_ventas_periodo('2024-01-01', '2024-12-31', NULL, NULL);
```

### Función para Calcular IVA
```sql
DELIMITER //

CREATE FUNCTION fn_calcular_iva(
    p_monto DECIMAL(12,2),
    p_tasa DECIMAL(4,2)
) 
RETURNS DECIMAL(12,2)
DETERMINISTIC
BEGIN
    RETURN ROUND(p_monto * p_tasa / 100, 2);
END //

DELIMITER ;

-- Uso: SELECT fn_calcular_iva(1000, 16); -- 160.00
```

### Función para Estado del Stock
```sql
DELIMITER //

CREATE FUNCTION fn_estado_stock(
    p_id_producto INT,
    p_id_almacen INT
)
RETURNS VARCHAR(20)
DETERMINISTIC
READS SQL DATA
BEGIN
    DECLARE v_stock DECIMAL(12,4);
    DECLARE v_minimo DECIMAL(12,4);
    DECLARE v_reorden DECIMAL(12,4);
    
    SELECT stock_actual, stock_minimo, punto_reorden 
    INTO v_stock, v_minimo, v_reorden
    FROM productos_stock
    WHERE id_producto = p_id_producto AND id_almacen = p_id_almacen;
    
    RETURN CASE 
        WHEN v_stock <= 0 THEN 'SIN STOCK'
        WHEN v_stock <= v_reorden THEN 'REORDENAR'
        WHEN v_stock <= v_minimo THEN 'MÍNIMO'
        ELSE 'OK'
    END;
END //

DELIMITER ;

-- Uso: SELECT fn_estado_stock(1, 1);
```

## Manejo de Errores

```sql
DELIMITER //

CREATE PROCEDURE sp_ejemplo_manejo_errores()
BEGIN
    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        GET DIAGNOSTICS CONDITION 1 @sqlstate = RETURNED_SQLSTATE, 
            @errno = MYSQL_ERRNO, @text = MESSAGE_TEXT;
        ROLLBACK;
        SELECT CONCAT('Error ', @errno, ' (', @sqlstate, '): ', @text) AS error;
    END;
    
    DECLARE CONTINUE HANDLER FOR NOT FOUND
    BEGIN
        -- Cuando no se encuentra un registro
        SET @not_found = TRUE;
    END;
    
    DECLARE CONTINUE HANDLER FOR SQLWARNING
    BEGIN
        -- Advertencias (no fatales)
        GET DIAGNOSTICS CONDITION 1 @warning_text = MESSAGE_TEXT;
    END;
    
    START TRANSACTION;
    -- Operaciones aquí
    COMMIT;
END //

DELIMITER ;
```

---

## Anterior: [02 Cte Y Recursividad](../06-sql-avanzado/02-cte-y-recursividad.md)
## Siguiente: [04 Triggers](../06-sql-avanzado/04-triggers.md)

[Volver al índice](../README.md)
