# 3.1 Modelo de Datos para Ventas

## Arquitectura del Módulo de Ventas

El módulo de ventas es el núcleo de cualquier sistema transaccional. Administra el ciclo completo: cotización → pedido → facturación → cobro → devolución.

## Diagrama Entidad-Relación

```
┌─────────────────────────────────────────────────────────────────┐
│                        CLIENTES                                  │
├─────────────────────────────────────────────────────────────────┤
│ PK id_cliente                                                    │
│    codigo, nombre, rfc, email, telefono                          │
│    direccion, credito_limite, saldo_actual, activo               │
└────────────────┬────────────────────────────────────────────────┘
                 │ 1:N
                 ▼
┌─────────────────────────────────────────────────────────────────┐
│                        FACTURAS                                  │
├─────────────────────────────────────────────────────────────────┤
│ PK id_factura                                                    │
│    folio, uuid, serie, fecha_emision                             │
│    FK id_cliente, FK id_vendedor, FK id_sucursal                 │
│    subtotal, descuento, iva, total                               │
│    metodo_pago, forma_pago, estado                               │
└────────────────┬────────────────────────────────────────────────┘
                 │ 1:N
                 ▼
┌─────────────────────────────────────────────────────────────────┐
│                    FACTURAS_DETALLE                               │
├─────────────────────────────────────────────────────────────────┤
│ PK id_detalle                                                    │
│    FK id_factura, FK id_producto                                 │
│    cantidad, precio_unitario, descuento                          │
│    subtotal, iva, total                                          │
└─────────────────────────────────────────────────────────────────┘
                 │
                 │ N:1
                 ▼
┌─────────────────────────────────────────────────────────────────┐
│                        PRODUCTOS                                 │
├─────────────────────────────────────────────────────────────────┤
│ PK id_producto                                                   │
│    sku, codigo_barras, nombre, precio_venta, stock, activo       │
└─────────────────────────────────────────────────────────────────┘
```

## Tablas del Módulo de Ventas

### 1. Clientes

La tabla de **clientes** almacena la información maestra de todas las personas físicas o morales a las que se les vende. Cada cliente tiene un código único para identificarlo en el sistema, y su RFC debe ser único por requisitos fiscales. El campo `tipo_persona` (fisica/moral) es fundamental porque determina la estructura de datos requerida: las personas físicas tienen apellidos, mientras que las morales tienen razón social. El límite de crédito (`credito_limite`) y los días de crédito (`dias_credito`) controlan las condiciones de pago, y la lista de precios determina qué tarifa se aplica al cliente.

```sql
CREATE TABLE clientes (
    id_cliente INT PRIMARY KEY AUTO_INCREMENT,
    codigo VARCHAR(20) UNIQUE NOT NULL,
    tipo_persona ENUM('fisica', 'moral') NOT NULL,
    nombre VARCHAR(150) NOT NULL,
    apellido_paterno VARCHAR(100),
    apellido_materno VARCHAR(100),
    rfc VARCHAR(13) UNIQUE NOT NULL,
    curp VARCHAR(18),
    razon_social VARCHAR(200),
    email VARCHAR(150),
    telefono VARCHAR(20),
    celular VARCHAR(20),
    id_municipio INT,
    codigo_postal VARCHAR(10),
    direccion TEXT,
    fecha_registro DATETIME DEFAULT CURRENT_TIMESTAMP,
    credito_limite DECIMAL(12,2) DEFAULT 0,
    saldo_actual DECIMAL(12,2) DEFAULT 0,
    dias_credito INT DEFAULT 0,
    lista_precios ENUM('publico', 'mayoreo', 'distribuidor') DEFAULT 'publico',
    descuento_especial DECIMAL(5,2) DEFAULT 0,
    activo BOOLEAN DEFAULT TRUE,
    notas TEXT,
    FOREIGN KEY (id_municipio) REFERENCES municipios(id_municipio)
);
```

### 2. Facturas (Cabecera)

