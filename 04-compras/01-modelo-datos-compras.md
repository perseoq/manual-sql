# 4.1 Modelo de Datos para Compras

## Arquitectura del Módulo de Compras

El módulo de compras gestiona el ciclo de abastecimiento: solicitud → cotización → orden de compra → recepción → factura de proveedor → pago.

## Diagrama Entidad-Relación

```
┌───────────────┐     ┌───────────────────┐     ┌───────────────┐
│  PROVEEDORES   │     │  ORDENES_COMPRA   │     │  PRODUCTOS    │
├───────────────┤     ├───────────────────┤     ├───────────────┤
│ PK id_proveedor│────>│ PK id_orden       │<────│ PK id_producto│
│    rfc         │     │    folio           │     │    sku        │
│    nombre      │     │    FK id_proveedor │     │    nombre     │
│    telefono    │     │    fecha_emision   │     │    precio     │
│    email       │     │    fecha_entrega   │     └───────────────┘
│    dias_credito│     │    subtotal            │
│    activo      │     │    iva             │        1:N
└───────────────┘     │    total           │         │
        │              │    estado          │         │
        │              └────────┬──────────┘         │
        │                       │ 1:N                │
        │                       ▼                    │
        │              ┌───────────────────┐         │
        │              │ OC_DETALLE        │         │
        │              ├───────────────────┤         │
        │              │ PK id_detalle     │─────────┘
        │              │    FK id_orden    │
        │              │    FK id_producto │
        │              │    cantidad_sol   │
        │              │    cantidad_recib │
        │              │    precio_unitario│
        │              └───────────────────┘
```

## Tablas del Módulo de Compras

### 1. Proveedores

Esta tabla almacena el catálogo maestro de proveedores. Cada proveedor se identifica de forma única mediante `id_proveedor` como clave primaria autoincremental, y cuenta con un `codigo` interno único y un `rfc` único para fines fiscales. La tabla incluye información de contacto, condiciones comerciales como `dias_credito` y `credito_limite`, así como indicadores de rendimiento como `calificacion` y `plazo_entrega_promedio`. El campo `activo` permite deshabilitar proveedores sin eliminar su historial. La clave foránea hacia `municipios` permite segmentar reportes por ubicación geográfica.

```sql
CREATE TABLE proveedores (
    id_proveedor INT PRIMARY KEY AUTO_INCREMENT,
    codigo VARCHAR(20) UNIQUE NOT NULL,
    rfc VARCHAR(13) UNIQUE NOT NULL,
    nombre_comercial VARCHAR(150) NOT NULL,
    razon_social VARCHAR(200),
    email VARCHAR(150),
    telefono VARCHAR(20),
    sitio_web VARCHAR(255),
    id_municipio INT,
    direccion TEXT,
    condiciones_pago VARCHAR(100),
    dias_credito INT DEFAULT 30,
    credito_limite DECIMAL(12,2) DEFAULT 0,
    saldo_actual DECIMAL(12,2) DEFAULT 0,
    plazo_entrega_promedio INT DEFAULT 5,
    calificacion DECIMAL(2,1) DEFAULT 5.0,
    activo BOOLEAN DEFAULT TRUE,
    fecha_registro DATETIME DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (id_municipio) REFERENCES municipios(id_municipio)
);
```

### 2. Contactos de Proveedor

Esta tabla almacena múltiples personas de contacto por cada proveedor. La clave foránea `id_proveedor` relaciona cada contacto con su proveedor padre, y el campo booleano `es_principal` permite designar al contacto predeterminado para comunicaciones comerciales. Se incluye `whatsapp` como canal adicional de comunicación, una práctica común en entornos de abastecimiento en Latinoamérica.

```sql
CREATE TABLE proveedores_contactos (
    id_contacto INT PRIMARY KEY AUTO_INCREMENT,
    id_proveedor INT NOT NULL,
    nombre VARCHAR(150) NOT NULL,
    cargo VARCHAR(100),
    email VARCHAR(150),
    telefono VARCHAR(20),
    whatsapp VARCHAR(20),
    es_principal BOOLEAN DEFAULT FALSE,
    FOREIGN KEY (id_proveedor) REFERENCES proveedores(id_proveedor)
);
```

### 3. Productos de Proveedor (Catálogo)

Esta tabla implementa la relación muchos-a-muchos entre proveedores y productos, definiendo qué productos ofrece cada proveedor y a qué precio de compra. La clave primaria compuesta por `id_proveedor` e `id_producto` garantiza que no se duplique la asignación. El campo `codigo_proveedor` almacena la referencia que el proveedor usa internamente para cada producto, facilitando la comunicación en órdenes de compra.

