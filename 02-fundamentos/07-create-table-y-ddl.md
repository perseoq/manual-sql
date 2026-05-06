# 2.7 CREATE TABLE y DDL

## Data Definition Language (DDL)

Hasta ahora has trabajado con datos que ya existen en tablas. Pero ¿cómo se crean esas tablas? ¿Quién define qué columnas tiene una tabla, de qué tipo son, qué restricciones tienen? Eso es trabajo del DDL (Data Definition Language) — el lenguaje que define la ESTRUCTURA de la base de datos.

DDL incluye comandos como CREATE (crear bases de datos, tablas, índices, vistas), ALTER (modificar estructuras existentes) y DROP (eliminar estructuras). Mientras que DML (INSERT, UPDATE, DELETE) trabaja con los DATOS dentro de las tablas, DDL trabaja con el CONTENEDOR de esos datos.

Piensa en DDL como los planos de arquitectura de un edificio (dónde van las paredes, las puertas, las ventanas), mientras que DML es la gente que entra y sale del edificio. Sin buenos planos, el edificio será un desastre.

## CREATE DATABASE

Antes de crear tablas, necesitas una base de datos donde ponerlas. CREATE DATABASE crea ese contenedor. Cada motor de base de datos tiene sus propias opciones, pero el concepto es el mismo.

Las opciones más importantes son:
- **CHARACTER SET** (o ENCODING): define el juego de caracteres. `utf8mb4` (MySQL) o `UTF8` (PostgreSQL) son los estándares modernos que soportan emojis y caracteres especiales.
- **COLLATE** (o LC_COLLATE): define las reglas de ordenamiento y comparación. `utf8mb4_unicode_ci` significa "case insensitive" (no distingue mayúsculas/minúsculas) y usa el estándar Unicode. `es_MX.UTF-8` es para español de México.

Elegir el collation correcto es importante: ¿Quieres que 'Á' y 'a' se consideren iguales? ¿Quieres que 'ñ' esté después de 'n'? El collation lo define.

```sql
-- MySQL
CREATE DATABASE sistema_ventas
    CHARACTER SET utf8mb4
    COLLATE utf8mb4_unicode_ci;

-- PostgreSQL
CREATE DATABASE sistema_ventas
    WITH ENCODING 'UTF8'
    LC_COLLATE 'es_MX.UTF-8'
    LC_CTYPE 'es_MX.UTF-8';

-- SQL Server
CREATE DATABASE SistemaVentas;
```

## CREATE TABLE

### Sintaxis Completa

CREATE TABLE es el corazón del DDL. Aquí es donde defines la estructura de tus tablas. La sintaxis general es:

`CREATE TABLE [IF NOT EXISTS] nombre_tabla ( definiciones_de_columnas, constraints_de_tabla )`

El `IF NOT EXISTS` evita un error si la tabla ya existe. Simplemente no hace nada si ya está creada. Es útil en scripts que se ejecutan múltiples veces.

Cada columna se define con: `nombre tipo [constraints] [DEFAULT valor]`. Los constraints (restricciones) son reglas que los datos deben cumplir, como NOT NULL (no puede estar vacío), UNIQUE (no puede repetirse), o DEFAULT (valor por defecto si no se especifica).

```sql
CREATE TABLE [IF NOT EXISTS] nombre_tabla (
    columna1 tipo_dato [constraints] [DEFAULT valor],
    columna2 tipo_dato [constraints],
    ...,
    [constraints_de_tabla]
) [opciones_de_motor];
```

### Tipos de Datos Principales

#### Numéricos

Elegir el tipo numérico correcto es importante para la precisión y el espacio. Aquí tienes los principales:

- **INT**: el entero estándar. Sirve para IDs, contadores, y la mayoría de números enteros. Rango: aproximadamente -2 mil millones a +2 mil millones.
- **BIGINT**: cuando INT se queda corto (por ejemplo, IDs de transacciones bancarias o tablas con miles de millones de registros).
- **SMALLINT** y **TINYINT**: para rangos pequeños. TINYINT es ideal para banderas (0/1) o valores pequeños como edad, calificaciones, etc. Ocupan menos espacio que INT.
- **DECIMAL(p, s)**: para dinero y valores que deben ser EXACTOS. El parámetro p es la precisión (total de dígitos) y s es la escala (dígitos después del punto). `DECIMAL(12,2)` significa 10 dígitos enteros + 2 decimales. NUNCA uses FLOAT para dinero — los errores de redondeo pueden costarte dinero real.
- **FLOAT** y **REAL**: para valores científicos o aproximados donde un pequeño error de redondeo es aceptable. No uses para precios o cantidades contables.

```sql
INT            -- Entero estándar (-2^31 a 2^31-1)
BIGINT         -- Entero grande (-2^63 a 2^63-1)
SMALLINT       -- Entero pequeño (-32768 a 32767)
TINYINT        -- Entero muy pequeño (-128 a 127)
DECIMAL(p,s)   -- Decimal exacto (p=precisión, s=escala)
NUMERIC(p,s)   -- Similar a DECIMAL
FLOAT          -- Punto flotante (aproximado)
REAL           -- Punto flotante (menor precisión)
```

#### Cadenas de Texto

- **CHAR(n)**: longitud fija. Siempre ocupa n caracteres, aunque guardes solo 1. Ideal para códigos de longitud fija como RFC (13 caracteres), códigos postales, ISO de países (3 caracteres). Rápido pero derrochador si el texto varía mucho.
- **VARCHAR(n)**: longitud variable. Solo ocupa el espacio necesario (más 1-2 bytes de overhead). Ideal para nombres, direcciones, emails. n es el máximo. En MySQL, el máximo es 65,535 (pero limitado por el tamaño de fila).
- **TEXT**: para textos largos como descripciones de productos, notas, comentarios. No tiene un rendimiento tan bueno como VARCHAR para búsquedas, pero permite almacenar mucho texto.
- **MEDIUMTEXT** y **LONGTEXT**: para textos aún más grandes como artículos, documentos, o incluso libros completos.

