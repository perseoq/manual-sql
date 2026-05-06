# 2.6 INSERT, UPDATE, DELETE

Bienvenido al capítulo donde tus tablas COBRAN VIDA. Hasta ahora has aprendido a leer datos con SELECT, pero una base de datos sin datos es como una biblioteca vacía. Con INSERT, UPDATE y DELETE puedes agregar, modificar y eliminar registros. Estos comandos pertenecen al grupo DML (Data Manipulation Language) — el lenguaje que manipula los datos dentro de las tablas.

**IMPORTANTE**: A diferencia de SELECT, estos comandos CAMBIAN la base de datos para siempre. Un UPDATE sin WHERE, un DELETE sin WHERE, o un INSERT incorrecto pueden causar estragos. Por eso siempre debes usar transacciones (START TRANSACTION / COMMIT / ROLLBACK) cuando hagas operaciones que modifiquen datos importantes.

## INSERT - Agregar Datos

### INSERT Básico

El comando INSERT agrega nuevas filas a una tabla. Piensa en ello como llenar un formulario: escribes los valores en cada campo y los guardas como un nuevo registro.

La sintaxis es: `INSERT INTO tabla (columnas) VALUES (valores)`. Las columnas se listan entre paréntesis después del nombre de la tabla, y los valores correspondientes van en el mismo orden entre paréntesis después de VALUES. La estructura debe coincidir: primera columna con primer valor, segunda con segundo, etc.

En el ejemplo, estamos agregando un nuevo cliente a la tabla `clientes`. Proporcionamos nombre, email, teléfono y RFC. La columna `id_cliente` no se incluye porque típicamente es AUTO_INCREMENT (se genera automáticamente). Las columnas que no se listan recibirán su valor DEFAULT o NULL si no se especifica.

```sql
INSERT INTO tabla (columna1, columna2, columna3)
VALUES (valor1, valor2, valor3);

-- Ejemplo
INSERT INTO clientes (nombre, email, telefono, rfc)
VALUES ('Juan Pérez', 'juan@email.com', '5512345678', 'PEPJ850101ABC');
```

### INSERT Múltiples Filas

Si necesitas insertar varios registros a la vez, SQL te permite hacerlo en una sola instrucción. En lugar de escribir 4 INSERTs separados, escribes una sola instrucción con múltiples listas de VALUES separadas por comas. Esto es mucho más eficiente: la base de datos procesa una sola transacción en lugar de cuatro.

Imagina que estás cargando el catálogo de productos inicial de tu tienda. Con INSERT múltiple, agregas todos los productos de una sola vez. Es más rápido, genera menos tráfico en la red, y todo se registra como una sola operación.

Cada fila entre paréntesis representa un nuevo producto. Asegúrate de que el orden de los valores coincida con el orden de las columnas que listaste.

```sql
INSERT INTO productos (codigo_barras, nombre_producto, id_categoria, precio_venta, stock_actual) VALUES
    ('7501234567890', 'Laptop HP EliteBook', 1, 28000.00, 10),
    ('7501234567891', 'Monitor 27" 4K', 1, 8500.00, 25),
    ('7501234567892', 'Teclado Mecánico', 1, 1200.00, 50),
    ('7501234567893', 'Mouse Inalámbrico', 1, 450.00, 100);
```

### INSERT con SELECT

Esta es una de las variantes más poderosas de INSERT. En lugar de escribir valores fijos, puedes tomar los resultados de un SELECT e insertarlos directamente en otra tabla. Es como "copiar y pegar" datos entre tablas.

El primer ejemplo crea una copia de seguridad de los clientes activos. El SELECT obtiene los datos de los clientes activos y los inserta en `clientes_backup`. Las columnas del SELECT deben coincidir en orden y tipo con las columnas listadas en el INSERT. Es una forma rápida de respaldar datos antes de hacer cambios masivos.

