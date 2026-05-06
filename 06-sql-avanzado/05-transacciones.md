# 6.5 Transacciones

## Introducción

Una transacción es una unidad de trabajo que contiene una o más operaciones SQL. Sigue el principio ACID: Atomicidad, Consistencia, Aislamiento, Durabilidad.

### ACID
```
A → Atomicidad: Todo o nada. Si falla una operación, se deshacen todas.
C → Consistencia: La BD pasa de un estado válido a otro estado válido.
I → Aislamiento: Las transacciones concurrentes no interfieren entre sí.
D → Durabilidad: Una vez confirmados, los cambios persisten ante fallos.
```

## Control de Transacciones

### Comandos Básicos
```sql
START TRANSACTION;  -- Iniciar transacción
COMMIT;             -- Confirmar cambios
ROLLBACK;           -- Deshacer cambios
SAVEPOINT nombre;   -- Punto de guardado
ROLLBACK TO nombre; -- Deshacer hasta el savepoint
```

### Ejemplo Básico
```sql
START TRANSACTION;

UPDATE productos_stock SET stock_actual = stock_actual - 1 WHERE id_producto = 100;
INSERT INTO facturas_detalle (id_factura, id_producto, cantidad, total) VALUES (5000, 100, 1, 2500);

-- Si todo está bien:
COMMIT;

-- Si algo salió mal:
ROLLBACK;
```

## Transacciones en Ventas

### Venta Completa con Transacción
```sql
START TRANSACTION;

-- 1. Insertar factura
INSERT INTO facturas (folio, id_cliente, id_vendedor, subtotal, iva, total, metodo_pago)
VALUES ('F-A00001', 1, 1, 5000, 800, 5800, 'tarjeta_credito');

SET @id_factura = LAST_INSERT_ID();

-- 2. Insertar detalle
INSERT INTO facturas_detalle (id_factura, id_producto, cantidad, precio_unitario, subtotal, iva, total)
VALUES 
    (@id_factura, 1, 2, 2500, 5000, 800, 5800);

-- 3. Verificar y actualizar stock
UPDATE productos_stock 
SET stock_actual = stock_actual - 2 
WHERE id_producto = 1 AND stock_actual >= 2;

-- Verificar que se afectó una fila
IF ROW_COUNT() != 1 THEN
    ROLLBACK;
    SELECT 'Error: Stock insuficiente' AS mensaje;
ELSE
    COMMIT;
    SELECT 'Venta registrada exitosamente' AS mensaje;
END IF;
```

## Transacción con Manejo de Errores

### MySQL
```sql
DELIMITER //

CREATE PROCEDURE sp_venta_segura(
    IN p_id_cliente INT, 
    IN p_id_vendedor INT,
    IN p_id_producto INT,
    IN p_cantidad INT,
    IN p_precio DECIMAL(12,2)
)
BEGIN
    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        ROLLBACK;
        SELECT 'Error en la transacción. Operación cancelada.' AS mensaje;
    END;
    
    START TRANSACTION;
    
    -- Verificar stock
    IF (SELECT stock_actual FROM productos_stock WHERE id_producto = p_id_producto) < p_cantidad THEN
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Stock insuficiente';
    END IF;
    
    -- Insertar factura
    INSERT INTO facturas (folio, id_cliente, id_vendedor, subtotal, iva, total, metodo_pago)
    VALUES (CONCAT('F-', DATE_FORMAT(NOW(), '%Y%m%d-'), LPAD(LAST_INSERT_ID()+1, 5, '0')),
            p_id_cliente, p_id_vendedor, p_cantidad * p_precio, p_cantidad * p_precio * 0.16, p_cantidad * p_precio * 1.16, 'efectivo');
    
    SET @id_factura = LAST_INSERT_ID();
    
    -- Insertar detalle
    INSERT INTO facturas_detalle (id_factura, id_producto, cantidad, precio_unitario, subtotal, iva, total)
    VALUES (@id_factura, p_id_producto, p_cantidad, p_precio, p_cantidad * p_precio, p_cantidad * p_precio * 0.16, p_cantidad * p_precio * 1.16);
    
    -- Actualizar stock
    UPDATE productos_stock SET stock_actual = stock_actual - p_cantidad WHERE id_producto = p_id_producto;
    
    COMMIT;
    SELECT 'Venta exitosa' AS mensaje, @id_factura AS id_factura;
END //

DELIMITER ;
```