Regla práctica: si el texto es corto y variable, usa VARCHAR. Si es largo, usa TEXT. Si es de longitud fija, CHAR.

```sql
CHAR(n)        -- Fijo, longitud n (máx 255)
VARCHAR(n)     -- Variable, max n (máx 65,535 en MySQL)
TEXT           -- Texto largo (máx 65,535)
MEDIUMTEXT     -- Texto mediano (máx 16,777,215)
LONGTEXT       -- Texto muy largo (máx 4,294,967,295)
```

#### Fechas y Tiempo

- **DATE**: solo la fecha (YYYY-MM-DD). Para cumpleaños, fechas de registro sin hora.
- **TIME**: solo la hora (HH:MI:SS). Para horarios de apertura, duraciones.
- **DATETIME**: fecha y hora combinadas. El estándar para la mayoría de casos: cuándo se registró un cliente, cuándo se emitió una factura.
- **TIMESTAMP**: como DATETIME pero con soporte de timezone y un rango más limitado (1970-2038 en MySQL). Además, se actualiza automáticamente con CURRENT_TIMESTAMP si lo configuras. Ideal para "última modificación".
- **YEAR**: solo el año. Para tablas históricas, modelos de autos, etc.

La gran diferencia entre DATETIME y TIMESTAMP: TIMESTAMP se almacena internamente como segundos desde 1970 (UTC) y se convierte a la zona horaria de la conexión al leerlo. DATETIME se almacena tal cual sin conversión de zona horaria.

```sql
DATE           -- Solo fecha (YYYY-MM-DD)
TIME           -- Solo hora (HH:MI:SS)
DATETIME       -- Fecha y hora (YYYY-MM-DD HH:MI:SS)
TIMESTAMP      -- Fecha y hora con timezone
YEAR           -- Solo año
```

#### Booleanos y Especiales

- **BOOLEAN**: verdadero/falso. En MySQL es un alias de TINYINT(1) (0 = false, 1 = true). En PostgreSQL es un tipo real.
- **ENUM**: una lista de valores permitidos (solo MySQL). Ej: `ENUM('activo', 'inactivo', 'suspendido')`. Útil pero ten cuidado: cambiar los valores permitidos después requiere ALTER TABLE.
- **SET**: múltiples valores de una lista (solo MySQL). Ej: `SET('rojo', 'verde', 'azul')` permite guardar 'rojo,verde'.
- **JSON**: datos en formato JSON. Disponible en MySQL 5.7+ y PostgreSQL 9.2+. Permite guardar documentos estructurados y consultar dentro de ellos con funciones como `JSON_EXTRACT()`.
- **UUID**: identificador único universal. Mejor que AUTO_INCREMENT para sistemas distribuidos porque no colisionan entre diferentes servidores.

```sql
BOOLEAN        -- TRUE o FALSE (en MySQL es TINYINT(1))
ENUM('a','b')  -- Un valor de una lista (MySQL)
SET('a','b')   -- Múltiples valores de una lista (MySQL)
JSON           -- Datos JSON (MySQL 5.7+, PostgreSQL 9.2+)
UUID           -- Identificador único universal
```

### Constraints (Restricciones)

Las restricciones son las REGLAS del negocio que se aplican a nivel de base de datos. Son importantísimas porque garantizan la integridad de los datos aunque la aplicación tenga errores.

Analicemos cada restricción en el ejemplo:

- **PRIMARY KEY** en `id_empleado`: identifica de manera ÚNICA cada fila. No puede haber dos empleados con el mismo ID, y nunca puede ser NULL. Toda tabla DEBE tener una primary key (aunque SQL no te obligue, es una mala práctica no tenerla).

- **UNIQUE** en `codigo` y `email`: garantiza que no haya duplicados. Dos empleados no pueden tener el mismo código de empleado ni el mismo email. A diferencia de PRIMARY KEY, UNIQUE permite múltiples valores NULL (en MySQL, una fila NULL; en PostgreSQL, múltiples NULLs).

- **NOT NULL** en `nombre`: obligatorio. No puedes crear un empleado sin nombre. El negocio lo exige.

- **CHECK** en `edad` y `salario`: validación personalizada. `CHECK (edad >= 18)` garantiza que el empleado sea mayor de edad. `CONSTRAINT ck_salario CHECK (salario >= 10000)` (con nombre explícito) garantiza salario mínimo. Si intentas insertar edad=15 o salario=5000, la base de datos lo RECHAZA.

- **DEFAULT**: valor por defecto. Si no especificas salario, será 0. Si no especificas fecha_ingreso, será la fecha actual. Si no especificas activo, será TRUE.

- **FOREIGN KEY**: la restricción más importante para mantener la integridad referencial. Garantiza que `id_departamento` en empleados SIEMPRE corresponda a un departamento que existe en la tabla departamentos. No puedes asignar un empleado a un departamento que no existe. Las acciones `ON DELETE SET NULL` y `ON UPDATE CASCADE` definen qué pasa cuando el departamento referenciado se elimina o actualiza.