El segundo ejemplo es más interesante: crea una tabla promocional con el 85% del precio original para productos de la categoría 3 que tengan buen stock. Esto es exactamente lo que harías para preparar una campaña de ventas: seleccionas productos que cumplen ciertos criterios y los insertas con precios modificados en una tabla de promociones.

```sql
-- Crear una copia de seguridad de clientes activos
INSERT INTO clientes_backup (id_cliente, nombre, email, rfc, fecha_registro)
SELECT id_cliente, nombre, email, rfc, fecha_registro
FROM clientes
WHERE activo = TRUE;

-- Insertar productos de una categoría a otra
INSERT INTO productos_promocion (id_producto, nombre_producto, precio_promocion)
SELECT id_producto, nombre_producto, precio_venta * 0.85
FROM productos
WHERE id_categoria = 3 AND stock_actual > 20;
```

### INSERT IGNORE / INSERT OR REPLACE

¿Qué pasa cuando intentas insertar un registro que ya existe (por ejemplo, con un ID duplicado o un RFC repetido)? Normalmente obtienes un error. Pero a veces quieres decir "si ya existe, ignóralo" o "si ya existe, actualízalo". Para eso existen estas variantes.

**INSERT IGNORE** (MySQL): Intenta insertar, pero si hay un conflicto (duplicado en PRIMARY KEY o UNIQUE), simplemente ignora esa fila y sigue con las demás. No genera error.

**ON CONFLICT DO NOTHING** (PostgreSQL): Es el equivalente a INSERT IGNORE.

**ON DUPLICATE KEY UPDATE** (MySQL): Esto es un "UPSERT" (UPDATE + INSERT). Intenta insertar; si el registro ya existe, en lugar de fallar, actualiza las columnas que especifiques. En el ejemplo, si el producto con id=1 ya existe, actualiza su precio y suma al stock.

**ON CONFLICT DO UPDATE** (PostgreSQL): Es el equivalente con la palabra clave EXCLUDED para referirse a los valores que se intentaron insertar.

El UPSERT es muy útil para sincronizar datos entre sistemas o para procesos de carga donde no sabes si los registros ya existen.

```sql
-- MySQL: Ignorar si ya existe (por duplicado de clave única)
INSERT IGNORE INTO clientes (id_cliente, nombre, email)
VALUES (100, 'María López', 'maria@email.com');

-- PostgreSQL: ON CONFLICT DO NOTHING
INSERT INTO clientes (id_cliente, nombre, email)
VALUES (100, 'María López', 'maria@email.com')
ON CONFLICT (id_cliente) DO NOTHING;

-- MySQL: Actualizar si existe (UPSERT)
INSERT INTO productos (id_producto, nombre_producto, precio_venta, stock_actual)
VALUES (1, 'Laptop HP ProBook', 26000.00, 12)
ON DUPLICATE KEY UPDATE
    precio_venta = VALUES(precio_venta),
    stock_actual = stock_actual + VALUES(stock_actual);

-- PostgreSQL: UPSERT
INSERT INTO productos (id_producto, nombre_producto, precio_venta, stock_actual)
VALUES (1, 'Laptop HP ProBook', 26000.00, 12)
ON CONFLICT (id_producto) DO UPDATE SET
    precio_venta = EXCLUDED.precio_venta,
    stock_actual = productos.stock_actual + EXCLUDED.stock_actual;
```

## UPDATE - Modificar Datos

### UPDATE Básico

UPDATE modifica registros existentes. La estructura es: especificas la tabla, asignas nuevos valores a las columnas con SET, y filtras qué filas modificar con WHERE.

**REPITE CONMIGO: SIEMPRE USA WHERE EN UPDATE**. Si ejecutas `UPDATE clientes SET activo = FALSE` sin WHERE, TODOS los clientes se volverán inactivos. No hay confirmación, no hay "¿estás seguro?". Es instantáneo y permanente (sin transacción de por medio).

El primer ejemplo es seguro: actualiza el teléfono de un cliente específico (id_cliente = 100). Solo UNA fila se modifica.