La tabla de **facturas** es la cabecera del documento fiscal. Cada factura representa una transacción de venta completa. El campo `folio` es único y se genera automáticamente mediante una secuencia (vista más adelante). El `uuid` almacena el identificador del CFDI (Comprobante Fiscal Digital) para facturación electrónica. La factura se relaciona con un cliente, un vendedor (quien realizó la venta) y una sucursal. Los campos `subtotal`, `descuento`, `iva` y `total` almacenan los montos calculados a partir del detalle, y `estado` controla el ciclo de vida: activa, cancelada, devuelta o pendiente de cobro.

```sql
CREATE TABLE facturas (
    id_factura INT PRIMARY KEY AUTO_INCREMENT,
    folio VARCHAR(30) UNIQUE NOT NULL,
    uuid VARCHAR(36) UNIQUE,
    serie VARCHAR(10),
    numero_factura INT,
    tipo_comprobante ENUM('I', 'E', 'N', 'P') DEFAULT 'I',
    id_cliente INT NOT NULL,
    id_vendedor INT,
    id_sucursal INT,
    fecha_emision DATETIME DEFAULT CURRENT_TIMESTAMP,
    fecha_vencimiento DATE,
    id_orden_compra INT,
    forma_pago VARCHAR(10),
    metodo_pago VARCHAR(20),
    num_parcialidad INT DEFAULT 1,
    moneda VARCHAR(3) DEFAULT 'MXN',
    tipo_cambio DECIMAL(10,6) DEFAULT 1,
    subtotal DECIMAL(12,2),
    descuento DECIMAL(12,2) DEFAULT 0,
    iva DECIMAL(12,2),
    ieps DECIMAL(12,2) DEFAULT 0,
    total DECIMAL(12,2),
    total_letra VARCHAR(500),
    estado ENUM('activa', 'cancelada', 'devuelta_parcial', 'devuelta_total', 'pendiente_cobro') DEFAULT 'activa',
    notas TEXT,
    usuario_creacion VARCHAR(100),
    fecha_creacion DATETIME DEFAULT CURRENT_TIMESTAMP,
    usuario_modificacion VARCHAR(100),
    fecha_modificacion DATETIME ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (id_cliente) REFERENCES clientes(id_cliente),
    FOREIGN KEY (id_vendedor) REFERENCES vendedores(id_vendedor),
    FOREIGN KEY (id_sucursal) REFERENCES sucursales(id_sucursal)
);
```

### 3. Facturas Detalle

La tabla **facturas_detalle** almacena cada línea de producto incluida en una factura. Es una relación muchos-a-muchos entre facturas y productos, donde cada registro representa un producto específico con su cantidad, precio unitario, descuento aplicado e importes calculados. La clave foránea `ON DELETE CASCADE` asegura que si se elimina una factura, automáticamente se eliminen todas sus líneas de detalle. El uso de `DECIMAL` para cantidades con precisión 4 decimales permite manejar productos que se venden por peso o medida (ej: 1.5 kg).

```sql
CREATE TABLE facturas_detalle (
    id_detalle INT PRIMARY KEY AUTO_INCREMENT,
    id_factura INT NOT NULL,
    id_producto INT NOT NULL,
    cantidad DECIMAL(12,4) NOT NULL,
    precio_unitario DECIMAL(12,2) NOT NULL,
    descuento DECIMAL(12,2) DEFAULT 0,
    iva DECIMAL(12,2),
    ieps DECIMAL(12,2) DEFAULT 0,
    subtotal DECIMAL(12,2),
    total DECIMAL(12,2),
    FOREIGN KEY (id_factura) REFERENCES facturas(id_factura) ON DELETE CASCADE,
    FOREIGN KEY (id_producto) REFERENCES productos(id_producto)
);
```

### 4. Facturas Pagos

Gestiona los pagos recibidos contra una factura. Una factura puede tener múltiples pagos (parcialidades), de ahí la relación 1:N entre facturas y pagos. Los campos `banco_origen` y `cuenta_origen` registran la cuenta bancaria desde la que se pagó, útiles para conciliación bancaria. El campo `estatus` controla si el pago está pendiente de verificación, aplicado o rechazado, permitiendo manejar pagos anticipados o cheques sin fondo.