## Niveles de Aislamiento

Controlan cómo las transacciones concurrentes interactúan entre sí:

| Nivel | Dirty Read | Non-Repeatable Read | Phantom Read |
|-------|------------|--------------------|--------------|
| READ UNCOMMITTED | Posible | Posible | Posible |
| READ COMMITTED | Evitado | Posible | Posible |
| REPEATABLE READ | Evitado | Evitado | Posible (MySQL: Evitado) |
| SERIALIZABLE | Evitado | Evitado | Evitado |

```sql
-- Ver nivel actual
-- MySQL
SELECT @@transaction_isolation;

-- PostgreSQL
SHOW transaction_isolation;

-- Cambiar nivel
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ;
SET GLOBAL TRANSACTION ISOLATION LEVEL READ COMMITTED;
```

## Bloqueos (Locks)

### Bloqueo Explícito
```sql
START TRANSACTION;

-- Bloquear fila para evitar que otros la modifiquen
SELECT * FROM productos_stock 
WHERE id_producto = 1 
FOR UPDATE;

-- Ahora podemos actualizar con seguridad
UPDATE productos_stock 
SET stock_actual = stock_actual - 1 
WHERE id_producto = 1;

COMMIT;  -- Libera el lock
```

### Bloqueo Compartido
```sql
START TRANSACTION;

-- Bloqueo compartido (otros pueden leer pero no modificar)
SELECT * FROM productos 
WHERE id_categoria = 1 
LOCK IN SHARE MODE;

-- Solo lectura, no modificación
COMMIT;
```

## Deadlocks y Cómo Evitarlos

### Ejemplo de Deadlock
```sql
-- Transacción 1
START TRANSACTION;
UPDATE productos_stock SET stock_actual = stock_actual - 1 WHERE id_producto = 1;
UPDATE productos_stock SET stock_actual = stock_actual - 2 WHERE id_producto = 2;
COMMIT;

-- Transacción 2
START TRANSACTION;
UPDATE productos_stock SET stock_actual = stock_actual - 3 WHERE id_producto = 2;
UPDATE productos_stock SET stock_actual = stock_actual - 1 WHERE id_producto = 1;
COMMIT;
-- ⚠️ Deadlock si se ejecutan concurrentemente en orden inverso
```

### Cómo Prevenir Deadlocks
```sql
-- Siempre acceder a los recursos en el MISMO ORDEN
-- Ordenar por id_producto antes de actualizar

START TRANSACTION;

-- Actualizar en orden ascendente siempre
UPDATE productos_stock SET stock_actual = stock_actual - 1 
WHERE id_producto = 1;
UPDATE productos_stock SET stock_actual = stock_actual - 2 
WHERE id_producto = 2;

COMMIT;
```

## SAVEPOINT

Puntos intermedios para rollback parcial:

```sql
START TRANSACTION;

INSERT INTO facturas (folio, total) VALUES ('F-001', 1000);
SAVEPOINT factura_creada;

INSERT INTO facturas_detalle (id_factura, id_producto, cantidad) 
VALUES (LAST_INSERT_ID(), 1, 2);

-- Si el detalle falla, podemos revertir solo esa parte
ROLLBACK TO SAVEPOINT factura_creada;

-- E intentar de nuevo
INSERT INTO facturas_detalle (id_factura, id_producto, cantidad) 
VALUES (LAST_INSERT_ID(), 2, 3);

COMMIT;
```

## Transacciones Anidadas

