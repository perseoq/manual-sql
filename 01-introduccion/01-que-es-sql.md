# 1.1 ¿Qué es SQL?

## Historia y Evolución

Imagina que tienes un enorme archivero con miles de carpetas y cada carpeta contiene documentos de tus clientes. SQL es el lenguaje que te permite decirle al archivero: "tráeme todos los clientes que vivan en Monterrey y hayan comprado más de $5,000 este mes". SQL (Structured Query Language) es el lenguaje estándar para comunicarse con bases de datos relacionales. Fue desarrollado originalmente en IBM por Donald D. Chamberlin y Raymond F. Boyce en la década de 1970, basándose en el modelo relacional del matemático Edgar F. Codd.

Hoy en día, prácticamente todas las aplicaciones de negocio usan SQL: desde el POS de una tienda de abarrotes hasta el sistema bancario más grande del mundo. Es uno de los lenguajes de programación más valiosos porque no importa qué tecnología uses (PHP, Python, Java, C#), todas necesitan SQL para guardar y recuperar datos.

### Línea de Tiempo
- **1970**: Edgar Codd publica "A Relational Model of Data for Large Shared Data Banks"
- **1974**: Chamberlin y Boyce desarrollan SEQUEL (Structured English Query Language)
- **1979**: Oracle lanza el primer DBMS comercial basado en SQL
- **1986**: ANSI estandariza SQL por primera vez (SQL-86)
- **1989**: SQL-89, revisión menor
- **1992**: SQL-92 (SQL2), revisión mayor con joins, funciones de agregación
- **1999**: SQL:1999 (SQL3), introduce CTE, triggers, recursividad
- **2003**: SQL:2003, introduce window functions, XML
- **2008**: SQL:2008, mejoras en TRUNCATE, FETCH FIRST
- **2011**: SQL:2011, temporales, mejoras en window functions
- **2016**: SQL:2016, JSON, pattern matching, polimorfismo
- **2019**: SQL:2019, arrays multidimensionales, mejoras en JSON
- **2023**: SQL:2023, property graphs, mejoras en JSON y SQL/JSON

## ¿Por qué SQL en Ventas, Compras e Inventarios?

Los sistemas transaccionales (ERP, POS, e-commerce) generan millones de registros diarios: cada venta en caja, cada producto que llega al almacén, cada orden de compra a un proveedor. Sin SQL, toda esa información estaría dispersa en hojas de cálculo, notas adhesivas o en la memoria de los empleados. SQL permite centralizar, consultar y analizar todos esos datos de manera rápida y confiable.

### En Ventas

Imagina que eres gerente de una tienda con 10 vendedores y necesitas saber al cierre del día quién vendió más. Con SQL, escribes una consulta como la siguiente, que agrupa las facturas por vendedor, suma los totales y los ordena de mayor a menor:

```sql
SELECT 
    v.nombre_vendedor,       -- Mostramos el nombre del vendedor
    COUNT(f.id_factura) AS total_ventas,  -- Contamos cuántas facturas hizo
    SUM(f.total) AS monto_total            -- Sumamos el importe de todas sus ventas
FROM facturas f
JOIN vendedores v ON f.id_vendedor = v.id_vendedor
WHERE f.fecha = CURRENT_DATE              -- Solo las ventas de hoy
GROUP BY v.nombre_vendedor                -- Agrupamos por vendedor
ORDER BY monto_total DESC;                -- Ordenamos de mayor a menor
```

Esta consulta lee la tabla `facturas`, la combina con `vendedores` para obtener los nombres, filtra solo las facturas del día actual, agrupa por vendedor y calcula dos métricas: cuántas facturas emitió cada uno y por qué monto total. En una tienda con 50 ventas diarias, esta consulta se ejecuta en milisegundos.

### En Compras

Como encargado de compras, necesitas evaluar qué proveedores cumplen mejor los plazos de entrega. Esta consulta calcula el promedio de días que cada proveedor tarda en entregar, considerando solo aquellos con más de 5 órdenes en los últimos 6 meses:

```sql
SELECT 
    p.nombre_proveedor,
    COUNT(oc.id_orden) AS ordenes_emitidas,
    AVG(oc.dias_entrega) AS dias_promedio_entrega,
    SUM(oc.total) AS monto_total_compras
FROM ordenes_compra oc
JOIN proveedores p ON oc.id_proveedor = p.id_proveedor
WHERE oc.fecha_emision >= DATE_SUB(CURRENT_DATE, INTERVAL 6 MONTH)
GROUP BY p.id_proveedor
HAVING COUNT(oc.id_orden) > 5
ORDER BY dias_promedio_entrega ASC;
```

La función `AVG` calcula el promedio de los días de entrega. `HAVING` filtra los grupos después de agrupar (similar a WHERE pero para grupos). El resultado permite identificar qué proveedores son más confiables.

### En Inventarios

Para evitar quedarte sin stock, necesitas saber qué productos están por debajo de su mínimo. Esta consulta compara el stock actual contra el mínimo y calcula la diferencia:

```sql
SELECT 
    p.codigo_producto,
    p.nombre_producto,
    p.stock_actual,           -- Cuántas unidades tenemos hoy
    p.stock_minimo,           -- Cuántas unidades deberíamos tener como mínimo
    (p.stock_actual - p.stock_minimo) AS diferencia  -- Cuántas unidades faltan (negativo = faltante)
FROM productos p
WHERE p.stock_actual < p.stock_minimo  -- Solo productos con problemas
ORDER BY diferencia ASC;               -- Los más críticos primero
```

## Componentes del Lenguaje SQL

SQL se divide en varios sublenguajes, cada uno con un propósito específico. Piensa en ellos como diferentes herramientas en una caja de herramientas: cada una sirve para un tipo de operación distinto.

### DDL (Data Definition Language) - Definir la estructura

DDL crea y modifica la estructura de la base de datos: las tablas, sus columnas y las relaciones entre ellas. Es como el arquitecto que diseña los planos del edificio antes de construirlo.

```sql
-- CREATE: Crea una nueva tabla con sus columnas y tipos de datos
CREATE TABLE clientes (
    id_cliente INT PRIMARY KEY AUTO_INCREMENT,  -- Número único que se genera solo
    nombre VARCHAR(100) NOT NULL,                -- Texto de hasta 100 caracteres, obligatorio
    email VARCHAR(150) UNIQUE,                   -- Correo, no puede repetirse
    fecha_registro DATETIME DEFAULT CURRENT_TIMESTAMP  -- Fecha automática al crear
);

-- ALTER: Modifica una tabla existente (agrega una columna)
ALTER TABLE clientes ADD COLUMN telefono VARCHAR(20);

-- DROP: Elimina la tabla y todos sus datos (¡cuidado!)
DROP TABLE clientes_temporales;
```

### DML (Data Manipulation Language) - Manipular datos

DML trabaja con los datos dentro de las tablas: agregar, modificar y eliminar registros. Es como el albañil que pone los ladrillos y hace cambios en el edificio ya construido.

```sql
-- INSERT: Agrega un nuevo registro a la tabla
INSERT INTO clientes (nombre, email) VALUES ('Juan Pérez', 'juan@email.com');

-- UPDATE: Modifica registros existentes (SIEMPRE usar WHERE)
UPDATE clientes SET telefono = '+525512345678' WHERE id_cliente = 1;

-- DELETE: Elimina registros (SIEMPRE usar WHERE, o borrarás todo)
DELETE FROM clientes WHERE id_cliente = 100;
```

### DQL (Data Query Language) - Consultar datos

DQL es el sublenguaje más usado. Su único comando es SELECT, pero es extraordinariamente flexible. Permite desde consultas simples ("dame todos los clientes") hasta análisis complejos ("dame los 10 productos más vendidos por categoría en el último trimestre"). La mayoría de este manual se centra en DQL.

```sql
-- Consulta simple: todos los datos de clientes que se registraron después del 1 de enero de 2024
SELECT * FROM clientes WHERE fecha_registro > '2024-01-01';
```

### DCL (Data Control Language) - Controlar accesos

DCL gestiona quién puede hacer qué en la base de datos. Es crucial para la seguridad: no quieres que un cajero pueda borrar facturas o que un becario vea los salarios de los empleados.

```sql
-- GRANT: Dar permisos (SELECT y INSERT en la BD ventas al usuario analista)
GRANT SELECT, INSERT ON ventas.* TO 'analista'@'localhost';

-- REVOKE: Quitar permisos (ya no puede borrar)
REVOKE DELETE ON ventas.* FROM 'analista'@'localhost';
```

### TCL (Transaction Control Language) - Gestionar transacciones

TCL maneja grupos de operaciones que deben ejecutarse como una unidad: o todas se completan o ninguna. Por ejemplo, cuando registras una venta, necesitas: 1) crear la factura, 2) descontar del inventario, 3) registrar el pago. Si falla alguno de estos pasos, todos los demás deben deshacerse para no dejar datos inconsistentes.