```sql
CREATE TABLE facturas_pagos (
    id_pago INT PRIMARY KEY AUTO_INCREMENT,
    id_factura INT NOT NULL,
    fecha_pago DATETIME DEFAULT CURRENT_TIMESTAMP,
    monto DECIMAL(12,2) NOT NULL,
    forma_pago VARCHAR(10),
    referencia VARCHAR(100),
    banco_origen VARCHAR(100),
    cuenta_origen VARCHAR(50),
    banco_destino VARCHAR(100),
    cuenta_destino VARCHAR(50),
    estatus ENUM('pendiente', 'aplicado', 'rechazado') DEFAULT 'pendiente',
    FOREIGN KEY (id_factura) REFERENCES facturas(id_factura)
);
```

### 5. Devoluciones

Las devoluciones representan el proceso inverso a una venta. La tabla **devoluciones** registra la cabecera de la devolución, vinculada a la factura original. El campo `tipo` distingue entre devolución total (se devuelve todo) o parcial (solo algunos productos). El campo `estado` controla el flujo de aprobación: pendiente, aprobada, rechazada o procesada. La tabla **devoluciones_detalle** registra qué productos específicos y en qué cantidades se devuelven, referenciando el detalle original de la factura para mantener la trazabilidad.

```sql
CREATE TABLE devoluciones (
    id_devolucion INT PRIMARY KEY AUTO_INCREMENT,
    folio VARCHAR(30) UNIQUE NOT NULL,
    id_factura_original INT NOT NULL,
    fecha_devolucion DATETIME DEFAULT CURRENT_TIMESTAMP,
    motivo TEXT NOT NULL,
    tipo ENUM('total', 'parcial') NOT NULL,
    subtotal DECIMAL(12,2),
    iva DECIMAL(12,2),
    total DECIMAL(12,2),
    estado ENUM('pendiente', 'aprobada', 'rechazada', 'procesada') DEFAULT 'pendiente',
    usuario_autoriza VARCHAR(100),
    FOREIGN KEY (id_factura_original) REFERENCES facturas(id_factura)
);

CREATE TABLE devoluciones_detalle (
    id_detalle INT PRIMARY KEY AUTO_INCREMENT,
    id_devolucion INT NOT NULL,
    id_detalle_factura INT NOT NULL,
    cantidad_devuelta INT NOT NULL,
    precio_unitario DECIMAL(12,2),
    subtotal DECIMAL(12,2),
    iva DECIMAL(12,2),
    total DECIMAL(12,2),
    FOREIGN KEY (id_devolucion) REFERENCES devoluciones(id_devolucion),
    FOREIGN KEY (id_detalle_factura) REFERENCES facturas_detalle(id_detalle)
);
```

### 6. Cotizaciones

Las **cotizaciones** permiten a los vendedores generar presupuestos para clientes potenciales antes de la venta formal. Cada cotización tiene un folio único, fecha de validez (después de la cual expira), y un campo `probabilidad_cierre` que ayuda a pronosticar ventas futuras. Cuando una cotización es aceptada, su estado cambia a 'convertida' y se puede generar la factura correspondiente. La tabla **cotizaciones_detalle** almacena los productos cotizados con sus precios, permitiendo que el cliente vea el desglose antes de decidir la compra.

