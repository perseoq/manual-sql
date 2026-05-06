# 5.1 Modelo de Datos para Inventarios

## Arquitectura del Módulo de Inventarios

Gestiona el stock, movimientos, valorización y trazabilidad de productos en múltiples almacenes.

## Diagrama Entidad-Relación

```
┌───────────────────┐     ┌──────────────────────┐     ┌───────────────────┐
│    ALMACENES      │     │  PRODUCTOS_STOCK      │     │    PRODUCTOS      │
├───────────────────┤     ├──────────────────────┤     ├───────────────────┤
│ PK id_almacen     │<────│ PK id_producto        │────>│ PK id_producto    │
│    codigo         │     │ PK id_almacen         │     │    sku            │
│    nombre         │     │    stock_actual       │     │    nombre         │
│    tipo           │     │    stock_minimo       │     │    precio_compra  │
│    activo         │     │    stock_maximo       │     │    precio_venta   │
└───────────────────┘     │    punto_reorden      │     └───────────────────┘
        │                  │    costo_promedio     │
        │                  │    ultima_actualizacion│
        │                  └──────────────────────┘
        │                          1:N
        │                           │
        │                           ▼
        │                  ┌──────────────────────┐
        │                  │ INV_MOVIMIENTOS       │
        │                  ├──────────────────────┤
        │                  │ PK id_movimiento      │
        └─────────────────>│    FK id_producto     │
                           │    FK id_almacen      │
                           │    tipo_movimiento    │
                           │    cantidad           │
                           │    costo_unitario     │
                           │    referencia         │
                           │    fecha_movimiento   │
                           └──────────────────────┘
```

## Tablas del Módulo de Inventarios

### 1. Almacenes
```sql
CREATE TABLE almacenes (
    id_almacen INT PRIMARY KEY AUTO_INCREMENT,
    id_sucursal INT NOT NULL,
    codigo VARCHAR(10) UNIQUE NOT NULL,
    nombre VARCHAR(100) NOT NULL,
    tipo ENUM('principal', 'secundario', 'transito', 'devoluciones', 'merma', 'cuarentena') DEFAULT 'principal',
    ubicacion TEXT,
    capacidad_maxima DECIMAL(12,4),
    activo BOOLEAN DEFAULT TRUE,
    fecha_creacion DATETIME DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (id_sucursal) REFERENCES sucursales(id_sucursal)
);
```

### 2. Stock por Producto y Almacén
```sql
CREATE TABLE productos_stock (
    id_producto INT NOT NULL,
    id_almacen INT NOT NULL,
    stock_actual DECIMAL(12,4) DEFAULT 0,
    stock_reservado DECIMAL(12,4) DEFAULT 0,
    stock_disponible DECIMAL(12,4) GENERATED ALWAYS AS (stock_actual - stock_reservado) STORED,
    stock_minimo DECIMAL(12,4) DEFAULT 0,
    stock_maximo DECIMAL(12,4) DEFAULT 0,
    punto_reorden DECIMAL(12,4) DEFAULT 0,
    costo_promedio DECIMAL(12,4) DEFAULT 0,
    ultimo_costo DECIMAL(12,4) DEFAULT 0,
    fecha_ultima_compra DATETIME,
    fecha_ultima_venta DATETIME,
    ultima_actualizacion DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    PRIMARY KEY (id_producto, id_almacen),
    FOREIGN KEY (id_producto) REFERENCES productos(id_producto),
    FOREIGN KEY (id_almacen) REFERENCES almacenes(id_almacen)
);
```

### 3. Movimientos de Inventario
```sql
CREATE TABLE inventario_movimientos (
    id_movimiento INT PRIMARY KEY AUTO_INCREMENT,
    id_producto INT NOT NULL,
    id_almacen INT,
    tipo_movimiento VARCHAR(50) NOT NULL,
    cantidad DECIMAL(12,4) NOT NULL,
    costo_unitario DECIMAL(12,4),
    existencia_antes DECIMAL(12,4),
    existencia_despues DECIMAL(12,4) GENERATED ALWAYS AS (existencia_antes + cantidad) STORED,
    referencia_tipo VARCHAR(50),
    referencia_id INT,
    lote VARCHAR(50),
    fecha_caducidad DATE,
    ubicacion_fisica VARCHAR(100),
    fecha_movimiento DATETIME DEFAULT CURRENT_TIMESTAMP,
    usuario VARCHAR(100),
    notas TEXT,
    FOREIGN KEY (id_producto) REFERENCES productos(id_producto),
    FOREIGN KEY (id_almacen) REFERENCES almacenes(id_almacen)
);

-- Índices para movimientos
CREATE INDEX idx_movimientos_producto ON inventario_movimientos(id_producto);
CREATE INDEX idx_movimientos_fecha ON inventario_movimientos(fecha_movimiento);
CREATE INDEX idx_movimientos_tipo ON inventario_movimientos(tipo_movimiento);
CREATE INDEX idx_movimientos_referencia ON inventario_movimientos(referencia_tipo, referencia_id);
CREATE INDEX idx_movimientos_lote ON inventario_movimientos(lote);
```