```sql
CREATE TABLE empleados (
    id_empleado INT PRIMARY KEY,                    -- Clave primaria
    codigo VARCHAR(20) UNIQUE NOT NULL,             -- Único y no nulo
    nombre VARCHAR(100) NOT NULL,                   -- No nulo
    email VARCHAR(150) UNIQUE,                      -- Único
    edad INT CHECK (edad >= 18),                    -- Validación
    salario DECIMAL(12,2) DEFAULT 0,                -- Valor por defecto
    id_departamento INT,
    fecha_ingreso DATE DEFAULT CURRENT_DATE,        -- Fecha actual por defecto
    activo BOOLEAN DEFAULT TRUE,
    FOREIGN KEY (id_departamento)                   -- Clave foránea
        REFERENCES departamentos(id_departamento)
        ON DELETE SET NULL                          -- Acción en borrado
        ON UPDATE CASCADE,                         -- Acción en actualización
    CONSTRAINT ck_salario CHECK (salario >= 10000)  -- Constraint con nombre
);
```

### Acciones Referenciales (Foreign Keys)

Cuando defines una clave foránea, debes decidir QUÉ PASA cuando la fila referenciada se elimina o actualiza. Esto es crítico para la integridad de los datos:

- **ON DELETE CASCADE**: si eliminas el departamento, TODOS los empleados de ese departamento se eliminan automáticamente. Útil para relaciones padre-hijo donde el hijo no tiene sentido sin el padre (ej: factura y sus líneas de detalle).

- **ON DELETE SET NULL**: si eliminas el departamento, los empleados de ese departamento quedan con `id_departamento = NULL`. El empleado no se elimina, pero pierde su asignación. Útil cuando la relación es opcional.

- **ON DELETE RESTRICT** (default): NO PERMITE eliminar el departamento si tiene empleados. La base de datos lanza un error. Es la opción más segura.

- **ON DELETE NO ACTION**: similar a RESTRICT. En algunos motores, la diferencia es cuándo se evalúa la restricción (inmediata vs diferida).

- **ON UPDATE CASCADE**: si cambias el ID del departamento (raro, pero posible), los empleados se actualizan automáticamente al nuevo ID.

En el ejemplo de `facturas_detalle`:
- Si se elimina una factura (`ON DELETE CASCADE`), sus líneas de detalle se eliminan automáticamente. Tiene sentido: si no existe la factura, sus líneas sobran.
- Si se intenta eliminar un producto (`ON DELETE RESTRICT`), la base de datos lo IMPIDE si hay líneas de detalle que lo referencien. No puedes borrar un producto que ya se ha vendido.

```sql
ON DELETE CASCADE     → Eliminar en cascada
ON DELETE SET NULL    → Poner NULL en los hijos
ON DELETE RESTRICT    → No permitir eliminar (default)
ON DELETE NO ACTION   → Similar a RESTRICT
ON UPDATE CASCADE     → Actualizar en cascada
ON UPDATE SET NULL    → Poner NULL al actualizar
```

Ejemplo:
```sql
CREATE TABLE facturas_detalle (
    id_detalle INT PRIMARY KEY AUTO_INCREMENT,
    id_factura INT NOT NULL,
    id_producto INT NOT NULL,
    cantidad INT NOT NULL,
    FOREIGN KEY (id_factura) REFERENCES facturas(id_factura)
        ON DELETE CASCADE,              -- Si se elimina factura, elimina detalle
    FOREIGN KEY (id_producto) REFERENCES productos(id_producto)
        ON DELETE RESTRICT              -- No permitir eliminar producto si tiene ventas
);
```

## ALTER TABLE - Modificar Tablas

Las bases de datos no son estáticas. A medida que el negocio evoluciona, necesitas modificar la estructura de las tablas. ALTER TABLE es el comando para hacerlo.

### Agregar Columnas

ADD COLUMN agrega nuevas columnas a una tabla existente. En MySQL, puedes especificar la posición con AFTER. Si no usas AFTER, la columna se agrega al final.

El primer ejemplo agrega una columna `whatsapp` DESPUÉS de la columna `telefono`. El segundo ejemplo agrega varias columnas a la vez, lo que es más eficiente que hacer ALTER TABLE múltiples veces.

IMPORTANTE: Si la tabla ya tiene datos, la nueva columna se llenará con NULL (o con el DEFAULT que especifiques). ALTER TABLE en tablas grandes puede ser muy lento y bloquear la tabla durante la operación.

```sql
ALTER TABLE clientes
ADD COLUMN whatsapp VARCHAR(20) AFTER telefono;

ALTER TABLE clientes
ADD COLUMN (
    fecha_nacimiento DATE,
    genero ENUM('M', 'F', 'Otro'),
    notas TEXT
);
```

### Modificar Columnas

A veces necesitas cambiar el tipo de una columna, su nombre, o sus restricciones. Cada motor tiene su sintaxis:

En MySQL:
- **MODIFY COLUMN**: cambia el tipo y/o restricciones de una columna existente (sin cambiar el nombre).
- **CHANGE COLUMN**: cambia el nombre Y el tipo al mismo tiempo.

En PostgreSQL:
- **ALTER COLUMN TYPE**: cambia el tipo de la columna.
- **RENAME COLUMN**: cambia solo el nombre.

⚠️ Cambiar el tipo de una columna existente puede fallar si los datos actuales no son compatibles con el nuevo tipo. Por ejemplo, cambiar VARCHAR a INT fallará si hay texto no numérico.

```sql
-- MySQL
ALTER TABLE productos
MODIFY COLUMN precio_venta DECIMAL(12,2) NOT NULL;

ALTER TABLE productos
CHANGE COLUMN precio_venta precio_final DECIMAL(12,2);

-- PostgreSQL
ALTER TABLE productos
ALTER COLUMN precio_venta TYPE DECIMAL(12,2);

ALTER TABLE productos
RENAME COLUMN precio_venta TO precio_final;
```

### Eliminar Columnas