```sql
CREATE TABLE proveedores_productos (
    id_proveedor INT NOT NULL,
    id_producto INT NOT NULL,
    codigo_proveedor VARCHAR(50),
    precio_compra DECIMAL(12,2) NOT NULL,
    plazo_entrega INT,
    descuento_volumen TEXT,
    PRIMARY KEY (id_proveedor, id_producto),
    FOREIGN KEY (id_proveedor) REFERENCES proveedores(id_proveedor),
    FOREIGN KEY (id_producto) REFERENCES productos(id_producto)
);
```

### 4. Órdenes de Compra

Esta tabla representa el documento central del proceso de compras. Cada orden se identifica con un `folio` único legible por humanos. El campo `estado` utiliza un ENUM que modela el ciclo de vida completo: desde `borrador` hasta `cerrada`. La orden registra las fechas clave (`fecha_emision`, `fecha_estimada_entrega`, `fecha_real_entrega`), condiciones financieras (`moneda`, `tipo_cambio` para compras internacionales), y los montos calculados (`subtotal`, `iva`, `total`, `gastos_envio`). Los campos `id_usuario_solicita` e `id_usuario_autoriza` implementan la segregación de funciones.

```sql
CREATE TABLE ordenes_compra (
    id_orden INT PRIMARY KEY AUTO_INCREMENT,
    folio VARCHAR(30) UNIQUE NOT NULL,
    id_proveedor INT NOT NULL,
    id_sucursal INT,
    id_usuario_solicita VARCHAR(100),
    id_usuario_autoriza VARCHAR(100),
    fecha_emision DATETIME DEFAULT CURRENT_TIMESTAMP,
    fecha_estimada_entrega DATE,
    fecha_real_entrega DATE,
    tipo_envio ENUM('terrestre', 'aereo', 'maritimo', 'propio') DEFAULT 'terrestre',
    condiciones_pago VARCHAR(200),
    moneda VARCHAR(3) DEFAULT 'MXN',
    tipo_cambio DECIMAL(10,6) DEFAULT 1,
    subtotal DECIMAL(12,2),
    iva DECIMAL(12,2),
    total DECIMAL(12,2),
    gastos_envio DECIMAL(12,2) DEFAULT 0,
    estado ENUM('borrador', 'pendiente_autorizacion', 'autorizada', 'enviada',
                'confirmada', 'recibida_parcial', 'recibida_total', 
                'cancelada', 'cerrada') DEFAULT 'borrador',
    notas TEXT,
    FOREIGN KEY (id_proveedor) REFERENCES proveedores(id_proveedor),
    FOREIGN KEY (id_sucursal) REFERENCES sucursales(id_sucursal)
);
```

### 5. Detalle de Orden de Compra

Esta tabla almacena las líneas o partidas de cada orden de compra. La columna generada `cantidad_pendiente` se calcula automáticamente como la diferencia entre lo solicitado y lo recibido, evitando inconsistencias. El uso de `ON DELETE CASCADE` asegura que al eliminar una orden se eliminen también sus detalles. Cada detalle registra el precio negociado (`precio_unitario`) y los descuentos aplicados, permitiendo rastrear el costo exacto por producto en cada orden.

```sql
CREATE TABLE ordenes_compra_detalle (
    id_detalle INT PRIMARY KEY AUTO_INCREMENT,
    id_orden INT NOT NULL,
    id_producto INT NOT NULL,
    cantidad_solicitada DECIMAL(12,4) NOT NULL,
    cantidad_recibida DECIMAL(12,4) DEFAULT 0,
    cantidad_pendiente DECIMAL(12,4) GENERATED ALWAYS AS (cantidad_solicitada - cantidad_recibida) STORED,
    precio_unitario DECIMAL(12,2) NOT NULL,
    descuento DECIMAL(12,2) DEFAULT 0,
    subtotal DECIMAL(12,2),
    iva DECIMAL(12,2),
    total DECIMAL(12,2),
    FOREIGN KEY (id_orden) REFERENCES ordenes_compra(id_orden) ON DELETE CASCADE,
    FOREIGN KEY (id_producto) REFERENCES productos(id_producto)
);
```

### 6. Recepciones de Compra

Estas tablas registran la entrada física de mercancía al almacén. La tabla `recepciones_compra` agrupa una recepción por orden de compra, mientras que `recepciones_compra_detalle` captura cada producto recibido con su lote, fecha de caducidad y ubicación de almacenamiento. Este diseño permite recepciones parciales (una orden puede recibirse en múltiples eventos) y mantiene la trazabilidad desde la orden hasta el inventario físico.