```sql
CREATE TABLE cotizaciones (
    id_cotizacion INT PRIMARY KEY AUTO_INCREMENT,
    folio VARCHAR(30) UNIQUE NOT NULL,
    id_cliente INT NOT NULL,
    id_vendedor INT,
    fecha_emision DATETIME DEFAULT CURRENT_TIMESTAMP,
    fecha_validez DATE,
    subtotal DECIMAL(12,2),
    descuento DECIMAL(12,2) DEFAULT 0,
    iva DECIMAL(12,2),
    total DECIMAL(12,2),
    estado ENUM('activa', 'aceptada', 'rechazada', 'vencida', 'convertida') DEFAULT 'activa',
    probabilidad_cierre INT DEFAULT 50,
    FOREIGN KEY (id_cliente) REFERENCES clientes(id_cliente),
    FOREIGN KEY (id_vendedor) REFERENCES vendedores(id_vendedor)
);

CREATE TABLE cotizaciones_detalle (
    id_detalle INT PRIMARY KEY AUTO_INCREMENT,
    id_cotizacion INT NOT NULL,
    id_producto INT NOT NULL,
    cantidad INT NOT NULL,
    precio_unitario DECIMAL(12,2) NOT NULL,
    descuento DECIMAL(12,2) DEFAULT 0,
    subtotal DECIMAL(12,2),
    iva DECIMAL(12,2),
    total DECIMAL(12,2),
    FOREIGN KEY (id_cotizacion) REFERENCES cotizaciones(id_cotizacion),
    FOREIGN KEY (id_producto) REFERENCES productos(id_producto)
);
```

## Índices para Ventas

Los índices son fundamentales para mantener el rendimiento del sistema a medida que crecen los datos. Aquí creamos índices en las columnas que se usan frecuentemente en consultas WHERE, JOIN y ORDER BY. Por ejemplo, `idx_facturas_fecha` acelera las consultas por rango de fechas (reportes diarios, mensuales), mientras que `idx_facturas_cliente` agiliza la búsqueda del historial de un cliente. El índice compuesto `idx_facturas_estado_fecha` es especialmente útil para consultas que filtran por estado y fecha simultáneamente, como "facturas activas del mes actual".

```sql
CREATE INDEX idx_clientes_rfc ON clientes(rfc);
CREATE INDEX idx_clientes_email ON clientes(email);
CREATE INDEX idx_clientes_codigo ON clientes(codigo);
CREATE INDEX idx_facturas_folio ON facturas(folio);
CREATE INDEX idx_facturas_fecha ON facturas(fecha_emision);
CREATE INDEX idx_facturas_cliente ON facturas(id_cliente);
CREATE INDEX idx_facturas_vendedor ON facturas(id_vendedor);
CREATE INDEX idx_facturas_estado_fecha ON facturas(estado, fecha_emision);
CREATE INDEX idx_facturas_detalle_factura ON facturas_detalle(id_factura);
CREATE INDEX idx_facturas_detalle_producto ON facturas_detalle(id_producto);
CREATE INDEX idx_cotizaciones_cliente ON cotizaciones(id_cliente);
CREATE INDEX idx_cotizaciones_vendedor ON cotizaciones(id_vendedor);
CREATE INDEX idx_cotizaciones_estado ON cotizaciones(estado);
```

## Vistas para Ventas

### Vista de Estado de Cuenta del Cliente

Esta vista consolida la información financiera de cada cliente en un solo lugar. Calcula el crédito disponible restando el saldo actual del límite de crédito, cuenta las facturas pendientes de pago, y muestra la próxima fecha de vencimiento. Es útil para el departamento de cobranza y para tomar decisiones sobre si aprobar una nueva venta a crédito. Al ser una vista, los datos siempre están actualizados porque la consulta se ejecuta en tiempo real contra las tablas base.

```sql
CREATE VIEW vista_estado_cuenta AS
SELECT 
    c.id_cliente,
    c.nombre,
    c.rfc,
    c.credito_limite,
    c.saldo_actual,
    c.credito_limite - c.saldo_actual AS credito_disponible,
    COUNT(f.id_factura) AS facturas_pendientes,
    COALESCE(SUM(CASE WHEN f.estado = 'activa' THEN f.total ELSE 0 END), 0) AS total_pendiente,
    MAX(f.fecha_emision) AS ultima_factura,
    MIN(CASE WHEN f.estado = 'activa' THEN f.fecha_vencimiento END) AS proximo_vencimiento
FROM clientes c
LEFT JOIN facturas f ON c.id_cliente = f.id_cliente AND f.estado IN ('activa', 'pendiente_cobro')
GROUP BY c.id_cliente;
```