DROP COLUMN elimina una columna y TODOS sus datos. No hay deshacer (a menos que tengas backup). Si la columna tenía datos, se pierden para siempre.

`DROP COLUMN IF EXISTS` (MySQL 8.0+) evita un error si la columna no existe. Esto es útil en scripts que se ejecutan en diferentes versiones del esquema.

```sql
ALTER TABLE productos DROP COLUMN descripcion_antigua;
ALTER TABLE productos DROP COLUMN IF EXISTS campo_temporal;
```

### Agregar/Eliminar Constraints

Puedes agregar o eliminar restricciones después de crear la tabla. Esto es útil cuando:
- Creaste la tabla sin constraints y ahora necesitas agregarlos.
- Una restricción está causando problemas y necesitas eliminarla temporalmente.
- Descubriste que necesitas una clave foránea que no definiste inicialmente.

Para agregar: `ADD PRIMARY KEY`, `ADD FOREIGN KEY`, `ADD UNIQUE INDEX`, `ADD CONSTRAINT ... CHECK`.
Para eliminar: `DROP PRIMARY KEY`, `DROP FOREIGN KEY`, `DROP INDEX`, `DROP CONSTRAINT`.

El nombre de los constraints es importante: cuando eliminas una foreign key, DEBES saber su nombre (por ejemplo, `fk_factura`). Si no le pusiste nombre, el motor genera uno automático que puede ser difícil de adivinar.

```sql
-- Agregar clave primaria
ALTER TABLE productos ADD PRIMARY KEY (id_producto);

-- Agregar clave foránea
ALTER TABLE facturas_detalle
ADD CONSTRAINT fk_factura FOREIGN KEY (id_factura)
REFERENCES facturas(id_factura);

-- Agregar unique
ALTER TABLE productos ADD UNIQUE INDEX idx_sku (sku);

-- Agregar check
ALTER TABLE productos ADD CONSTRAINT ck_precio CHECK (precio_venta > 0);

-- Eliminar constraint
ALTER TABLE productos DROP INDEX idx_sku;
ALTER TABLE productos DROP FOREIGN KEY fk_factura;
ALTER TABLE productos DROP CONSTRAINT ck_precio;
```

## DROP TABLE - Eliminar Tablas

DROP TABLE elimina la tabla y TODOS sus datos permanentemente. No hay confirmación, no hay papelera de reciclaje. Úsalo con EXTREMO cuidado.

`IF EXISTS` evita un error si la tabla no existe. Puedes eliminar múltiples tablas separándolas con comas.

⚠️ Si hay claves foráneas que apuntan a esta tabla, DROP TABLE fallará a menos que elimines primero las tablas dependientes o uses el cascade correspondiente.

```sql
DROP TABLE tabla_temporal;              -- Eliminar tabla
DROP TABLE IF EXISTS tabla_temporal;    -- Eliminar si existe (sin error)
DROP TABLE tabla1, tabla2, tabla3;      -- Eliminar múltiples
```

## CREATE INDEX - Índices

Los índices son estructuras que aceleran las consultas. Los verás a detalle en el siguiente capítulo, pero aquí tienes una introducción a cómo se crean:

- **Índice simple**: en una sola columna. Útil para búsquedas por esa columna.
- **Índice único**: además de acelerar búsquedas, garantiza que no haya valores duplicados. Útil para RFC, email, etc.
- **Índice compuesto**: en múltiples columnas. El orden de las columnas importa (se explica en el siguiente capítulo).
- **FULLTEXT**: para búsqueda de texto completo. Permite buscar palabras dentro de texto largo de manera eficiente, usando MATCH...AGAINST.
- **Índice parcial** (PostgreSQL): solo indexa las filas que cumplen una condición. Así el índice es más pequeño y rápido.

```sql
-- Índice simple
CREATE INDEX idx_apellido ON clientes(apellido);

-- Índice único
CREATE UNIQUE INDEX idx_rfc ON clientes(rfc);

-- Índice compuesto (múltiples columnas)
CREATE INDEX idx_nombre_categoria ON productos(nombre_producto, id_categoria);

-- Índice de texto completo (búsqueda textual)
CREATE FULLTEXT INDEX idx_busqueda ON productos(nombre_producto, descripcion);

-- Índice parcial (PostgreSQL)
CREATE INDEX idx_activos ON clientes(id_cliente) WHERE activo = TRUE;
```

## CREATE VIEW - Vistas

Una vista es como una "consulta guardada" que se comporta como una tabla virtual. No almacena datos (a menos que sea materializada), sino que ejecuta la consulta subyacente cada vez que la usas.

¿Para qué sirven?
- **Simplificar consultas complejas**: si siempre haces el mismo JOIN de 5 tablas, crea una vista y selecciona de ella.
- **Seguridad**: puedes dar acceso a la vista sin dar acceso a las tablas subyacentes. Por ejemplo, una vista que muestra empleados sin incluir salarios.
- **Consistencia**: todos los reportes usan la misma lógica definida en la vista.

La vista `vista_ventas_resumidas` une facturas, clientes, vendedores y detalle, y agrupa por factura. Una vez creada, los usuarios pueden hacer `SELECT * FROM vista_ventas_resumidas WHERE fecha >= '2024-01-01'` sin preocuparse por los JOINs.

```sql
CREATE VIEW vista_ventas_resumidas AS
SELECT 
    DATE(f.fecha_emision) AS fecha,
    c.nombre AS cliente,
    v.nombre_vendedor AS vendedor,
    COUNT(fd.id_detalle) AS articulos,
    SUM(fd.cantidad) AS piezas,
    f.total
FROM facturas f
JOIN clientes c ON f.id_cliente = c.id_cliente
JOIN vendedores v ON f.id_vendedor = v.id_vendedor
JOIN facturas_detalle fd ON f.id_factura = fd.id_factura
WHERE f.estado = 'activa'
GROUP BY f.id_factura;

-- Usar la vista
SELECT * FROM vista_ventas_resumidas WHERE fecha >= '2024-01-01';
```