```sql
CREATE TABLE recepciones_compra (
    id_recepcion INT PRIMARY KEY AUTO_INCREMENT,
    folio VARCHAR(30) UNIQUE NOT NULL,
    id_orden INT NOT NULL,
    fecha_recepcion DATETIME DEFAULT CURRENT_TIMESTAMP,
    id_usuario_recibe VARCHAR(100),
    id_almacen INT,
    notas TEXT,
    estado ENUM('pendiente', 'completa', 'parcial') DEFAULT 'pendiente',
    FOREIGN KEY (id_orden) REFERENCES ordenes_compra(id_orden),
    FOREIGN KEY (id_almacen) REFERENCES almacenes(id_almacen)
);

CREATE TABLE recepciones_compra_detalle (
    id_detalle INT PRIMARY KEY AUTO_INCREMENT,
    id_recepcion INT NOT NULL,
    id_detalle_orden INT NOT NULL,
    cantidad_recibida DECIMAL(12,4) NOT NULL,
    lote VARCHAR(50),
    fecha_caducidad DATE,
    ubicacion VARCHAR(50),
    FOREIGN KEY (id_recepcion) REFERENCES recepciones_compra(id_recepcion),
    FOREIGN KEY (id_detalle_orden) REFERENCES ordenes_compra_detalle(id_detalle)
);
```

### 7. Facturas de Proveedor

Esta tabla almacena los documentos fiscales emitidos por los proveedores. El campo `uuid` almacena el UUID del CFDI (Comprobante Fiscal Digital por Internet) para facturas electrónicas mexicanas, garantizando la unicidad fiscal. La relación con `id_orden` vincula la factura con la orden de compra correspondiente, mientras que `fecha_vencimiento` determina la fecha límite de pago para calcular antigüedad de cuentas por pagar.

```sql
CREATE TABLE facturas_proveedor (
    id_factura_proveedor INT PRIMARY KEY AUTO_INCREMENT,
    folio_proveedor VARCHAR(50) NOT NULL,
    uuid VARCHAR(36) UNIQUE,
    id_proveedor INT NOT NULL,
    id_orden INT,
    fecha_emision DATE NOT NULL,
    fecha_vencimiento DATE,
    subtotal DECIMAL(12,2),
    iva DECIMAL(12,2),
    total DECIMAL(12,2),
    estado ENUM('pendiente', 'pagada', 'parcial', 'cancelada') DEFAULT 'pendiente',
    FOREIGN KEY (id_proveedor) REFERENCES proveedores(id_proveedor),
    FOREIGN KEY (id_orden) REFERENCES ordenes_compra(id_orden)
);
```

### 8. Pagos a Proveedores

Esta tabla registra los pagos realizados contra cada factura de proveedor. La clave foránea hacia `facturas_proveedor` permite relacionar múltiples pagos parciales a una misma factura. Los campos `forma_pago` y `referencia` capturan el método de pago (transferencia, cheque, efectivo) y el número de referencia bancaria, facilitando la conciliación bancaria.

```sql
CREATE TABLE pagos_proveedor (
    id_pago INT PRIMARY KEY AUTO_INCREMENT,
    id_factura_proveedor INT NOT NULL,
    fecha_pago DATETIME DEFAULT CURRENT_TIMESTAMP,
    monto DECIMAL(12,2) NOT NULL,
    forma_pago VARCHAR(20),
    referencia VARCHAR(100),
    cuenta_bancaria VARCHAR(50),
    FOREIGN KEY (id_factura_proveedor) REFERENCES facturas_proveedor(id_factura_proveedor)
);
```

## Índices para Compras

Los índices mejoran el rendimiento de las consultas más frecuentes en el módulo de compras. Se indexan `rfc` y `codigo` en proveedores para búsquedas exactas, `folio` en órdenes de compra para localización rápida, y las columnas usadas en filtros frecuentes como `id_proveedor`, `fecha_emision`, `estado` y `fecha_estimada_entrega`. Los índices en `recepciones_compra` y `facturas_proveedor` optimizan las búsquedas por orden y proveedor respectivamente.

```sql
CREATE INDEX idx_proveedores_rfc ON proveedores(rfc);
CREATE INDEX idx_proveedores_codigo ON proveedores(codigo);

CREATE INDEX idx_ordenes_folio ON ordenes_compra(folio);
CREATE INDEX idx_ordenes_proveedor ON ordenes_compra(id_proveedor);
CREATE INDEX idx_ordenes_fecha ON ordenes_compra(fecha_emision);
CREATE INDEX idx_ordenes_estado ON ordenes_compra(estado);
CREATE INDEX idx_ordenes_fecha_entrega ON ordenes_compra(fecha_estimada_entrega);

CREATE INDEX idx_recepciones_orden ON recepciones_compra(id_orden);
CREATE INDEX idx_facturas_proveedor_proveedor ON facturas_proveedor(id_proveedor);
```

