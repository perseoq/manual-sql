# 6.4 Triggers

## Introducción

Un trigger es un conjunto de instrucciones que se ejecutan automáticamente en respuesta a eventos (INSERT, UPDATE, DELETE) sobre una tabla.

### Tipos de Triggers

```sql
BEFORE INSERT    → Antes de insertar
AFTER INSERT     → Después de insertar
BEFORE UPDATE    → Antes de actualizar
AFTER UPDATE     → Después de actualizar
BEFORE DELETE    → Antes de eliminar
AFTER DELETE     → Después de eliminar
INSTEAD OF       → En lugar de (solo vistas)
```

## Sintaxis

### MySQL
```sql
DELIMITER //

CREATE TRIGGER nombre_trigger
{BEFORE | AFTER} {INSERT | UPDATE | DELETE}
ON nombre_tabla FOR EACH ROW
BEGIN
    -- NEW.columna → nuevo valor (INSERT/UPDATE)
    -- OLD.columna → valor anterior (UPDATE/DELETE)
END //

DELIMITER ;
```

## Triggers para Ventas

### Actualizar Stock al Facturar
```sql
DELIMITER //

CREATE TRIGGER trg_after_insert_factura_detalle
AFTER INSERT ON facturas_detalle
FOR EACH ROW
BEGIN
    -- Actualizar stock
    UPDATE productos_stock ps
    SET ps.stock_actual = ps.stock_actual - NEW.cantidad,
        ps.fecha_ultima_venta = NOW()
    WHERE ps.id_producto = NEW.id_producto;
    
    -- Registrar movimiento de inventario
    INSERT INTO inventario_movimientos (
        id_producto, tipo_movimiento, cantidad, costo_unitario,
        existencia_antes, referencia_tipo, referencia_id, usuario
    ) VALUES (
        NEW.id_producto, 'salida_venta', -NEW.cantidad,
        (SELECT costo_promedio FROM productos_stock WHERE id_producto = NEW.id_producto LIMIT 1),
        (SELECT stock_actual + NEW.cantidad FROM productos_stock WHERE id_producto = NEW.id_producto),
        'factura', NEW.id_factura, 'sistema'
    );
END //

DELIMITER ;
```

### Actualizar Saldo del Cliente
```sql
DELIMITER //

CREATE TRIGGER trg_after_insert_factura
AFTER INSERT ON facturas
FOR EACH ROW
BEGIN
    IF NEW.metodo_pago = 'credito' THEN
        UPDATE clientes 
        SET saldo_actual = saldo_actual + NEW.total
        WHERE id_cliente = NEW.id_cliente;
    END IF;
END //

DELIMITER ;
```

### Restaurar Stock al Cancelar Factura
```sql
DELIMITER //

CREATE TRIGGER trg_before_update_factura_cancelacion
BEFORE UPDATE ON facturas
FOR EACH ROW
BEGIN
    IF NEW.estado = 'cancelada' AND OLD.estado = 'activa' THEN
        -- Restaurar stock
        UPDATE productos_stock ps
        JOIN facturas_detalle fd ON ps.id_producto = fd.id_producto
        SET ps.stock_actual = ps.stock_actual + fd.cantidad
        WHERE fd.id_factura = NEW.id_factura;
        
        -- Restaurar saldo si era crédito
        IF OLD.metodo_pago = 'credito' THEN
            UPDATE clientes 
            SET saldo_actual = saldo_actual - OLD.total
            WHERE id_cliente = OLD.id_cliente;
        END IF;
    END IF;
END //

DELIMITER ;
```

## Triggers para Inventarios

### Validar Stock Mínimo antes de Venta
```sql
DELIMITER //

CREATE TRIGGER trg_before_insert_factura_detalle
BEFORE INSERT ON facturas_detalle
FOR EACH ROW
BEGIN
    DECLARE v_stock DECIMAL(12,4);
    
    SELECT stock_actual INTO v_stock
    FROM productos_stock
    WHERE id_producto = NEW.id_producto;
    
    IF v_stock < NEW.cantidad THEN
        SIGNAL SQLSTATE '45000' 
        SET MESSAGE_TEXT = 'Stock insuficiente para realizar la venta';
    END IF;
END //

DELIMITER ;
```