### Vistas Materializadas (PostgreSQL)

Las vistas normales ejecutan la consulta cada vez que las usas. Las vistas materializadas, en cambio, almacenan FÍSICAMENTE el resultado. Son más rápidas para consultar (porque los datos ya están calculados) pero más lentas de mantener (porque hay que refrescarlas).

Son ideales para reportes pesados que no necesitan estar 100% actualizados: ventas mensuales, resúmenes de inventario, indicadores de rendimiento.

`REFRESH MATERIALIZED VIEW` vuelve a ejecutar la consulta y actualiza los datos almacenados. Puedes programarlo para que se ejecute cada noche, cada hora, o según necesites.

```sql
CREATE MATERIALIZED VIEW mv_ventas_mensuales AS
SELECT 
    DATE_TRUNC('month', fecha_emision) AS mes,
    COUNT(*) AS facturas,
    SUM(total) AS ingresos
FROM facturas
WHERE estado = 'activa'
GROUP BY DATE_TRUNC('month', fecha_emision)
WITH DATA;

-- Actualizar vista materializada
REFRESH MATERIALIZED VIEW mv_ventas_mensuales;
```

## CREATE SEQUENCE (PostgreSQL, Oracle)

Las secuencias son generadores de números autoincrementales. En MySQL usas AUTO_INCREMENT en la columna directamente. En PostgreSQL y Oracle, las secuencias son objetos independientes que puedes usar en múltiples tablas.

Puedes controlar dónde empieza (START WITH), de cuánto en cuánto incrementa (INCREMENT BY), valores mínimos y máximos, y si debe reiniciarse al llegar al máximo (CYCLE).

`nextval('seq_folio_factura')` obtiene el siguiente valor de la secuencia. Esto es útil para generar folios de facturas, números de orden, etc.

```sql
-- PostgreSQL
CREATE SEQUENCE seq_folio_factura
    START WITH 10000
    INCREMENT BY 1
    MINVALUE 1
    MAXVALUE 999999
    CYCLE;

-- Usar la secuencia
INSERT INTO facturas (folio, id_cliente, total)
VALUES (nextval('seq_folio_factura'), 1, 5000.00);
```

## CREATE FUNCTION / PROCEDURE

Las funciones y procedimientos almacenados son código SQL que se guarda en la base de datos y se ejecuta en el servidor. Son como "programas" dentro de la base de datos.

**Funciones**: retornan un valor. Se usan dentro de expresiones SQL. En el ejemplo, `calcular_iva(1000, 16)` retorna 160.00. Son deterministas (misma entrada = misma salida).

**Procedimientos**: pueden realizar múltiples operaciones (INSERTs, UPDATEs, etc.) y no necesitan retornar un valor. En el ejemplo, `registrar_venta()` inserta una factura y retorna el ID generado.

`DELIMITER //` cambia el delimitador temporalmente para que el motor no interprete los punto y coma dentro del cuerpo del procedimiento como el fin del comando.

Las funciones y procedimientos son útiles para:
- Encapsular lógica de negocio en la base de datos
- Reutilizar código entre aplicaciones
- Mejorar seguridad (el usuario solo necesita permiso para ejecutar el procedimiento, no para acceder a las tablas directamente)

```sql
-- Función: Calcula el IVA
DELIMITER //
CREATE FUNCTION calcular_iva(monto DECIMAL(12,2), tasa DECIMAL(4,2))
RETURNS DECIMAL(12,2)
DETERMINISTIC
BEGIN
    RETURN ROUND(monto * tasa / 100, 2);
END //
DELIMITER ;

-- Usar función
SELECT calcular_iva(1000, 16);  -- 160.00

-- Procedimiento: Registrar venta
DELIMITER //
CREATE PROCEDURE registrar_venta(
    IN p_id_cliente INT,
    IN p_id_vendedor INT,
    IN p_total DECIMAL(12,2)
)
BEGIN
    INSERT INTO facturas (id_cliente, id_vendedor, total, fecha_emision)
    VALUES (p_id_cliente, p_id_vendedor, p_total, NOW());
    
    SELECT LAST_INSERT_ID() AS id_factura;
END //
DELIMITER ;
```

## 📊 Modelo Completo para un Sistema de Ventas

Este es el modelo completo de base de datos para un sistema ERP de ventas. A continuación, cada tabla se explica con su propósito y las decisiones de diseño.

### Catálogos Geográficos

Las tablas `paises`, `estados` y `municipios` forman una jerarquía geográfica. Se usan para ubicar clientes, sucursales y almacenes. La tabla `paises` usa `CHAR(3)` como clave primaria porque el código ISO 3166-1 alpha-3 es de longitud fija (3 caracteres, ej: 'MEX', 'USA'). `id_pais` en `estados` es una clave foránea que referencia a `paises`.

```sql
-- =============================================
-- ESQUEMA COMPLETO PARA ERP DE VENTAS
-- =============================================

CREATE DATABASE IF NOT EXISTS erp_ventas
    CHARACTER SET utf8mb4
    COLLATE utf8mb4_unicode_ci;

USE erp_ventas;

-- TABLAS MAESTRAS
CREATE TABLE paises (
    id_pais CHAR(3) PRIMARY KEY,  -- ISO 3166-1 alpha-3
    nombre VARCHAR(100) NOT NULL
);

CREATE TABLE estados (
    id_estado INT PRIMARY KEY AUTO_INCREMENT,
    id_pais CHAR(3) NOT NULL,
    nombre VARCHAR(100) NOT NULL,
    FOREIGN KEY (id_pais) REFERENCES paises(id_pais)
);

CREATE TABLE municipios (
    id_municipio INT PRIMARY KEY AUTO_INCREMENT,
    id_estado INT NOT NULL,
    nombre VARCHAR(150) NOT NULL,
    FOREIGN KEY (id_estado) REFERENCES estados(id_estado)
);
```