```sql
START TRANSACTION;
    -- Paso 1: Descontar del inventario
    UPDATE inventario SET stock = stock - 1 WHERE id_producto = 100;
    -- Paso 2: Registrar la venta
    INSERT INTO ventas_detalle (id_factura, id_producto, cantidad) VALUES (5000, 100, 1);
-- Si todo salió bien, confirmamos los cambios
COMMIT;
-- Si algo falló, haríamos ROLLBACK en lugar de COMMIT
```

## Motores de Base de Datos Populares

Existen múltiples sistemas de bases de datos que entienden SQL, pero cada uno tiene sus particularidades. La tabla siguiente te ayuda a elegir cuál usar según tus necesidades:

| Motor | Tipo | Licencia | Ideal para | SQL Dialecto |
|-------|------|----------|------------|--------------|
| **MySQL** | Relacional | Open Source | Web, ERP pequeños-medianos | MySQL SQL |
| **PostgreSQL** | Objeto-Relacional | Open Source | Empresarial, GIS, analítica | PostgreSQL SQL |
| **SQL Server** | Relacional | Propietaria | Empresarial, BI, .NET | T-SQL |
| **Oracle** | Objeto-Relacional | Propietaria | Gran empresa, misión crítica | PL/SQL |
| **SQLite** | Embebida | Public Domain | Móvil, IoT, desarrollo | SQLite SQL |
| **MariaDB** | Relacional | Open Source | Web, drop-in MySQL | MariaDB SQL |