El segundo ejemplo es la ADVERTENCIA: `UPDATE clientes SET activo = FALSE` sin WHERE. Esto pone a TODOS los clientes como inactivos. A veces esto es lo que quieres (por ejemplo, desactivar todos los usuarios temporalmente), pero si no es tu intención, acabas de causar un desastre.

```sql
UPDATE clientes
SET telefono = '5522223333'
WHERE id_cliente = 100;

-- ⚠️ SIN WHERE actualiza TODOS los registros
UPDATE clientes SET activo = FALSE;  -- ¡TODOS inactivos!
```

### UPDATE con JOIN

A veces necesitas actualizar una tabla basándote en valores de OTRA tabla. Por ejemplo: "aplica un 5% de descuento a las facturas activas de clientes VIP". Necesitas información de clientes (tipo_cliente) para modificar facturas.

MySQL, PostgreSQL y SQL Server tienen sintaxis ligeramente diferente para esto, pero el concepto es el mismo: unes dos tablas y actualizas la primera basándote en condiciones de ambas.

En MySQL, escribes `UPDATE facturas f JOIN clientes c ON ... SET f.total = ... WHERE c.tipo_cliente = 'VIP'`. En PostgreSQL, usas `FROM clientes c WHERE f.id_cliente = c.id_cliente AND c.tipo_cliente = 'VIP'`. En SQL Server, es similar a MySQL pero con el alias después de UPDATE.

Elige la sintaxis de tu motor, pero el patrón es universal: JOIN para obtener datos relacionados y UPDATE para modificarlos.

```sql
-- MySQL
UPDATE facturas f
JOIN clientes c ON f.id_cliente = c.id_cliente
SET f.total = f.total * 0.95
WHERE c.tipo_cliente = 'VIP' AND f.estado = 'activa';

-- PostgreSQL
UPDATE facturas f
SET total = f.total * 0.95
FROM clientes c
WHERE f.id_cliente = c.id_cliente
  AND c.tipo_cliente = 'VIP'
  AND f.estado = 'activa';

-- SQL Server
UPDATE f
SET f.total = f.total * 0.95
FROM facturas f
JOIN clientes c ON f.id_cliente = c.id_cliente
WHERE c.tipo_cliente = 'VIP' AND f.estado = 'activa';
```

### UPDATE con Subconsulta

Las subconsultas también funcionan en UPDATE. Son útiles cuando necesitas calcular un valor basado en datos agregados de la misma u otra tabla.

El primer ejemplo aumenta 10% el precio de productos que están por debajo del precio promedio. La subconsulta `(SELECT AVG(precio_venta) FROM productos)` calcula el promedio GENERAL, y el WHERE solo selecciona los que están debajo de ese promedio. Es una forma de ajustar precios para que ningún producto esté muy por debajo del promedio del mercado interno.

El segundo ejemplo es más avanzado: actualiza el saldo actual de cada cliente basándose en la suma de sus facturas impagas (método de pago = crédito y estado activa). La subconsulta está correlacionada: `WHERE f.id_cliente = c.id_cliente` conecta la subconsulta con el cliente que se está actualizando. COALESCE(SUM(total), 0) asegura que si el cliente no tiene facturas, el saldo sea 0 en lugar de NULL.

```sql
-- Aumentar precio de productos que están por debajo del promedio
UPDATE productos p
SET precio_venta = precio_venta * 1.10
WHERE precio_venta < (SELECT AVG(precio_venta) FROM productos);

-- Actualizar saldo de clientes basado en facturas impagas
UPDATE clientes c
SET saldo_actual = (
    SELECT COALESCE(SUM(total), 0)
    FROM facturas f
    WHERE f.id_cliente = c.id_cliente
      AND f.estado = 'activa'
      AND f.metodo_pago = 'credito'
);
```

### UPDATE con CASE

¿Necesitas actualizar diferentes filas con diferentes valores en una sola instrucción? CASE dentro de SET es la solución. Es como un "si-entonces" para cada fila.