### Sucursales y Almacenes

La tabla `sucursales` representa las tiendas físicas o sucursales de la empresa. `codigo` es UNIQUE para tener un identificador corto y legible (ej: 'SUC-001'). `id_municipio` es opcional (por eso no tiene NOT NULL) — una sucursal podría no tener dirección geográfica registrada.

`almacenes` depende de `sucursales` (cada sucursal puede tener múltiples almacenes). El campo `tipo` usa ENUM para restringir los valores: principal, secundario, tránsito, devoluciones. Es más seguro que un VARCHAR libre.

```sql
CREATE TABLE sucursales (
    id_sucursal INT PRIMARY KEY AUTO_INCREMENT,
    codigo VARCHAR(10) UNIQUE NOT NULL,
    nombre VARCHAR(150) NOT NULL,
    id_municipio INT,
    direccion TEXT,
    telefono VARCHAR(20),
    email VARCHAR(150),
    activo BOOLEAN DEFAULT TRUE,
    FOREIGN KEY (id_municipio) REFERENCES municipios(id_municipio)
);

CREATE TABLE almacenes (
    id_almacen INT PRIMARY KEY AUTO_INCREMENT,
    id_sucursal INT NOT NULL,
    codigo VARCHAR(10) UNIQUE NOT NULL,
    nombre VARCHAR(100) NOT NULL,
    tipo ENUM('principal', 'secundario', 'transito', 'devoluciones') DEFAULT 'principal',
    activo BOOLEAN DEFAULT TRUE,
    FOREIGN KEY (id_sucursal) REFERENCES sucursales(id_sucursal)
);
```

### Categorías y Unidades de Medida

`categorias` tiene una auto-referencia: `id_categoria_padre` apunta a la misma tabla. Esto permite jerarquías de categorías anidadas (Electrónicos > Computadoras > Laptops). `nivel` indica la profundidad (1 = categoría raíz, 2 = subcategoría, etc.). No es estrictamente necesario (se puede calcular), pero acelera las consultas.

`unidades_medida` es un catálogo simple: piezas, kilogramos, litros, metros, cajas, etc. Se referencia desde productos.

```sql
CREATE TABLE categorias (
    id_categoria INT PRIMARY KEY AUTO_INCREMENT,
    codigo VARCHAR(20) UNIQUE NOT NULL,
    nombre VARCHAR(100) NOT NULL,
    id_categoria_padre INT,
    nivel INT DEFAULT 1,
    activo BOOLEAN DEFAULT TRUE,
    FOREIGN KEY (id_categoria_padre) REFERENCES categorias(id_categoria)
);

CREATE TABLE unidades_medida (
    id_unidad INT PRIMARY KEY AUTO_INCREMENT,
    codigo VARCHAR(10) UNIQUE NOT NULL,
    nombre VARCHAR(50) NOT NULL,
    abreviacion VARCHAR(10)
);
```

### Productos

La tabla `productos` es el corazón del catálogo. `sku` (Stock Keeping Unit) es el código interno del producto, único y obligatorio. `codigo_barras` también es único pero opcional (puede ser NULL).

Los precios usan DECIMAL, nunca FLOAT. `precio_compra` tiene 4 decimales porque los costos suelen tener más precisión que los precios de venta. `iva` e `ieps` tienen valor por defecto para el caso más común. `controla_stock` es un booleano: algunos productos (como servicios) no requieren control de inventario.

La clave foránea a `categorias` tiene `id_categoria INT NOT NULL` — todo producto debe tener una categoría.

```sql
CREATE TABLE productos (
    id_producto INT PRIMARY KEY AUTO_INCREMENT,
    sku VARCHAR(30) UNIQUE NOT NULL,
    codigo_barras VARCHAR(50) UNIQUE,
    nombre_comercial VARCHAR(200) NOT NULL,
    descripcion TEXT,
    id_categoria INT NOT NULL,
    id_unidad INT NOT NULL,
    precio_compra DECIMAL(12,4),
    precio_venta DECIMAL(12,2) NOT NULL,
    iva DECIMAL(4,2) DEFAULT 16.00,
    ieps DECIMAL(4,2) DEFAULT 0,
    peso DECIMAL(10,4),
    volumen DECIMAL(10,4),
    activo BOOLEAN DEFAULT TRUE,
    controla_stock BOOLEAN DEFAULT TRUE,
    fecha_creacion DATETIME DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (id_categoria) REFERENCES categorias(id_categoria),
    FOREIGN KEY (id_unidad) REFERENCES unidades_medida(id_unidad)
);
```

### Stock por Almacén

`productos_stock` usa una clave primaria COMPUESTA: `(id_producto, id_almacen)`. Esto significa que la combinación producto+almacén es única. Un mismo producto puede tener diferente stock en diferentes almacenes.

Los campos `stock_minimo`, `stock_maximo` y `punto_reorden` se usan para generar alertas de inventario: cuando el stock actual cae por debajo del punto de reorden, hay que comprar más. `costo_promedio` se actualiza con cada compra (costo promedio ponderado).