## Vistas para Compras

### Órdenes Pendientes de Entrega

Esta vista muestra las órdenes de compra activas pendientes de recibir total o parcialmente. Calcula indicadores clave como `dias_restantes` para la fecha estimada de entrega, `porcentaje_recibido` para medir el avance de recepción, y la cantidad de productos pendientes. Es útil para dar seguimiento diario a proveedores y detectar retrasos en las entregas.

```sql
CREATE VIEW vista_ordenes_pendientes AS
SELECT 
    oc.id_orden,
    oc.folio,
    p.nombre_comercial AS proveedor,
    p.telefono,
    p.email,
    oc.fecha_emision,
    oc.fecha_estimada_entrega,
    DATEDIFF(oc.fecha_estimada_entrega, CURRENT_DATE) AS dias_restantes,
    oc.total,
    oc.estado,
    COUNT(ocd.id_detalle) AS productos_solicitados,
    SUM(ocd.cantidad_pendiente > 0) AS productos_pendientes,
    ROUND(SUM(ocd.cantidad_recibida) / NULLIF(SUM(ocd.cantidad_solicitada), 0) * 100, 2) AS porcentaje_recibido
FROM ordenes_compra oc
JOIN proveedores p ON oc.id_proveedor = p.id_proveedor
LEFT JOIN ordenes_compra_detalle ocd ON oc.id_orden = ocd.id_orden
WHERE oc.estado IN ('enviada', 'confirmada', 'recibida_parcial')
GROUP BY oc.id_orden;
```

### Productos para Reordenar

Esta vista identifica productos cuyo stock actual está por debajo del punto de reorden. Calcula la `cantidad_sugerida` a comprar y su `costo_sugerido`, y clasifica la urgencia en cuatro niveles: URGENTE (sin stock), REORDENAR (por debajo del punto de reorden), MÍNIMO (por debajo del stock mínimo) y OK. Integra datos de inventario, catálogo de productos y precios de proveedores en una sola consulta.

```sql
CREATE VIEW vista_punto_reorden AS
SELECT 
    p.id_producto,
    p.sku,
    p.nombre_comercial,
    c.nombre AS categoria,
    ps.stock_actual,
    ps.stock_minimo,
    ps.punto_reorden,
    ps.stock_maximo,
    pp.precio_compra,
    pp.id_proveedor,
    pr.nombre_comercial AS proveedor,
    pr.plazo_entrega_promedio,
    ps.punto_reorden - ps.stock_actual AS cantidad_sugerida,
    (ps.punto_reorden - ps.stock_actual) * pp.precio_compra AS costo_sugerido,
    CASE 
        WHEN ps.stock_actual <= 0 THEN 'URGENTE'
        WHEN ps.stock_actual <= ps.punto_reorden THEN 'REORDENAR'
        WHEN ps.stock_actual <= ps.stock_minimo THEN 'MÍNIMO'
        ELSE 'OK'
    END AS prioridad
FROM productos p
JOIN categorias c ON p.id_categoria = c.id_categoria
JOIN productos_stock ps ON p.id_producto = ps.id_producto
LEFT JOIN proveedores_productos pp ON p.id_producto = pp.id_producto
LEFT JOIN proveedores pr ON pp.id_proveedor = pr.id_proveedor
WHERE p.activo = TRUE AND p.controla_stock = TRUE
HAVING prioridad IN ('URGENTE', 'REORDENAR', 'MÍNIMO')
ORDER BY ps.stock_actual ASC;
```

## Datos de Prueba

```sql
INSERT INTO proveedores (codigo, rfc, nombre_comercial, razon_social, email, telefono, dias_credito, plazo_entrega_promedio, calificacion) VALUES
    ('PROV001', 'TEC880101ABC', 'Tecnología Global S.A.', 'Tecnología Global S.A. de C.V.', 'ventas@tecnologiaglobal.com', '5555112233', 45, 7, 4.5),
    ('PROV002', 'DIS850203DEF', 'Distribuidora Office', 'Distribuidora Office del Centro S.A.', 'pedidos@officecenter.com', '5555223344', 30, 3, 4.8),
    ('PROV003', 'EMP900304GHI', 'Empaques y Más', 'Empaques y Más S. de R.L.', 'ventas@empaquesymas.com', '5555334455', 60, 10, 4.2);

INSERT INTO proveedores_contactos (id_proveedor, nombre, cargo, email, telefono, es_principal) VALUES
    (1, 'Roberto Hernández', 'Gerente de Ventas', 'roberto@tecnologiaglobal.com', '5555112244', TRUE),
    (2, 'Sofía Torres', 'Ejecutiva de Cuenta', 'sofia@officecenter.com', '5555223355', TRUE);

-- Asignar productos a proveedores
INSERT INTO proveedores_productos (id_proveedor, id_producto, codigo_proveedor, precio_compra, plazo_entrega) VALUES
    (1, 1, 'HP-PROBOOK-450', 17500.00, 5),
    (1, 2, 'MS-OPTICAL-100', 160.00, 3),
    (2, 3, 'OF-CAMISA-001', 230.00, 2);
```