### 4. Tipos de Movimiento Configurables
```sql
CREATE TABLE tipos_movimiento (
    codigo VARCHAR(50) PRIMARY KEY,
    nombre VARCHAR(100) NOT NULL,
    signo ENUM('+', '-') NOT NULL,
    afecta_costo BOOLEAN DEFAULT TRUE,
    requiere_autorizacion BOOLEAN DEFAULT FALSE,
    activo BOOLEAN DEFAULT TRUE
);

INSERT INTO tipos_movimiento (codigo, nombre, signo, afecta_costo) VALUES
    ('entrada_compra', 'Entrada por Compra', '+', TRUE),
    ('salida_venta', 'Salida por Venta', '-', FALSE),
    ('ajuste_entrada', 'Ajuste por Sobrante', '+', TRUE),
    ('ajuste_salida', 'Ajuste por Faltante', '-', FALSE),
    ('transferencia_entrada', 'Entrada por Transferencia', '+', FALSE),
    ('transferencia_salida', 'Salida por Transferencia', '-', FALSE),
    ('produccion_entrada', 'Entrada por Producción', '+', TRUE),
    ('produccion_salida', 'Salida por Producción', '-', FALSE),
    ('merma', 'Merma', '-', FALSE),
    ('devolucion_cliente', 'Devolución de Cliente', '+', TRUE),
    ('devolucion_proveedor', 'Devolución a Proveedor', '-', TRUE),
    ('inventario_inicial', 'Inventario Inicial', '+', TRUE);
```

### 5. Conteos Cíclicos
```sql
CREATE TABLE conteos_ciclicos (
    id_conteo INT PRIMARY KEY AUTO_INCREMENT,
    id_almacen INT NOT NULL,
    id_producto INT NOT NULL,
    fecha_programada DATE,
    fecha_conteo DATETIME,
    cantidad_sistema DECIMAL(12,4),
    cantidad_fisica DECIMAL(12,4),
    diferencia DECIMAL(12,4) GENERATED ALWAYS AS (cantidad_fisica - cantidad_sistema) STORED,
    costo_unitario DECIMAL(12,4),
    diferencia_valorizada DECIMAL(12,4) GENERATED ALWAYS AS ((cantidad_fisica - cantidad_sistema) * costo_unitario) STORED,
    usuario_conteo VARCHAR(100),
    estado ENUM('pendiente', 'conteado', 'ajustado', 'investigacion') DEFAULT 'pendiente',
    notas TEXT,
    FOREIGN KEY (id_almacen) REFERENCES almacenes(id_almacen),
    FOREIGN KEY (id_producto) REFERENCES productos(id_producto)
);
```

### 6. Transferencias entre Almacenes
```sql
CREATE TABLE transferencias (
    id_transferencia INT PRIMARY KEY AUTO_INCREMENT,
    folio VARCHAR(30) UNIQUE NOT NULL,
    id_almacen_origen INT NOT NULL,
    id_almacen_destino INT NOT NULL,
    fecha_creacion DATETIME DEFAULT CURRENT_TIMESTAMP,
    fecha_envio DATETIME,
    fecha_recepcion DATETIME,
    usuario_crea VARCHAR(100),
    usuario_recibe VARCHAR(100),
    estado ENUM('borrador', 'enviada', 'en_transito', 'recibida', 'cancelada') DEFAULT 'borrador',
    notas TEXT,
    FOREIGN KEY (id_almacen_origen) REFERENCES almacenes(id_almacen),
    FOREIGN KEY (id_almacen_destino) REFERENCES almacenes(id_almacen)
);

CREATE TABLE transferencias_detalle (
    id_detalle INT PRIMARY KEY AUTO_INCREMENT,
    id_transferencia INT NOT NULL,
    id_producto INT NOT NULL,
    cantidad_enviada DECIMAL(12,4) NOT NULL,
    cantidad_recibida DECIMAL(12,4) DEFAULT 0,
    costo_unitario DECIMAL(12,4),
    FOREIGN KEY (id_transferencia) REFERENCES transferencias(id_transferencia),
    FOREIGN KEY (id_producto) REFERENCES productos(id_producto)
);
```

## Triggers para Control de Inventario