### Vista de Productos Más Vendidos

Muestra un ranking de productos ordenados por ingresos generados. Incluye el número de veces que se ha vendido cada producto (en facturas distintas), las unidades totales vendidas y el precio promedio al que se ha vendido. Los gerentes de producto y compras usan esta vista para identificar los artículos más rentables y tomar decisiones sobre reposición de inventario y promociones.

```sql
CREATE VIEW vista_top_productos AS
SELECT 
    p.id_producto,
    p.nombre_comercial,
    c.nombre AS categoria,
    COUNT(DISTINCT f.id_factura) AS veces_vendido,
    SUM(fd.cantidad) AS unidades_vendidas,
    SUM(fd.total) AS ingresos_generados,
    ROUND(SUM(fd.total) / NULLIF(SUM(fd.cantidad), 0), 2) AS precio_promedio
FROM productos p
JOIN categorias c ON p.id_categoria = c.id_categoria
JOIN facturas_detalle fd ON p.id_producto = fd.id_producto
JOIN facturas f ON fd.id_factura = f.id_factura AND f.estado = 'activa'
GROUP BY p.id_producto
ORDER BY ingresos_generados DESC;
```

## Secuencia de Folios

En los sistemas de ventas es crítico generar folios consecutivos sin saltos ni duplicados, especialmente por requisitos fiscales. La tabla `secuencias_folios` actúa como un contador para cada tipo de documento (facturas, cotizaciones, devoluciones) y serie. La función `generar_folio()` incrementa atómicamente el consecutivo y construye el folio con el formato deseado. Por ejemplo, para facturas generaría "FA00000001", "FA00000002", etc. El uso de `LAST_INSERT_ID(consecutivo + 1)` garantiza que la actualización sea atómica, evitando folios duplicados en entornos concurrentes.

```sql
CREATE TABLE secuencias_folios (
    id_secuencia INT PRIMARY KEY AUTO_INCREMENT,
    tabla VARCHAR(50) NOT NULL,
    serie VARCHAR(10) DEFAULT '',
    prefijo VARCHAR(10) DEFAULT '',
    consecutivo INT DEFAULT 1,
    digitos INT DEFAULT 8,
    año YEAR DEFAULT YEAR(CURRENT_DATE),
    UNIQUE KEY (tabla, serie, año)
);

INSERT INTO secuencias_folios (tabla, serie, prefijo, consecutivo) VALUES
    ('facturas', 'A', 'F', 1),
    ('cotizaciones', 'A', 'COT', 1),
    ('devoluciones', 'A', 'DEV', 1);

DELIMITER //
CREATE FUNCTION generar_folio(p_tabla VARCHAR(50), p_serie VARCHAR(10))
RETURNS VARCHAR(30)
DETERMINISTIC
BEGIN
    DECLARE v_folio VARCHAR(30);
    DECLARE v_prefijo VARCHAR(10);
    DECLARE v_consecutivo INT;
    DECLARE v_digitos INT;
    
    UPDATE secuencias_folios 
    SET consecutivo = LAST_INSERT_ID(consecutivo + 1)
    WHERE tabla = p_tabla AND serie = p_serie AND año = YEAR(CURRENT_DATE);
    
    SELECT prefijo, consecutivo - 1, digitos 
    INTO v_prefijo, v_consecutivo, v_digitos
    FROM secuencias_folios 
    WHERE tabla = p_tabla AND serie = p_serie AND año = YEAR(CURRENT_DATE);
    
    SET v_folio = CONCAT(v_prefijo, p_serie, LPAD(v_consecutivo, v_digitos, '0'));
    
    RETURN v_folio;
END //
DELIMITER ;
```

## Datos de Prueba