```sql
CREATE TABLE productos_stock (
    id_producto INT NOT NULL,
    id_almacen INT NOT NULL,
    stock_actual DECIMAL(12,4) DEFAULT 0,
    stock_minimo DECIMAL(12,4) DEFAULT 0,
    stock_maximo DECIMAL(12,4) DEFAULT 0,
    punto_reorden DECIMAL(12,4) DEFAULT 0,
    costo_promedio DECIMAL(12,4) DEFAULT 0,
    ultima_actualizacion DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    PRIMARY KEY (id_producto, id_almacen),
    FOREIGN KEY (id_producto) REFERENCES productos(id_producto),
    FOREIGN KEY (id_almacen) REFERENCES almacenes(id_almacen)
);
```

### Clientes

`clientes` es una tabla maestra fundamental. `tipo_persona` distingue entre persona física y moral (empresa). `rfc` es único y obligatorio por requisitos fiscales. `razon_social` solo aplica para personas morales.

Los campos de crédito (`credito_limite`, `saldo_actual`, `dias_credito`) gestionan las cuentas por cobrar. `lista_precios` define qué precios aplican a este cliente (público, mayoreo, distribuidor) — esto permite estrategias de precios diferenciadas.

```sql
CREATE TABLE clientes (
    id_cliente INT PRIMARY KEY AUTO_INCREMENT,
    codigo VARCHAR(20) UNIQUE NOT NULL,
    tipo_persona ENUM('fisica', 'moral') NOT NULL,
    nombre VARCHAR(150) NOT NULL,
    rfc VARCHAR(13) UNIQUE NOT NULL,
    razon_social VARCHAR(200),
    email VARCHAR(150),
    telefono VARCHAR(20),
    id_municipio INT,
    direccion TEXT,
    fecha_registro DATETIME DEFAULT CURRENT_TIMESTAMP,
    credito_limite DECIMAL(12,2) DEFAULT 0,
    saldo_actual DECIMAL(12,2) DEFAULT 0,
    dias_credito INT DEFAULT 0,
    lista_precios ENUM('publico', 'mayoreo', 'distribuidor') DEFAULT 'publico',
    activo BOOLEAN DEFAULT TRUE,
    FOREIGN KEY (id_municipio) REFERENCES municipios(id_municipio)
);
```

### Vendedores

`vendedores` tiene `comision_porcentaje` para saber qué porcentaje de comisión gana cada vendedor. `meta_mensual` es el objetivo de ventas. La relación con `sucursales` permite saber en qué sucursal trabaja cada vendedor.

```sql
CREATE TABLE vendedores (
    id_vendedor INT PRIMARY KEY AUTO_INCREMENT,
    codigo VARCHAR(20) UNIQUE NOT NULL,
    nombre VARCHAR(150) NOT NULL,
    email VARCHAR(150),
    telefono VARCHAR(20),
    id_sucursal INT,
    comision_porcentaje DECIMAL(5,2) DEFAULT 5.00,
    meta_mensual DECIMAL(12,2),
    activo BOOLEAN DEFAULT TRUE,
    FOREIGN KEY (id_sucursal) REFERENCES sucursales(id_sucursal)
);
```

### Formas y Métodos de Pago

`formas_pago` y `metodos_pago_sat` son tablas pequeñas de catálogo. La primera es de uso interno; la segunda corresponde al catálogo oficial del SAT (Servicio de Administración Tributaria en México) para facturación electrónica CFDI. Se mantienen separadas porque los códigos del SAT pueden cambiar y no queremos que eso afecte la operación interna.

```sql
CREATE TABLE formas_pago (
    codigo VARCHAR(20) PRIMARY KEY,
    descripcion VARCHAR(100) NOT NULL,
    activo BOOLEAN DEFAULT TRUE
);

CREATE TABLE metodos_pago_sat (
    c_forma_pago VARCHAR(10) PRIMARY KEY,
    descripcion VARCHAR(200) NOT NULL
);
```

### Facturas (Tabla Transaccional)

`facturas` es la tabla transaccional más importante. `folio` es único y se genera secuencialmente (ej: 'F-2024-5000'). `uuid` almacena el UUID del CFDI (Comprobante Fiscal Digital por Internet) — es opcional porque no todas las facturas se timbran.

`tipo_comprobante` usa ENUM con los valores del SAT: I (Ingreso), E (Egreso), N (Nómina), P (Pago). `estado` gestiona el ciclo de vida: activa, cancelada, devuelta parcial, devuelta total, pendiente.

Los totales (subtotal, descuento, IVA, IEPS, total) se almacenan explícitamente (no se calculan en cada consulta) porque una vez emitida la factura, esos valores NO deben cambiar aunque los precios de los productos cambien en el catálogo.

Las claves foráneas conectan con clientes, vendedores y sucursales. Nota que `id_vendedor` es opcional (no todas las ventas tienen vendedor asignado, como en comercio electrónico).

```sql
-- TABLAS TRANSACCIONALES
CREATE TABLE facturas (
    id_factura INT PRIMARY KEY AUTO_INCREMENT,
    folio VARCHAR(30) UNIQUE NOT NULL,
    uuid VARCHAR(36) UNIQUE,  -- CFDI UUID
    serie VARCHAR(10),
    id_cliente INT NOT NULL,
    id_vendedor INT,
    id_sucursal INT,
    fecha_emision DATETIME DEFAULT CURRENT_TIMESTAMP,
    fecha_pago DATETIME,
    tipo_comprobante ENUM('I', 'E', 'N', 'P') DEFAULT 'I',  -- Ingreso, Egreso, Nómina, Pago
    subtotal DECIMAL(12,2),
    descuento DECIMAL(12,2) DEFAULT 0,
    iva DECIMAL(12,2),
    ieps DECIMAL(12,2) DEFAULT 0,
    total DECIMAL(12,2),
    forma_pago VARCHAR(10),
    metodo_pago VARCHAR(20),
    estado ENUM('activa', 'cancelada', 'devuelta_parcial', 'devuelta_total', 'pendiente') DEFAULT 'activa',
    notas TEXT,
    fecha_creacion DATETIME DEFAULT CURRENT_TIMESTAMP,
    usuario_creacion VARCHAR(100),
    FOREIGN KEY (id_cliente) REFERENCES clientes(id_cliente),
    FOREIGN KEY (id_vendedor) REFERENCES vendedores(id_vendedor),
    FOREIGN KEY (id_sucursal) REFERENCES sucursales(id_sucursal)
);
```