## Reglas de Negocio

### Autorización Automática por Monto

Este trigger BEFORE INSERT se ejecuta automáticamente al crear una orden de compra. Si el total es menor o igual a $10,000, la orden se autoriza automáticamente; de lo contrario, queda en estado `pendiente_autorizacion` para revisión manual. Esta regla de negocio agiliza las compras de bajo monto mientras mantiene control sobre las de alto valor.

```sql
DELIMITER //
CREATE TRIGGER trg_orden_compra_autorizacion
BEFORE INSERT ON ordenes_compra
FOR EACH ROW
BEGIN
    IF NEW.total <= 10000 THEN
        SET NEW.estado = 'autorizada';
    ELSE
        SET NEW.estado = 'pendiente_autorizacion';
    END IF;
END //
DELIMITER ;
```

### Actualizar Cantidad Recibida

Este procedimiento almacenado ejecuta la recepción de mercancía en una sola transacción: actualiza las cantidades recibidas en el detalle de la orden, registra el movimiento de inventario, recalcula el costo promedio ponderado, y actualiza el estado de la orden a `recibida_total` o `recibida_parcial` según corresponda. Centralizar esta lógica en un SP garantiza la consistencia de los datos.

```sql
DELIMITER //
CREATE PROCEDURE sp_recibir_orden_compra(
    IN p_id_orden INT,
    IN p_id_producto INT,
    IN p_cantidad_recibir DECIMAL(12,4),
    IN p_id_almacen INT
)
BEGIN
    DECLARE v_id_detalle INT;
    DECLARE v_cantidad_solicitada DECIMAL(12,4);
    DECLARE v_cantidad_recibida DECIMAL(12,4);
    DECLARE v_precio DECIMAL(12,2);
    
    -- Obtener detalle de la orden
    SELECT id_detalle, cantidad_solicitada, cantidad_recibida, precio_unitario
    INTO v_id_detalle, v_cantidad_solicitada, v_cantidad_recibida, v_precio
    FROM ordenes_compra_detalle
    WHERE id_orden = p_id_orden AND id_producto = p_id_producto;
    
    -- Actualizar cantidad recibida
    UPDATE ordenes_compra_detalle
    SET cantidad_recibida = cantidad_recibida + p_cantidad_recibir
    WHERE id_detalle = v_id_detalle;
    
    -- Registrar movimiento de inventario
    INSERT INTO inventario_movimientos (id_producto, id_almacen, tipo_movimiento, cantidad, costo_unitario, referencia_tipo, referencia_id)
    VALUES (p_id_producto, p_id_almacen, 'entrada_compra', p_cantidad_recibir, v_precio, 'orden_compra', p_id_orden);
    
    -- Actualizar stock
    UPDATE productos_stock
    SET stock_actual = stock_actual + p_cantidad_recibir,
        costo_promedio = (costo_promedio * stock_actual + v_precio * p_cantidad_recibir) / (stock_actual + p_cantidad_recibir)
    WHERE id_producto = p_id_producto AND id_almacen = p_id_almacen;
    
    -- Verificar si la orden está completamente recibida
    IF NOT EXISTS (
        SELECT 1 FROM ordenes_compra_detalle
        WHERE id_orden = p_id_orden AND cantidad_recibida < cantidad_solicitada
    ) THEN
        UPDATE ordenes_compra
        SET estado = 'recibida_total', fecha_real_entrega = CURRENT_DATE
        WHERE id_orden = p_id_orden;
    ELSE
        UPDATE ordenes_compra SET estado = 'recibida_parcial' WHERE id_orden = p_id_orden;
    END IF;
END //
DELIMITER ;
```

---

## Anterior: [04 Reportes Ventas](../03-ventas/04-reportes-ventas.md)
## Siguiente: [02 Consultas Compras](../04-compras/02-consultas-compras.md)

[Volver al índice](../README.md)