## Sintaxis Básica General

### Estructura de una Consulta SELECT

SELECT es el comando más importante. Su estructura general sigue este orden estricto (las cláusulas entre corchetes son opcionales):

```sql
SELECT [DISTINCT] columnas | expresiones | funciones  -- Qué quieres ver
FROM tabla(s)                                          -- De dónde sacas los datos
[JOIN ... ON condicion]                                 -- Cómo combinas tablas
[WHERE condicion(es)]                                   -- Qué filtros aplicas
[GROUP BY columna(s)]                                    -- Cómo agrupas los resultados
[HAVING condicion(es)]                                  -- Filtros sobre los grupos
[ORDER BY columna(s) [ASC|DESC]]                        -- Cómo ordenas
[LIMIT n [OFFSET m]];                                   -- Cuántos resultados quieres
```

**Nota importante para principiantes**: El orden en que escribes las cláusulas es importante. SQL espera que primero le digas qué columnas quieres ver (SELECT), luego de dónde (FROM), luego qué filtros (WHERE), etc. Si cambias el orden, obtendrás un error.

### Reglas Fundamentales que Debes Conocer

1. **SQL no distingue mayúsculas de minúsculas** en las palabras clave: `select * from clientes` funciona igual que `SELECT * FROM clientes`. Sin embargo, por convención, las palabras clave se escriben en MAYÚSCULAS y los nombres de columnas/tablas en minúsculas.

2. **Los textos se escriben entre comillas simples**: `'Juan Pérez'`, no "Juan Pérez". Las comillas dobles se usan para identificadores (nombres de columnas o tablas con espacios).

3. **Cada sentencia termina con punto y coma** (`;`). Si olvidas el punto y coma, la base de datos pensará que aún no has terminado de escribir.

4. **Los comentarios ayudan a documentar** tu código. Usa `--` para comentarios de una línea y `/* */` para múltiples líneas:

```sql
-- Este es un comentario de una línea
SELECT nombre, precio -- esto también es un comentario, pero al final de una línea
FROM productos;
/* Este es un comentario
   que puede ocupar varias líneas
   y es útil para explicaciones largas */
```

## Conceptos Clave para Entender tu Base de Datos

### Tablas y Relaciones

Piensa en una tabla como una hoja de cálculo de Excel:
- **Tabla**: Es una hoja completa (ej: "Clientes")
- **Columna**: Es un encabezado de columna en esa hoja (ej: "Nombre", "Email", "Teléfono")
- **Fila**: Es una fila con los datos de una persona (ej: todos los datos de María García)
- **Clave Primaria (PK)** : Es un identificador único que distingue a cada fila. Como el número de cédula de identidad: no hay dos filas con el mismo valor. Normalmente es un número que se auto-incrementa.
- **Clave Foránea (FK)** : Es una columna que referencia la clave primaria de otra tabla. Por ejemplo, en la tabla "Facturas", hay una columna `id_cliente` que dice "esta factura pertenece al cliente con ID 5".
- **Índice**: Es como el índice de un libro: acelera la búsqueda de datos. Si buscas clientes por email frecuentemente, creas un índice en la columna email para que la búsqueda sea instantánea.

### Normalización: Cómo Organizar tus Datos para Evitar Caos

La normalización es el proceso de organizar los datos para reducir la redundancia (no guardar la misma información en varios lugares). Se divide en niveles:

- **1FN (Primera Forma Normal)** : Cada celda debe contener un solo valor. No permits "María García, Juan Pérez" en una celda de nombres. Debes tener una fila por persona.
- **2FN (Segunda Forma Normal)** : Cada columna debe depender de toda la clave primaria, no solo de una parte. Si tu clave es compuesta (ej: id_factura + id_producto), cada columna debe depender de ambos.
- **3FN (Tercera Forma Normal)** : Las columnas deben depender directamente de la clave primaria, no de otras columnas. Por ejemplo, si tienes `id_cliente` y `nombre_cliente`, el nombre depende del id, no de otra columna.

### Modelo Entidad-Relación para Ventas

Este diagrama muestra cómo se relacionan las tablas principales de un sistema de ventas. Cada caja representa una tabla, y las flechas indican relaciones:

```
┌─────────────┐     ┌───────────────┐     ┌──────────────┐
│  CLIENTES   │     │   FACTURAS    │     │   PRODUCTOS  │
├─────────────┤     ├───────────────┤     ├──────────────┤
│ PK id_cli   │────>│ PK id_factura │     │ PK id_prod   │
│ nombre      │     │ FK id_cli     │<────│ nombre       │
│ email       │     │ FK id_prod    │     │ precio       │
│ telefono    │     │ fecha         │     │ stock        │
│ direccion   │     │ total         │     │ categoria    │
│ fecha_reg   │     │ estado        │     │              │
└─────────────┘     └───────────────┘     └──────────────┘
```

La flecha entre CLIENTES y FACTURAS significa "un cliente puede tener muchas facturas". La de FACTURAS a PRODUCTOS significa "en una factura se pueden incluir muchos productos y un producto puede aparecer en muchas facturas" (relación muchos-a-muchos).

## Buenas Prácticas desde el Inicio

✅ **Siempre usa parámetros** en lugar de concatenar strings en tu código. Esto previene SQL injection, uno de los ataques más peligrosos y comunes (lo veremos en detalle en la sección 7).

✅ **Nombra las tablas en plural** (`clientes`, `productos`, `facturas`) para que sea intuitivo que contienen múltiples registros.

✅ **Usa nombres descriptivos** y en español para los campos (`fecha_creacion` es mejor que `fc`). Tu yo del futuro te lo agradecerá.

✅ **Define constraints** (restricciones) como NOT NULL (campo obligatorio), UNIQUE (valor no repetible), CHECK (validación personalizada), FOREIGN KEY (relación entre tablas). Son tu red de seguridad.

✅ **Documenta tu esquema** con comentarios que expliquen el propósito de cada tabla y columna.

✅ **Elige tipos de datos adecuados**: usa DECIMAL para dinero (nunca FLOAT, porque tiene problemas de redondeo), INT para identificadores, DATE para fechas, VARCHAR para textos cortos, TEXT para descripciones largas.

---

## 📊 Ejemplo del Mundo Real: Esquema Inicial para un Negocio

El siguiente código crea la estructura completa para una tienda de retail. Cada CREATE TABLE está explicado para que entiendas qué hace cada parte:

```sql
-- Crear la base de datos (el contenedor de todas las tablas)
CREATE DATABASE tienda_retail;
USE tienda_retail;

-- Categorías de productos: permite clasificar los artículos
CREATE TABLE categorias (
    id_categoria INT PRIMARY KEY AUTO_INCREMENT,  -- ID único que se genera automáticamente
    nombre VARCHAR(100) NOT NULL,                  -- Nombre de la categoría (ej: "Electrónicos")
    descripcion TEXT,                               -- Explicación opcional de la categoría
    activo BOOLEAN DEFAULT TRUE,                    -- Si está activa o desactivada
    fecha_creacion DATETIME DEFAULT CURRENT_TIMESTAMP  -- Cuándo se creó
);

-- Productos: el catálogo de todo lo que vendemos
CREATE TABLE productos (
    id_producto INT PRIMARY KEY AUTO_INCREMENT,
    codigo_barras VARCHAR(50) UNIQUE,       -- Código de barras (único)
    nombre_producto VARCHAR(200) NOT NULL,    -- Nombre del producto
    id_categoria INT,                         -- A qué categoría pertenece
    precio_venta DECIMAL(10,2) NOT NULL,      -- Precio al público (DECIMAL evita errores de redondeo)
    costo_promedio DECIMAL(10,2),             -- Lo que nos costó en promedio
    stock_actual INT DEFAULT 0,               -- Cuántas unidades tenemos
    stock_minimo INT DEFAULT 10,              -- Mínimo antes de reordenar
    iva DECIMAL(4,2) DEFAULT 16.00,           -- Porcentaje de IVA (16% en México)
    activo BOOLEAN DEFAULT TRUE,
    FOREIGN KEY (id_categoria) REFERENCES categorias(id_categoria)  -- Relación con categorías
);

-- Clientes: quienes nos compran
CREATE TABLE clientes (
    id_cliente INT PRIMARY KEY AUTO_INCREMENT,
    nombre VARCHAR(150) NOT NULL,
    rfc VARCHAR(13) UNIQUE,                  -- RFC para facturación (único)
    email VARCHAR(150),
    telefono VARCHAR(20),
    direccion TEXT,
    fecha_registro DATETIME DEFAULT CURRENT_TIMESTAMP,
    credito_limite DECIMAL(12,2) DEFAULT 0,   -- Máximo que puede deber
    saldo_actual DECIMAL(12,2) DEFAULT 0       -- Cuánto debe actualmente
);

-- Vendedores: los empleados que realizan las ventas
CREATE TABLE vendedores (
    id_vendedor INT PRIMARY KEY AUTO_INCREMENT,
    codigo_empleado VARCHAR(20) UNIQUE,
    nombre_vendedor VARCHAR(150) NOT NULL,
    email VARCHAR(150),
    comision_porcentaje DECIMAL(5,2) DEFAULT 5.00,  -- % de comisión por venta
    activo BOOLEAN DEFAULT TRUE
);

-- Facturas (cabecera): la información general de cada venta
CREATE TABLE facturas (
    id_factura INT PRIMARY KEY AUTO_INCREMENT,
    folio VARCHAR(20) UNIQUE NOT NULL,         -- Número de factura (único)
    id_cliente INT NOT NULL,
    id_vendedor INT,
    fecha_emision DATETIME DEFAULT CURRENT_TIMESTAMP,
    subtotal DECIMAL(12,2),                    -- Total sin impuestos
    iva DECIMAL(12,2),                         -- Impuesto
    total DECIMAL(12,2),                       -- Total final
    metodo_pago ENUM('efectivo', 'tarjeta', 'transferencia', 'credito'),
    estado ENUM('activa', 'cancelada', 'devuelta') DEFAULT 'activa',
    FOREIGN KEY (id_cliente) REFERENCES clientes(id_cliente),
    FOREIGN KEY (id_vendedor) REFERENCES vendedores(id_vendedor)
);

-- Detalle de factura: cada producto incluido en la factura
CREATE TABLE facturas_detalle (
    id_detalle INT PRIMARY KEY AUTO_INCREMENT,
    id_factura INT NOT NULL,                    -- A qué factura pertenece
    id_producto INT NOT NULL,                   -- Qué producto es
    cantidad INT NOT NULL,                      -- Cuántas unidades
    precio_unitario DECIMAL(10,2) NOT NULL,      -- A qué precio se vendió
    descuento DECIMAL(10,2) DEFAULT 0,           -- Descuento aplicado
    importe DECIMAL(12,2),                       -- Total de la línea (cantidad * precio - descuento)
    FOREIGN KEY (id_factura) REFERENCES facturas(id_factura),
    FOREIGN KEY (id_producto) REFERENCES productos(id_producto)
);

-- Insertar datos de ejemplo para empezar a trabajar
INSERT INTO categorias (nombre) VALUES 
    ('Electrónicos'), ('Ropa'), ('Hogar'), ('Alimentos');

INSERT INTO productos (codigo_barras, nombre_producto, id_categoria, precio_venta, costo_promedio, stock_actual) VALUES
    ('7501234567890', 'Laptop HP ProBook', 1, 25000.00, 18000.00, 15),
    ('7501234567891', 'Mouse Inalámbrico', 1, 350.00, 180.00, 100),
    ('7501234567892', 'Camisa Algodón', 2, 450.00, 250.00, 200),
    ('7501234567893', 'Set Sábanas', 3, 890.00, 500.00, 45);

INSERT INTO clientes (nombre, rfc, email) VALUES
    ('María García López', 'GALM890101ABC', 'maria@email.com'),
    ('Carlos Rodríguez Pérez', 'ROPC850203DEF', 'carlos@email.com');

INSERT INTO vendedores (codigo_empleado, nombre_vendedor, email) VALUES
    ('V001', 'Ana Martínez', 'ana@tienda.com'),
    ('V002', 'Pedro Sánchez', 'pedro@tienda.com');
```

---

## Siguiente: [02 Tipos De Bases De Datos](../01-introduccion/02-tipos-de-bases-de-datos.md)

[Volver al índice](../README.md)