### Detalle de Factura

`facturas_detalle` tiene una relación de MUCHOS a UNO con `facturas`. Una factura tiene muchas líneas de detalle. La clave foránea `id_factura` tiene `ON DELETE CASCADE`: si la factura se elimina, todas sus líneas de detalle se eliminan automáticamente.

Cada línea registra cantidad (con 4 decimales para admitir fracciones), precio_unitario (el precio al momento de la venta, no el precio actual del producto), descuento (porcentaje o monto), y los totales por línea.

`id_producto` **no** tiene ON DELETE CASCADE (el RESTRICT implícito evita eliminar productos que ya se vendieron). Esto protege la integridad histórica.

```sql
CREATE TABLE facturas_detalle (
    id_detalle INT PRIMARY KEY AUTO_INCREMENT,
    id_factura INT NOT NULL,
    id_producto INT NOT NULL,
    cantidad DECIMAL(12,4) NOT NULL,
    precio_unitario DECIMAL(12,2) NOT NULL,
    descuento DECIMAL(12,2) DEFAULT 0,
    subtotal DECIMAL(12,2),
    iva DECIMAL(12,2),
    ieps DECIMAL(12,2) DEFAULT 0,
    total DECIMAL(12,2),
    FOREIGN KEY (id_factura) REFERENCES facturas(id_factura) ON DELETE CASCADE,
    FOREIGN KEY (id_producto) REFERENCES productos(id_producto)
);
```

### Pagos de Factura

`facturas_pagos` permite pagos parciales. Una factura puede tener múltiples pagos (por ejemplo, un anticipo y el saldo al entregar). Cada pago registra la fecha, el monto, la forma de pago y una referencia (número de transferencia, cheque, etc.).

La clave foránea a `facturas` no tiene ON DELETE CASCADE porque no quieres perder el historial de pagos si algo pasa con la factura (depende de la política del negocio).

```sql
CREATE TABLE facturas_pagos (
    id_pago INT PRIMARY KEY AUTO_INCREMENT,
    id_factura INT NOT NULL,
    fecha_pago DATETIME DEFAULT CURRENT_TIMESTAMP,
    monto DECIMAL(12,2) NOT NULL,
    forma_pago VARCHAR(10),
    referencia VARCHAR(100),
    FOREIGN KEY (id_factura) REFERENCES facturas(id_factura)
);
```

### Índices para Rendimiento

Los índices son cruciales para el rendimiento de las consultas en este modelo. Cada índice está diseñado para acelerar consultas específicas:

- `idx_facturas_fecha`: para reportes por rango de fechas.
- `idx_facturas_cliente`: para buscar todas las facturas de un cliente.
- `idx_facturas_estado`: para filtrar facturas activas/canceladas.
- `idx_facturas_vendedor`: para comisiones y reportes por vendedor.
- `idx_facturas_detalle_factura` y `idx_facturas_detalle_producto`: para JOINs rápidos entre detalle y facturas/productos.
- `idx_productos_categoria`: para filtrar productos por categoría.
- `idx_productos_sku` y `idx_clientes_rfc` y `idx_clientes_codigo`: para búsquedas exactas por estos códigos únicos.

```sql
-- ÍNDICES PARA RENDIMIENTO
CREATE INDEX idx_facturas_fecha ON facturas(fecha_emision);
CREATE INDEX idx_facturas_cliente ON facturas(id_cliente);
CREATE INDEX idx_facturas_estado ON facturas(estado);
CREATE INDEX idx_facturas_vendedor ON facturas(id_vendedor);
CREATE INDEX idx_facturas_detalle_factura ON facturas_detalle(id_factura);
CREATE INDEX idx_facturas_detalle_producto ON facturas_detalle(id_producto);
CREATE INDEX idx_productos_categoria ON productos(id_categoria);
CREATE INDEX idx_productos_sku ON productos(sku);
CREATE INDEX idx_clientes_rfc ON clientes(rfc);
CREATE INDEX idx_clientes_codigo ON clientes(codigo);
```

## Scripts para Generar Datos de Prueba

Una vez creadas las tablas, necesitas datos de prueba para empezar a trabajar. Aquí insertamos los catálogos más básicos: países y formas de pago. Sin estos datos mínimos, no podrías crear clientes (necesitan municipio → estado → país) ni facturas (necesitan forma de pago).

```sql
-- Poblar catálogos básicos
INSERT INTO paises (id_pais, nombre) VALUES ('MEX', 'México'), ('USA', 'Estados Unidos');
INSERT INTO formas_pago (codigo, descripcion) VALUES
    ('EFE', 'Efectivo'), ('TDC', 'Tarjeta Crédito'), ('TDD', 'Tarjeta Débito'),
    ('TRA', 'Transferencia'), ('CHE', 'Cheque');
```

---

## Anterior: [06 Insert Update Delete](../02-fundamentos/06-insert-update-delete.md)
## Siguiente: [08 Indices Y Optimizacion](../02-fundamentos/08-indices-y-optimizacion.md)

[Volver al índice](../README.md)