En este ejemplo, queremos aumentar precios, pero con porcentajes diferentes según la categoría: 15% para electrónicos (categoría 1), 10% para ropa (categoría 2), 5% para hogar (categoría 3), y 8% para todo lo demás (ELSE). Sin CASE, tendrías que ejecutar 4 UPDATEs separados. Con CASE, todo se hace en una sola instrucción atómica.

La sintaxis es: `SET columna = CASE WHEN condición THEN valor WHEN condición THEN valor ELSE valor END`. Cada fila evalúa su condición correspondiente y aplica el primer WHEN que cumpla.

```sql
-- Actualización masiva con diferentes criterios
UPDATE productos
SET precio_venta = CASE
    WHEN id_categoria = 1 THEN precio_venta * 1.15  -- 15% electrónicos
    WHEN id_categoria = 2 THEN precio_venta * 1.10  -- 10% ropa
    WHEN id_categoria = 3 THEN precio_venta * 1.05  -- 5% hogar
    ELSE precio_venta * 1.08                         -- 8% otros
END;
```

## DELETE - Eliminar Datos

### DELETE Básico

DELETE elimina filas de una tabla. Y al igual que UPDATE, **NUNCA** ejecutes DELETE sin WHERE a menos que estés 100% seguro de querer vaciar la tabla.

El primer ejemplo elimina un cliente específico. Simple y directo.

El segundo ejemplo es la ADVERTENCIA: `DELETE FROM clientes` sin WHERE elimina TODOS los clientes. Todas las filas. Para siempre (sin transacción). No hay papelera de reciclaje en SQL.

```sql
-- Eliminar un cliente específico
DELETE FROM clientes WHERE id_cliente = 999;

-- ⚠️ SIN WHERE elimina TODOS los registros
DELETE FROM clientes;  -- ¡Adiós datos!
```

### DELETE con JOIN

A veces necesitas eliminar filas de una tabla basándote en condiciones de otra tabla. Por ejemplo: "elimina todas las facturas de clientes que están inactivos". Necesitas unir facturas con clientes para saber qué facturas pertenecen a clientes inactivos.

En MySQL, usas `DELETE f FROM facturas f JOIN clientes c ON ...`. La clave es poner el alias de la tabla de la que quieres eliminar (f) después de DELETE. En PostgreSQL, usas `DELETE FROM facturas f USING clientes c WHERE ...`. USING en PostgreSQL es el equivalente al JOIN en este contexto.

```sql
-- MySQL: Eliminar facturas de clientes inactivos
DELETE f
FROM facturas f
JOIN clientes c ON f.id_cliente = c.id_cliente
WHERE c.activo = FALSE;

-- PostgreSQL
DELETE FROM facturas f
USING clientes c
WHERE f.id_cliente = c.id_cliente
  AND c.activo = FALSE;
```

### DELETE con Subconsulta

Las subconsultas en DELETE te permiten eliminar basándote en condiciones agregadas. El primer ejemplo elimina productos que nunca se han vendido (no aparecen en facturas_detalle). Esto limpia tu catálogo de productos obsoletos o que nunca se comercializaron.

El segundo ejemplo elimina movimientos de inventario de productos que ahora están inactivos. Es una limpieza típica de mantenimiento.

**NOTA**: En MySQL, no puedes usar la misma tabla en la subconsulta que estás eliminando directamente. Es decir, no puedes hacer `DELETE FROM productos WHERE id_producto IN (SELECT id_producto FROM productos WHERE ...)`. MySQL lo prohibe. Pero puedes rodearlo con un nivel adicional de subconsulta o usar JOIN.

```sql
-- Eliminar productos que nunca se han vendido
DELETE FROM productos
WHERE id_producto NOT IN (
    SELECT DISTINCT id_producto FROM facturas_detalle
);

-- Eliminar movimientos de inventario de productos inactivos
DELETE FROM inventario_movimientos
WHERE id_producto IN (
    SELECT id_producto FROM productos WHERE activo = FALSE
);
```

## TRUNCATE - Vaciar Tabla Rápido