### Actualizar Stock al Insertar Movimiento
```sql
DELIMITER //
CREATE TRIGGER trg_inventario_movimiento_insert
AFTER INSERT ON inventario_movimientos
FOR EACH ROW
BEGIN
    DECLARE v_costo_promedio DECIMAL(12,4);
    
    -- Actualizar existencias en productos_stock
    INSERT INTO productos_stock (id_producto, id_almacen, stock_actual, ultima_actualizacion)
    VALUES (NEW.id_producto, NEW.id_almacen, 0, NOW())
    ON DUPLICATE KEY UPDATE
        stock_actual = stock_actual + NEW.cantidad,
        ultima_actualizacion = NOW();
    
    -- Si es entrada y afecta costo, actualizar costo promedio
    IF NEW.cantidad > 0 AND NEW.costo_unitario IS NOT NULL AND NEW.costo_unitario > 0 THEN
        SELECT costo_promedio INTO v_costo_promedio
        FROM productos_stock
        WHERE id_producto = NEW.id_producto AND id_almacen = COALESCE(NEW.id_almacen, 0);
        
        UPDATE productos_stock
        SET costo_promedio = (
            (v_costo_promedio * (stock_actual - NEW.cantidad) + NEW.costo_unitario * NEW.cantidad)
            / NULLIF(stock_actual, 0)
        ),
        ultimo_costo = CASE WHEN NEW.cantidad > 0 THEN NEW.costo_unitario ELSE ultimo_costo END
        WHERE id_producto = NEW.id_producto AND id_almacen = COALESCE(NEW.id_almacen, 0);
    END IF;
    
    -- Actualizar fechas de última compra/venta
    IF NEW.tipo_movimiento = 'salida_venta' THEN
        UPDATE productos_stock
        SET fecha_ultima_venta = NEW.fecha_movimiento
        WHERE id_producto = NEW.id_producto AND id_almacen = COALESCE(NEW.id_almacen, 0);
    ELSEIF NEW.tipo_movimiento = 'entrada_compra' THEN
        UPDATE productos_stock
        SET fecha_ultima_compra = NEW.fecha_movimiento
        WHERE id_producto = NEW.id_producto AND id_almacen = COALESCE(NEW.id_almacen, 0);
    END IF;
END //
DELIMITER ;
```

## Vistas para Inventarios

### Stock General con Indicadores
```sql
CREATE VIEW vista_stock_general AS
SELECT 
    p.id_producto,
    p.sku,
    p.nombre_comercial,
    c.nombre AS categoria,
    ps.stock_actual,
    ps.stock_minimo,
    ps.stock_maximo,
    ps.punto_reorden,
    ps.costo_promedio,
    ps.stock_actual * ps.costo_promedio AS valor_inventario,
    CASE 
        WHEN ps.stock_actual <= 0 THEN 'Sin Stock'
        WHEN ps.stock_actual <= ps.punto_reorden THEN 'Punto de Reorden'
        WHEN ps.stock_actual <= ps.stock_minimo THEN 'Stock Mínimo'
        WHEN ps.stock_actual >= ps.stock_maximo THEN 'Sobre Stock'
        ELSE 'Normal'
    END AS estado_stock,
    ps.ultima_actualizacion
FROM productos p
JOIN categorias c ON p.id_categoria = c.id_categoria
JOIN productos_stock ps ON p.id_producto = ps.id_producto
WHERE p.activo = TRUE;
```

### Valor del Inventario
```sql
CREATE VIEW vista_valor_inventario AS
SELECT 
    a.nombre AS almacen,
    COUNT(DISTINCT ps.id_producto) AS productos_con_stock,
    SUM(ps.stock_actual) AS unidades_totales,
    ROUND(SUM(ps.stock_actual * ps.costo_promedio), 2) AS valor_costo,
    ROUND(SUM(ps.stock_actual * p.precio_venta), 2) AS valor_venta,
    ROUND(SUM(ps.stock_actual * (p.precio_venta - ps.costo_promedio)), 2) AS margen_potencial
FROM almacenes a
JOIN productos_stock ps ON a.id_almacen = ps.id_almacen
JOIN productos p ON ps.id_producto = p.id_producto
WHERE p.activo = TRUE
GROUP BY a.id_almacen;
```

## Datos de Prueba

```sql
-- Almacenes
INSERT INTO almacenes (id_sucursal, codigo, nombre, tipo) VALUES
    (1, 'ALM-PPAL', 'Almacén Principal', 'principal'),
    (1, 'ALM-SEC', 'Almacén Secundario', 'secundario'),
    (1, 'ALM-DEV', 'Almacén Devoluciones', 'devoluciones');

-- Stock inicial
INSERT INTO productos_stock (id_producto, id_almacen, stock_actual, stock_minimo, stock_maximo, punto_reorden, costo_promedio) VALUES
    (1, 1, 15, 5, 50, 10, 17500.00),
    (1, 2, 5, 2, 20, 5, 17600.00),
    (2, 1, 100, 20, 500, 50, 160.00),
    (3, 1, 200, 30, 500, 50, 230.00),
    (3, 2, 50, 10, 100, 20, 235.00);

-- Movimientos iniciales
INSERT INTO inventario_movimientos (id_producto, id_almacen, tipo_movimiento, cantidad, costo_unitario, existencia_antes, referencia_tipo, usuario) VALUES
    (1, 1, 'inventario_inicial', 15, 17500.00, 0, 'inicial', 'admin'),
    (2, 1, 'inventario_inicial', 100, 160.00, 0, 'inicial', 'admin'),
    (3, 1, 'inventario_inicial', 200, 230.00, 0, 'inicial', 'admin');
```

---

## Anterior: [03 Analisis Compras](../04-compras/03-analisis-compras.md)
## Siguiente: [02 Consultas Inventarios](../05-inventarios/02-consultas-inventarios.md)

[Volver al índice](../README.md)