### Auditoría de Precios
```sql
DELIMITER //

CREATE TRIGGER trg_before_update_precio_producto
BEFORE UPDATE ON productos
FOR EACH ROW
BEGIN
    IF OLD.precio_venta != NEW.precio_venta THEN
        INSERT INTO auditoria_cambios (
            tabla, id_registro, campo, valor_anterior, valor_nuevo, 
            usuario, fecha_cambio
        ) VALUES (
            'productos', NEW.id_producto, 'precio_venta',
            OLD.precio_venta, NEW.precio_venta,
            CURRENT_USER(), NOW()
        );
    END IF;
END //

DELIMITER ;
```

### Actualizar Costo Promedio en Entradas
```sql
DELIMITER //

CREATE TRIGGER trg_before_insert_movimiento_entrada
BEFORE INSERT ON inventario_movimientos
FOR EACH ROW
BEGIN
    DECLARE v_stock_actual DECIMAL(12,4);
    DECLARE v_costo_actual DECIMAL(12,4);
    
    IF NEW.cantidad > 0 AND NEW.tipo_movimiento = 'entrada_compra' THEN
        SELECT stock_actual, costo_promedio 
        INTO v_stock_actual, v_costo_actual
        FROM productos_stock
        WHERE id_producto = NEW.id_producto AND id_almacen = COALESCE(NEW.id_almacen, 0);
        
        -- Registrar existencia antes del movimiento
        SET NEW.existencia_antes = v_stock_actual;
        
        -- Actualizar costo promedio
        UPDATE productos_stock
        SET costo_promedio = (v_costo_actual * v_stock_actual + NEW.costo_unitario * NEW.cantidad) 
                            / NULLIF(v_stock_actual + NEW.cantidad, 0),
            ultimo_costo = NEW.costo_unitario
        WHERE id_producto = NEW.id_producto AND id_almacen = COALESCE(NEW.id_almacen, 0);
    END IF;
END //

DELIMITER ;
```

## Triggers para Auditoría

### Auditoría General de Cambios
```sql
-- Tabla de auditoría
CREATE TABLE auditoria_cambios (
    id_auditoria INT PRIMARY KEY AUTO_INCREMENT,
    tabla VARCHAR(50) NOT NULL,
    id_registro INT NOT NULL,
    campo VARCHAR(50) NOT NULL,
    valor_anterior TEXT,
    valor_nuevo TEXT,
    usuario VARCHAR(100),
    fecha_cambio DATETIME DEFAULT CURRENT_TIMESTAMP,
    tipo_operacion ENUM('INSERT', 'UPDATE', 'DELETE')
);

-- Trigger de auditoría para clientes
DELIMITER //

CREATE TRIGGER trg_audit_clientes_update
AFTER UPDATE ON clientes
FOR EACH ROW
BEGIN
    IF OLD.nombre != NEW.nombre THEN
        INSERT INTO auditoria_cambios (tabla, id_registro, campo, valor_anterior, valor_nuevo, usuario, tipo_operacion)
        VALUES ('clientes', NEW.id_cliente, 'nombre', OLD.nombre, NEW.nombre, CURRENT_USER(), 'UPDATE');
    END IF;
    
    IF OLD.email != NEW.email THEN
        INSERT INTO auditoria_cambios (tabla, id_registro, campo, valor_anterior, valor_nuevo, usuario, tipo_operacion)
        VALUES ('clientes', NEW.id_cliente, 'email', OLD.email, NEW.email, CURRENT_USER(), 'UPDATE');
    END IF;
    
    IF OLD.credito_limite != NEW.credito_limite THEN
        INSERT INTO auditoria_cambios (tabla, id_registro, campo, valor_anterior, valor_nuevo, usuario, tipo_operacion)
        VALUES ('clientes', NEW.id_cliente, 'credito_limite', OLD.credito_limite, NEW.credito_limite, CURRENT_USER(), 'UPDATE');
    END IF;
END //

DELIMITER ;
```

### Evitar Borrados Físicos (Borrado Lógico)
```sql
DELIMITER //

CREATE TRIGGER trg_before_delete_clientes
BEFORE DELETE ON clientes
FOR EACH ROW
BEGIN
    SIGNAL SQLSTATE '45000' 
    SET MESSAGE_TEXT = 'No se permite eliminar clientes. Use activo = FALSE para desactivarlos';
END //

DELIMITER ;
```

## Triggers con INSTEAD OF (Vistas)

```sql
-- Crear vista
CREATE VIEW vista_clientes_activos AS
SELECT * FROM clientes WHERE activo = TRUE;

-- Trigger INSTEAD OF en la vista (SQL Server / PostgreSQL)
-- MySQL no soporta INSTEAD OF triggers

-- PostgreSQL
CREATE OR REPLACE FUNCTION fn_insert_cliente_vista()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO clientes (nombre, email, activo)
    VALUES (NEW.nombre, NEW.email, TRUE);
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_instead_insert_clientes
INSTEAD OF INSERT ON vista_clientes_activos
FOR EACH ROW
EXECUTE FUNCTION fn_insert_cliente_vista();
```