TRUNCATE es como DELETE pero en esteroides. Elimina TODAS las filas de una tabla de manera mucho más rápida. Pero hay diferencias importantes.

DELETE elimina fila por fila, registra cada eliminación en el log de transacciones, dispara triggers, y no resetea el contador AUTO_INCREMENT.

TRUNCATE elimina la tabla y la vuelve a crear vacía. Es más rápido porque no registra cada fila, no dispara triggers, y resetea el AUTO_INCREMENT. PERO: en muchos motores, TRUNCATE no se puede revertir con ROLLBACK (especialmente en MySQL con tablas InnoDB grandes, aunque técnicamente sí es transaccional en InnoDB).

¿Cuándo usar cada uno? Usa DELETE cuando necesites filtrar (WHERE), cuando quieras mantener el AUTO_INCREMENT, o cuando necesites transaccionalidad. Usa TRUNCATE cuando quieras vaciar completamente una tabla temporal o de staging de la manera más rápida posible.

```sql
-- DELETE: lento, transaccional, dispara triggers
DELETE FROM facturas_temporales;

-- TRUNCATE: rápido, no transaccional (en MySQL), resetea AUTO_INCREMENT
TRUNCATE TABLE facturas_temporales;
```

| Característica | DELETE | TRUNCATE |
|----------------|--------|----------|
| Velocidad | Lento (fila por fila) | Muy rápido |
| Transaccional | Sí (ROLLBACK posible) | Depende del motor |
| Triggers | Dispara triggers | No dispara |
| AUTO_INCREMENT | No resetea | Resetea |
| WHERE | Sí | No |
| Espacio en disco | No libera | Libera (MySQL) |

## MERGE (UPSERT)

MERGE es una instrucción avanzada que combina INSERT, UPDATE y DELETE en una sola operación. Se usa cuando tienes una tabla "fuente" con datos nuevos y una tabla "destino" que debe sincronizarse. Para cada fila de la fuente, decides si insertarla (no existe en destino), actualizarla (ya existe), o incluso eliminarla.

En el ejemplo, tenemos un inventario_objetivo (destino) y un inventario_fuente (origen, probablemente de un proveedor). Cuando un producto ya existe en el destino (MATCHED), actualizamos su stock sumando la cantidad recibida. Cuando NO existe (NOT MATCHED), insertamos un nuevo registro con el stock inicial y un stock mínimo de 10.

MERGE está disponible en PostgreSQL, SQL Server, y Oracle. MySQL no lo soporta nativamente (usa ON DUPLICATE KEY UPDATE en su lugar).

```sql
-- PostgreSQL / SQL Server
MERGE INTO inventario_objetivo AS target
USING inventario_fuente AS source
ON target.id_producto = source.id_producto
WHEN MATCHED THEN
    UPDATE SET 
        stock_actual = target.stock_actual + source.cantidad_recibida,
        fecha_ultima_actualizacion = CURRENT_TIMESTAMP
WHEN NOT MATCHED THEN
    INSERT (id_producto, stock_actual, stock_minimo)
    VALUES (source.id_producto, source.cantidad_recibida, 10);
```

## 📊 Ejemplos del Negocio

### Registro de Venta Completo

Aquí tienes el ejemplo más importante de este capítulo: el registro completo de una venta en un sistema ERP. Esto NO es una sola instrucción, sino una TRANSACCIÓN que involucra múltiples pasos. Si falla alguno, TODO debe revertirse para no dejar datos inconsistentes.

La transacción comienza con `START TRANSACTION`. Luego:

1. **Insertar la factura**: Creamos el encabezado de la factura con folio, cliente, vendedor, subtotal, IVA, total y método de pago. Usamos `LAST_INSERT_ID()` para obtener el ID de la factura recién creada y guardarlo en `@id_factura`.

2. **Insertar el detalle**: Agregamos las líneas de productos vendidos. Cada línea especifica qué producto, cuántas unidades, precio unitario, subtotal, IVA y total. Nota que el detalle suma 3 productos con un subtotal de $7,000 y un total de $8,120 (con IVA incluido), que es MENOS que el total de la factura ($9,860) porque incluye un cuarto concepto o redondeo.