```sql
-- MySQL: No soporta transacciones anidadas verdaderas
-- START TRANSACTION dentro de otra transacción crea un savepoint implícito

START TRANSACTION;
    INSERT INTO clientes (nombre) VALUES ('Cliente A');
    
    START TRANSACTION;  -- Esto crea un savepoint implícito
        INSERT INTO facturas (folio, id_cliente) VALUES ('F-002', LAST_INSERT_ID());
    COMMIT;  -- Libera savepoint, no confirma la transacción externa
    
ROLLBACK;  -- Deshace TODO incluyendo la inserción del cliente y factura
```

## 📊 Ejemplos del Mundo Real

### Cierre de Día (Transacción Masiva)
```sql
DELIMITER //

CREATE PROCEDURE sp_cierre_dia()
BEGIN
    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        ROLLBACK;
        INSERT INTO log_errores (procedimiento, error, fecha)
        VALUES ('sp_cierre_dia', 'Error en cierre del día', NOW());
    END;
    
    START TRANSACTION;
    
    -- 1. Calcular ventas del día
    INSERT INTO resumen_diario (fecha, total_ventas, total_ingresos, total_clientes)
    SELECT 
        CURRENT_DATE,
        COUNT(*),
        SUM(total),
        COUNT(DISTINCT id_cliente)
    FROM facturas
    WHERE DATE(fecha_emision) = CURRENT_DATE AND estado = 'activa';
    
    -- 2. Marcar facturas como procesadas
    UPDATE facturas SET estado = 'procesada' 
    WHERE DATE(fecha_emision) = CURRENT_DATE AND estado = 'activa';
    
    -- 3. Generar cortes de caja por vendedor
    INSERT INTO cortes_caja (fecha, id_vendedor, total_ventas, monto_efectivo, monto_tarjeta, monto_transferencia)
    SELECT 
        CURRENT_DATE,
        v.id_vendedor,
        COUNT(f.id_factura),
        SUM(CASE WHEN f.metodo_pago = 'efectivo' THEN f.total ELSE 0 END),
        SUM(CASE WHEN f.metodo_pago IN ('tarjeta_credito', 'tarjeta_debito') THEN f.total ELSE 0 END),
        SUM(CASE WHEN f.metodo_pago = 'transferencia' THEN f.total ELSE 0 END)
    FROM vendedores v
    LEFT JOIN facturas f ON v.id_vendedor = f.id_vendedor 
        AND DATE(f.fecha_emision) = CURRENT_DATE
    GROUP BY v.id_vendedor;
    
    COMMIT;
END //

DELIMITER ;
```

### Revertir Transacción por Error
```sql
START TRANSACTION;

-- Intentar registrar venta
INSERT INTO facturas (folio, id_cliente, total) VALUES ('F-003', 999, 5000);
-- ⚠️ Error: id_cliente 999 no existe (FK violada)

-- MySQL automáticamente hace rollback de la instrucción que falló
-- pero la transacción sigue activa

SELECT 'La transacción sigue activa, podemos continuar o hacer rollback';

-- Podemos intentar otra operación
INSERT INTO facturas (folio, id_cliente, total) VALUES ('F-003', 1, 5000);

COMMIT;  -- Solo la segunda INSERT se confirma
```

## Monitoreo de Transacciones

```sql
-- Ver transacciones activas (MySQL)
SELECT * FROM information_schema.INNODB_TRX;

-- Ver locks
SELECT * FROM information_schema.INNODB_LOCKS;
SELECT * FROM information_schema.INNODB_LOCK_WAITS;

-- Ver procesos
SHOW FULL PROCESSLIST;

-- Matar una transacción bloqueada
KILL QUERY connection_id;
KILL CONNECTION connection_id;
```

---

## Anterior: [04 Triggers](../06-sql-avanzado/04-triggers.md)
## Siguiente: [06 Performance Tuning](../06-sql-avanzado/06-performance-tuning.md)

[Volver al índice](../README.md)