## 📊 Ejemplos del Mundo Real

### Trigger Completo: Registro de Venta
```sql
DELIMITER //

CREATE TRIGGER trg_completo_venta
AFTER INSERT ON facturas_detalle
FOR EACH ROW
BEGIN
    DECLARE v_costo DECIMAL(12,4);
    DECLARE v_metodo_pago VARCHAR(20);
    DECLARE v_id_cliente INT;
    DECLARE v_total DECIMAL(12,2);
    
    -- Obtener costo del producto
    SELECT costo_promedio INTO v_costo
    FROM productos_stock
    WHERE id_producto = NEW.id_producto;
    
    -- 1. Actualizar stock
    UPDATE productos_stock
    SET stock_actual = stock_actual - NEW.cantidad,
        fecha_ultima_venta = NOW()
    WHERE id_producto = NEW.id_producto;
    
    -- 2. Registrar movimiento
    INSERT INTO inventario_movimientos (
        id_producto, tipo_movimiento, cantidad, costo_unitario,
        referencia_tipo, referencia_id, usuario
    ) VALUES (
        NEW.id_producto, 'salida_venta', -NEW.cantidad, v_costo,
        'factura', NEW.id_factura, 'trigger_venta'
    );
    
    -- 3. Verificar si es crédito y actualizar saldo
    SELECT metodo_pago, id_cliente INTO v_metodo_pago, v_id_cliente
    FROM facturas WHERE id_factura = NEW.id_factura;
    
    IF v_metodo_pago = 'credito' THEN
        UPDATE clientes 
        SET saldo_actual = saldo_actual + NEW.total
        WHERE id_cliente = v_id_cliente;
    END IF;
    
    -- 4. Registrar comisión del vendedor (si existe)
    INSERT INTO comisiones_vendedores (id_factura, id_vendedor, monto_comision, fecha_registro)
    SELECT NEW.id_factura, f.id_vendedor, NEW.total * v.comision_porcentaje / 100, NOW()
    FROM facturas f
    JOIN vendedores v ON f.id_vendedor = v.id_vendedor
    WHERE f.id_factura = NEW.id_factura;
END //

DELIMITER ;
```

### Trigger de Seguridad: Detectar Insert Masivo
```sql
DELIMITER //

CREATE TRIGGER trg_seguridad_insert_masivo
BEFORE INSERT ON facturas
FOR EACH ROW
BEGIN
    DECLARE v_inserts_ultimo_minuto INT;
    
    SELECT COUNT(*) INTO v_inserts_ultimo_minuto
    FROM facturas
    WHERE usuario_creacion = NEW.usuario_creacion
      AND fecha_emision >= DATE_SUB(NOW(), INTERVAL 1 MINUTE);
    
    IF v_inserts_ultimo_minuto >= 100 THEN
        INSERT INTO alertas_seguridad (tipo, descripcion, usuario, ip_address, fecha_alerta)
        VALUES ('INSERT_MASIVO', 
                CONCAT('Intento de insert masivo: ', v_inserts_ultimo_minuto, ' facturas en 1 minuto'),
                NEW.usuario_creacion, 
                CONNECTION_ID(),
                NOW());
    END IF;
END //

DELIMITER ;
```

## Buenas Prácticas

✅ **No usar lógica de negocio compleja** en triggers (dificulta depuración)
✅ **Evitar triggers que llamen otros triggers** (efecto cascada)
✅ **Documentar cada trigger** con su propósito
✅ **Monitorear performance** (los triggers se ejecutan en cada fila)
✅ **Usar transacciones** explícitas si el trigger afecta múltiples tablas
✅ **Considerar usar procedures** en lugar de triggers para lógica compleja

```sql
-- Ver triggers existentes
SHOW TRIGGERS;

-- Ver creación del trigger
SHOW CREATE TRIGGER trg_after_insert_factura_detalle;

-- Eliminar trigger
DROP TRIGGER IF EXISTS trg_after_insert_factura_detalle;

-- Deshabilitar trigger (no soportado en MySQL, usar variable)
SET @disable_triggers = TRUE;
```

---

## Anterior: [03 Store Procedures](../06-sql-avanzado/03-store-procedures.md)
## Siguiente: [05 Transacciones](../06-sql-avanzado/05-transacciones.md)

[Volver al índice](../README.md)