```sql
-- Insertar clientes de prueba
INSERT INTO clientes (codigo, tipo_persona, nombre, rfc, email, telefono, credito_limite, dias_credito, lista_precios) VALUES
    ('CLI001', 'fisica', 'María García López', 'GALM890101ABC', 'maria.garcia@email.com', '5511112233', 50000, 30, 'publico'),
    ('CLI002', 'fisica', 'Juan Rodríguez Pérez', 'ROPJ850203DEF', 'juan.rodriguez@email.com', '5522223344', 100000, 45, 'mayoreo'),
    ('CLI003', 'moral', 'Comercializadora del Norte SA de CV', 'COM880101GHI', 'ventas@comercializadora.com', '5533334455', 500000, 60, 'distribuidor');

-- Insertar vendedores
INSERT INTO vendedores (codigo, nombre, email, comision_porcentaje, meta_mensual) VALUES
    ('V001', 'Ana Martínez López', 'ana.martinez@empresa.com', 5.00, 150000),
    ('V002', 'Pedro Sánchez García', 'pedro.sanchez@empresa.com', 4.50, 120000),
    ('V003', 'Laura Torres Ruiz', 'laura.torres@empresa.com', 5.50, 180000);
```

## Reglas de Negocio Implementadas en SQL

### Validar Crédito del Cliente

Esta consulta implementa una regla de negocio fundamental: verificar si un cliente tiene crédito disponible antes de autorizar una venta a crédito. Calcula el saldo disponible como la diferencia entre el límite de crédito y el saldo actual, y devuelve un mensaje claro indicando si puede o no comprar. Esta lógica se ejecuta antes de confirmar cualquier factura con método de pago 'credito'.

```sql
SELECT 
    id_cliente,
    nombre,
    credito_limite,
    saldo_actual,
    credito_limite - saldo_actual AS disponible,
    CASE 
        WHEN credito_limite - saldo_actual > 0 THEN 'Puede comprar'
        ELSE 'Crédito insuficiente'
    END AS estado_credito
FROM clientes
WHERE id_cliente = 1;
SELECT 
    id_cliente,
    nombre,
    credito_limite,
    saldo_actual,
    credito_limite - saldo_actual AS disponible,
    CASE 
        WHEN credito_limite - saldo_actual > 0 THEN 'Puede comprar'
        ELSE 'Crédito insuficiente'
    END AS estado_credito
FROM clientes
WHERE id_cliente = 1;
```

### Calcular Totales de Factura

Esta consulta valida la integridad de los datos comparando los totales calculados desde el detalle contra los totales almacenados en la cabecera de la factura. Es común que por errores de programa o manipulación directa en la base de datos, los totales de cabecera no coincidan con la suma del detalle. Esta validación detecta esas discrepancias para corregirlas a tiempo, especialmente importante antes de emitir facturas fiscales electrónicas.

```sql
SELECT 
    f.id_factura,
    f.folio,
    SUM(fd.subtotal) AS subtotal_calculado,
    SUM(fd.iva) AS iva_calculado,
    SUM(fd.total) AS total_calculado,
    f.subtotal AS subtotal_registrado,
    f.total AS total_registrado,
    CASE 
        WHEN SUM(fd.total) = f.total THEN 'OK'
        ELSE 'DIFERENCIA'
    END AS validacion
FROM facturas f
JOIN facturas_detalle fd ON f.id_factura = fd.id_factura
GROUP BY f.id_factura
HAVING validacion = 'DIFERENCIA';
SELECT 
    f.id_factura,
    f.folio,
    SUM(fd.subtotal) AS subtotal_calculado,
    SUM(fd.iva) AS iva_calculado,
    SUM(fd.total) AS total_calculado,
    f.subtotal AS subtotal_registrado,
    f.total AS total_registrado,
    CASE 
        WHEN SUM(fd.total) = f.total THEN 'OK'
        ELSE 'DIFERENCIA'
    END AS validacion
FROM facturas f
JOIN facturas_detalle fd ON f.id_factura = fd.id_factura
GROUP BY f.id_factura
HAVING validacion = 'DIFERENCIA';
```

---

## Anterior: [08 Indices Y Optimizacion](../02-fundamentos/08-indices-y-optimizacion.md)
## Siguiente: [02 Consultas Ventas](../03-ventas/02-consultas-ventas.md)

[Volver al índice](../README.md)