3. **Actualizar inventario**: Primero insertamos movimientos de inventario (salidas) para cada producto. Luego actualizamos el stock actual en la tabla productos, restando las cantidades vendidas usando una subconsulta que suma lo vendido.

Finalmente, `COMMIT` hace permanentes todos los cambios. Si algo sale mal en cualquier paso, ejecutarías `ROLLBACK` y todo volvería al estado anterior.

Este patrón es la base de cualquier sistema de ventas, facturación o ERP. Sin transacciones, podrías terminar con una factura sin detalle, o con stock descontado sin factura.

```sql
-- Transacción completa de una venta
START TRANSACTION;

-- 1. Insertar la factura
INSERT INTO facturas (folio, id_cliente, id_vendedor, subtotal, iva, total, metodo_pago)
VALUES ('F-2024-5000', 1, 1, 8500.00, 1360.00, 9860.00, 'tarjeta_credito');

SET @id_factura = LAST_INSERT_ID();

-- 2. Insertar detalle de productos
INSERT INTO facturas_detalle (id_factura, id_producto, cantidad, precio_unitario, subtotal, iva, total) VALUES
    (@id_factura, 1, 2, 2500.00, 5000.00, 800.00, 5800.00),
    (@id_factura, 2, 5, 350.00, 1750.00, 280.00, 2030.00),
    (@id_factura, 3, 1, 250.00, 250.00, 40.00, 290.00);

-- 3. Actualizar inventario
INSERT INTO inventario_movimientos (id_producto, tipo_movimiento, cantidad, costo_unitario, referencia_tipo, referencia_id) VALUES
    (1, 'salida_venta', -2, 1800.00, 'factura', @id_factura),
    (2, 'salida_venta', -5, 200.00, 'factura', @id_factura),
    (3, 'salida_venta', -1, 120.00, 'factura', @id_factura);

UPDATE productos p
JOIN (
    SELECT id_producto, SUM(cantidad) AS total_vendido
    FROM facturas_detalle WHERE id_factura = @id_factura
    GROUP BY id_producto
) v ON p.id_producto = v.id_producto
SET p.stock_actual = p.stock_actual - v.total_vendido;

COMMIT;
```

### Actualización Masiva de Precios

Este es un escenario típico de negocio: el gerente de producto decide aumentar 8% los precios de ciertas categorías, pero solo para productos que tengan suficiente stock (más de 5 unidades) y que estén activos.

La instrucción une productos con categorías para filtrar por nombre de categoría ('Electrónicos', 'Cómputo'), verifica el stock y el estado activo, y actualiza el precio redondeado a 2 decimales junto con la fecha de actualización.

**NOTA**: Antes de ejecutar un UPDATE masivo como este, SIEMPRE ejecuta primero un SELECT con el mismo WHERE para verificar qué registros se van a afectar. Es una práctica de seguridad básica.

```sql
-- Aumento general del 8% a productos de la categoría "Electrónicos"
-- Pero solo si tienen stock suficiente
UPDATE productos p
JOIN categorias c ON p.id_categoria = c.id_categoria
SET p.precio_venta = ROUND(p.precio_venta * 1.08, 2),
    p.fecha_actualizacion = CURRENT_TIMESTAMP
WHERE c.nombre IN ('Electrónicos', 'Cómputo')
  AND p.stock_actual > 5
  AND p.activo = TRUE;
```

### Cierre de Mes

El cierre mensual es un proceso crítico en cualquier negocio. Aquí se muestran dos pasos típicos:

1. **Respaldar datos históricos**: Insertar en `historico_ventas_mensuales` un resumen de las ventas del mes que se cierra. Esto preserva la información antes de eliminarla de la tabla transaccional.

2. **Limpiar datos temporales**: Eliminar las facturas del mes que ya se respaldaron. Esto mantiene la tabla de facturas manejable y mejora el rendimiento.

**⚠️ MUY IMPORTANTE**: Solo eliminas los datos fuente DESPUÉS de haber verificado que el respaldo se insertó correctamente. Idealmente, todo esto debería ir dentro de una transacción.

```sql
-- 1. Crear tabla de histórico
INSERT INTO historico_ventas_mensuales 
    (año, mes, id_cliente, total_compras, cantidad_compras)
SELECT 
    YEAR(fecha_emision),
    MONTH(fecha_emision),
    id_cliente,
    SUM(total),
    COUNT(*)
FROM facturas
WHERE estado = 'activa'
  AND YEAR(fecha_emision) = 2024
  AND MONTH(fecha_emision) = 1
GROUP BY YEAR(fecha_emision), MONTH(fecha_emision), id_cliente;

-- 2. Eliminar facturas temporales del mes
DELETE FROM facturas WHERE YEAR(fecha_emision) = 2024 AND MONTH(fecha_emision) = 1;
-- ⚠️ Solo si ya se respaldaron
```

## Buenas Prácticas

Aquí tienes las reglas de oro para trabajar con DML:

- **SIEMPRE usa WHERE en UPDATE y DELETE** a menos que quieras vaciar toda la tabla. Es el error número 1 de los desarrolladores SQL.
- **Verifica con SELECT antes de UPDATE/DELETE masivos**: primero ejecuta un SELECT con el mismo WHERE para ver cuántas filas y cuáles se van a afectar.
- **Usa transacciones** (START TRANSACTION / COMMIT / ROLLBACK) para operaciones que afectan múltiples tablas. Si algo falla, puedes deshacer todo.
- **Haz backup** antes de modificaciones masivas. Un respaldo te salva de desastres.
- **Limita permisos**: no todos los usuarios de la base de datos necesitan permiso para DELETE o UPDATE sin WHERE. Usa los privilegios de base de datos para restringir.

El ejemplo muestra el flujo correcto antes de un UPDATE masivo: primero verificas con SELECT qué registros se van a ver afectados y cómo quedarán después del cambio. Solo cuando estás seguro, ejecutas el UPDATE.

```sql
-- ANTES de un UPDATE masivo, verificar con SELECT
-- Paso 1: Ver qué registros se afectarán
SELECT id_producto, nombre_producto, precio_venta, precio_venta * 1.10 AS nuevo_precio
FROM productos
WHERE id_categoria = 1 AND precio_venta < 1000;

-- Paso 2: Si todo está bien, ejecutar UPDATE
UPDATE productos
SET precio_venta = precio_venta * 1.10
WHERE id_categoria = 1 AND precio_venta < 1000;
```

### Usar transacciones para operaciones peligrosas

Las transacciones son tu red de seguridad. Cuando ejecutas una operación riesgosa (como DELETE masivo), envuélvela en una transacción:

1. `START TRANSACTION` — comienza la transacción.
2. Ejecutas la operación (DELETE, UPDATE, etc.).
3. Verificas el resultado (por ejemplo, con `SELECT ROW_COUNT()` para ver cuántas filas se afectaron).
4. Si el resultado es correcto, ejecutas `COMMIT` para hacerlo permanente.
5. Si algo está mal, ejecutas `ROLLBACK` para deshacer TODO lo que hiciste desde el START TRANSACTION.

Nunca está de más: incluso los desarrolladores más experimentados usan transacciones para operaciones peligrosas.

```sql
START TRANSACTION;

-- Operación riesgosa
DELETE FROM productos WHERE activo = FALSE;

-- Verificar cuántos registros se eliminaron
SELECT ROW_COUNT() AS eliminados;

-- Si no es correcto, deshacer
ROLLBACK;

-- Si es correcto, confirmar
COMMIT;
```

---

## Anterior: [05 Subconsultas](../02-fundamentos/05-subconsultas.md)
## Siguiente: [07 Create Table Y Ddl](../02-fundamentos/07-create-table-y-ddl.md)

[Volver al índice](../README.md)
