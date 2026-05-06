# 2.2 WHERE y Filtros

## La Cláusula WHERE

En el capítulo anterior aprendiste a usar `SELECT` para traer datos de una tabla. Pero casi nunca quieres **todos** los datos. Si tienes 50,000 clientes, no te sirve de mucho verlos a todos; necesitas, por ejemplo, solo los clientes de "Guadalajara", o solo las facturas de "enero", o solo los productos con "stock menor a 10". La cláusula `WHERE` es el filtro de SQL: es como ponerle condiciones a tu consulta para que solo te traiga las filas que cumplen ciertos criterios. Sin `WHERE`, tu consulta trae todo. Con `WHERE`, le dices a la base de datos "de todo lo que tienes, solo quiero esto, esto y esto".

### Sintaxis

La estructura es simple: después del `FROM` (y antes del `ORDER BY` o `LIMIT` si los hay), escribes `WHERE` seguido de una o más condiciones. Cada condición es como una pregunta de sí/no para cada fila. Si la respuesta es "sí", la fila se incluye. Si es "no", se excluye.

```sql
SELECT columnas
FROM tabla
WHERE condicion(es);
```

## Operadores de Comparación

| Operador | Significado | Ejemplo |
|----------|-------------|---------|
| `=` | Igual a | `precio = 100` |
| `<>` o `!=` | Diferente de | `estado <> 'cancelada'` |
| `>` | Mayor que | `total > 1000` |
| `<` | Menor que | `stock < 10` |
| `>=` | Mayor o igual | `fecha >= '2024-01-01'` |
| `<=` | Menor o igual | `cantidad <= 5` |

Imagina que estos operadores son como los que usas en matemáticas desde la primaria, pero aplicados a bases de datos. El `=` verifica si dos cosas son exactamente iguales. El `<>` (o `!=`) pregunta si son diferentes. Los operadores de mayor/menor funcionan exactamente como en matemáticas, y funcionan no solo con números, sino también con fechas (una fecha más reciente es "mayor que" una más antigua) y con texto (orden alfabético). Estos operadores son los ladrillos básicos para construir cualquier filtro.

```sql
SELECT nombre, credito_limite, saldo_actual 
FROM clientes 
WHERE credito_limite > 10000;

SELECT folio, fecha_emision, total 
FROM facturas 
WHERE YEAR(fecha_emision) = 2024;

SELECT nombre_producto, precio_venta 
FROM productos 
WHERE precio_venta >= 100 AND precio_venta <= 500;
```

## Operadores Lógicos

### AND - Todas las condiciones deben cumplirse

El operador `AND` combina dos o más condiciones, y **todas** deben ser verdaderas para que una fila sea incluida. Piensa en ello como una lista de requisitos obligatorios: "Quiero facturas que sean activas **Y** de clientes premium **Y** con monto mayor a $5000". Si una factura no cumple aunque sea uno de esos requisitos, no aparece. Es como cuando tu jefe te dice "necesito un reporte de...", y cada condición que agrega reduce el conjunto de resultados.

```sql
SELECT * FROM facturas 
WHERE estado = 'activa' 
  AND id_cliente IN (SELECT id_cliente FROM clientes WHERE credito_limite > 50000)
  AND total > 5000;
```

### OR - Al menos una condición debe cumplirse

`OR` es más permisivo: con que **una** de las condiciones se cumpla, la fila es incluida. Es como decir "tráeme los productos que sean de electrónica **O** que cuesten más de $10,000". Un producto de electrónica que cueste $200 sí aplica, un producto de ropa que cueste $15,000 también, y un producto de electrónica que cueste $20,000 aplica por ambas razones. El único que se excluye es el que no cumple ninguna de las dos.

```sql
SELECT * FROM productos 
WHERE id_categoria = 1 OR precio_venta > 10000;
```

### NOT - Negación

`NOT` es el inversor: le das una condición y él te da todo lo que **no** la cumple. Es como decir "tráeme todo excepto esto". Por ejemplo, `NOT estado = 'cancelada'` te da todas las facturas excepto las canceladas. Puedes combinar `NOT` con otros operadores como `IN`: `id_categoria NOT IN (3)` te da los productos que no pertenecen a la categoría 3.

```sql
SELECT * FROM facturas 
WHERE NOT estado = 'cancelada';

SELECT * FROM productos 
WHERE id_categoria NOT IN (3);
```

## BETWEEN - Rangos

`BETWEEN` es un atajo para cuando quieres filtrar por un rango, ya sea de números o de fechas. En lugar de escribir `precio >= 500 AND precio <= 5000`, puedes escribir `precio BETWEEN 500 AND 5000`. Hace exactamente lo mismo pero es más limpio y fácil de leer. Es importante saber que `BETWEEN` es **inclusivo**: incluye los límites. O sea, `BETWEEN 500 AND 5000` incluye productos de exactamente $500 y exactamente $5000.

```sql
SELECT * FROM facturas 
WHERE fecha_emision BETWEEN '2024-01-01' AND '2024-03-31';

SELECT * FROM productos 
WHERE precio_venta BETWEEN 500 AND 5000;
```

## IN - Lista de Valores

Imagina que quieres productos de las categorías 1, 3, 5 y 7. Podrías escribir `id_categoria = 1 OR id_categoria = 3 OR id_categoria = 5 OR id_categoria = 7`, pero eso es muy verboso y propenso a errores. `IN` te permite pasar una lista de valores entre paréntesis: `id_categoria IN (1, 3, 5, 7)`. Es como decir "el valor debe ser uno de estos". Funciona con números y con texto: `estado IN ('enviada', 'confirmada', 'recibida_parcial')`. Es más corto, más claro, y más fácil de modificar.

```sql
SELECT * FROM productos 
WHERE id_categoria IN (1, 3, 5, 7);

SELECT * FROM ordenes_compra 
WHERE estado IN ('enviada', 'confirmada', 'recibida_parcial');

SELECT * FROM productos 
WHERE id_categoria = 1 OR id_categoria = 3 OR id_categoria = 5;
```

## LIKE - Búsqueda por Patrones

A veces no sabes el valor exacto que buscas. Tal vez recuerdas que el nombre del cliente empieza con "Mar" (María, Martha, Martín), o quieres encontrar todos los emails de Gmail (terminan en "@gmail.com"), o productos que contengan la palabra "Laptop" en cualquier parte del nombre. Para eso sirve `LIKE`: búsqueda por patrones usando **comodines**. El `%` significa "cualquier secuencia de caracteres (incluso cero)". El `_` significa "exactamente un carácter".

Comodines:
- `%` - Cualquier secuencia de caracteres (0 o más)
- `_` - Un solo carácter

Piénsalo así: `'Mar%'` se lee "que empiece con 'Mar' y después venga cualquier cosa". `'%@gmail.com'` se lee "que tenga cualquier cosa antes de '@gmail.com'". `'%Laptop%'` se lee "que tenga 'Laptop' en cualquier posición, con cualquier cosa antes o después".

```sql
SELECT * FROM clientes WHERE nombre LIKE 'Mar%';

SELECT * FROM clientes WHERE email LIKE '%@gmail.com';

SELECT * FROM productos WHERE nombre_producto LIKE '%Laptop%';

SELECT * FROM clientes WHERE rfc LIKE '_____________';

SELECT * FROM clientes WHERE nombre LIKE '____';
```

### LIKE con Escape

¿Qué pasa si necesitas buscar algo que contiene el mismo símbolo `%`? Por ejemplo, productos con "Descuento 100%" en el nombre. Como `%` es un comodín, necesitas una forma de decirle a SQL "este % en específico no es un comodín, es el carácter literal". Para eso existe `ESCAPE`: defines un carácter de escape (comúnmente la barra invertida `\`) y pones ese carácter antes del símbolo que quieres tratar como literal.

```sql
SELECT * FROM productos WHERE nombre_producto LIKE '%100\%%' ESCAPE '\';
```

## IS NULL / IS NOT NULL

Este es uno de los conceptos más importantes y confusos para los principiantes. En SQL, `NULL` no es un valor, sino la **ausencia** de valor. No es cero, no es una cadena vacía, no es "falso". Es "no sé", "no aplica", "aún no se ha registrado". Y aquí viene lo clave: `NULL = NULL` no es verdadero, es... también `NULL`. No puedes comparar NULL con el operador `=`; siempre debes usar `IS NULL` o `IS NOT NULL`. Es como preguntar "¿este valor desconocido es igual a este otro valor desconocido?" — no tienes manera de saberlo, así que la respuesta es también desconocida.

```sql
SELECT nombre, email FROM clientes WHERE telefono IS NULL;

SELECT * FROM productos WHERE descripcion IS NOT NULL;

SELECT * FROM ordenes_compra 
WHERE fecha_real_entrega IS NULL AND estado != 'cancelada';
```

⚠️ **Importante:** NULL no es igual a NULL. `NULL = NULL` es falso. Siempre usar `IS NULL`.

## Combinación de Operadores

Cuando combinas `AND`, `OR` y `NOT` en una misma consulta, SQL sigue un **orden de precedencia**: primero evalúa `NOT`, luego `AND`, y finalmente `OR`. Esto puede dar resultados inesperados si no usas paréntesis. La regla de oro es: **cuando tengas dudas, usa paréntesis**. Los paréntesis no solo hacen explícito lo que quieres, sino que también hacen el código más legible para otros programadores.

```sql
SELECT * FROM facturas 
WHERE estado = 'activa' OR estado = 'pendiente' AND total > 5000;

SELECT * FROM facturas 
WHERE (estado = 'activa' OR estado = 'pendiente') AND total > 5000;
```

## Filtros Avanzados con Fechas

Las fechas son uno de los tipos de datos más comunes en los filtros de negocio. Casi todos los reportes preguntan por períodos: "las ventas de hoy", "el mes actual", "el mes anterior", "los últimos 30 días". SQL tiene funciones como `CURRENT_DATE` (la fecha de hoy), `DATE_SUB()` (restar días a una fecha), `MONTH()` y `YEAR()` (extraer componentes), entre otras. Combinándolas puedes construir filtros dinámicos que siempre funcionen sin importar cuándo ejecutes la consulta.

```sql
SELECT * FROM facturas 
WHERE fecha_emision >= DATE_SUB(CURRENT_DATE, INTERVAL 30 DAY);

SELECT * FROM facturas 
WHERE MONTH(fecha_emision) = MONTH(CURRENT_DATE) 
  AND YEAR(fecha_emision) = YEAR(CURRENT_DATE);

SELECT * FROM facturas 
WHERE fecha_emision BETWEEN 
    DATE_FORMAT(DATE_SUB(CURRENT_DATE, INTERVAL 1 MONTH), '%Y-%m-01') 
    AND LAST_DAY(DATE_SUB(CURRENT_DATE, INTERVAL 1 MONTH));

SELECT * FROM ordenes_compra 
WHERE fecha_estimada_entrega < CURRENT_DATE 
  AND estado IN ('enviada', 'confirmada');
```

## Filtros con CASE

Aunque los filtros normalmente usan `WHERE`, a veces necesitas lógica condicional dentro del filtro mismo. Aquí `CASE` puede usarse dentro del `WHERE` para crear condiciones más complejas. Por ejemplo, "tráeme productos económicos (menos de $100) y también productos de las categorías 1 y 2 aunque cuesten más, pero solo hasta $1000". Escribir eso solo con `AND` y `OR` sería confuso; con `CASE` dentro del `WHERE` puedes expresar la lógica como una serie de condiciones que retornan verdadero o falso.

```sql
SELECT 
    nombre_producto,
    precio_venta,
    CASE 
        WHEN precio_venta < 100 THEN 'Económico'
        WHEN precio_venta BETWEEN 100 AND 1000 THEN 'Medio'
        WHEN precio_venta BETWEEN 1001 AND 10000 THEN 'Premium'
        ELSE 'Lujo'
    END AS categoria_precio
FROM productos
WHERE CASE 
    WHEN precio_venta < 100 THEN TRUE
    WHEN precio_venta BETWEEN 100 AND 1000 THEN TRUE
    WHEN id_categoria IN (1, 2) THEN TRUE
    ELSE FALSE
END;
```

## Filtros en JOIN (vs WHERE)

Cuando usas `LEFT JOIN`, el lugar donde pones el filtro cambia el resultado drásticamente. Si pones una condición en el `ON` del JOIN, filtras **antes** de unir las tablas. Si la pones en el `WHERE`, filtras **después** de unir. En un `LEFT JOIN`, filtrar en el `WHERE` sobre la tabla derecha puede convertir el LEFT JOIN en un INNER JOIN (porque las filas NULL que el LEFT JOIN habría generado se eliminan con el filtro). Es una sutileza que causa errores muy comunes.

```sql
SELECT c.nombre, f.folio, f.total
FROM clientes c
LEFT JOIN facturas f ON c.id_cliente = f.id_cliente 
    AND YEAR(f.fecha_emision) = 2024
WHERE c.activo = TRUE;

SELECT c.nombre, f.folio, f.total
FROM clientes c
LEFT JOIN facturas f ON c.id_cliente = f.id_cliente
WHERE c.activo = TRUE AND YEAR(f.fecha_emision) = 2024;
```

## 📊 Ejemplos Aplicados al Negocio

### Dashboard de Ventas - Filtros combinados

Este es un ejemplo real de un dashboard de ventas del día actual. Combina múltiples filtros en el `WHERE` (fecha de hoy, estado activa, total mayor a cero), después agrupa por cliente, y finalmente usa `HAVING` para quedarse solo con los clientes cuyas ventas totales superen los $1,000. Es un ejemplo práctico de cómo se ven las consultas en un sistema real: filtros, joins, agregaciones y ordenamiento trabajando juntos.

```sql
SELECT 
    c.nombre AS cliente,
    COUNT(f.id_factura) AS facturas,
    SUM(f.total) AS total_ventas,
    GROUP_CONCAT(DISTINCT fp.descripcion SEPARATOR ', ') AS formas_pago
FROM facturas f
JOIN clientes c ON f.id_cliente = c.id_cliente
LEFT JOIN formas_pago fp ON f.metodo_pago = fp.codigo
WHERE f.fecha_emision >= CURDATE()
    AND f.fecha_emision < CURDATE() + INTERVAL 1 DAY
    AND f.estado = 'activa'
    AND f.total > 0
GROUP BY c.id_cliente
HAVING total_ventas > 1000
ORDER BY total_ventas DESC;
```

### Alerta de Inventario

Esta consulta resuelve un problema de negocio muy concreto: detectar productos que necesitan reorden urgente. No solo busca productos con stock por debajo del mínimo, sino que además verifica que no haya ya una orden de compra en proceso para ese producto (usando `NOT EXISTS`). Además calcula cuánto costaría reponer el inventario. Es el tipo de consulta que alimenta un sistema de alertas automáticas.

```sql
SELECT 
    p.sku,
    p.nombre_producto,
    p.stock_actual,
    p.stock_minimo,
    p.precio_compra,
    (p.stock_minimo - p.stock_actual) AS cantidad_a_ordenar,
    (p.stock_minimo - p.stock_actual) * p.precio_compra AS costo_reposicion
FROM productos p
WHERE p.stock_actual < p.stock_minimo
    AND p.activo = TRUE
    AND NOT EXISTS (
        SELECT 1 FROM ordenes_compra_detalle ocd
        JOIN ordenes_compra oc ON ocd.id_orden = oc.id_orden
        WHERE ocd.id_producto = p.id_producto
        AND oc.estado IN ('enviada', 'confirmada', 'recibida_parcial')
    )
ORDER BY (p.stock_minimo - p.stock_actual) DESC;
```

### Análisis de Clientes por Segmento

Para estrategias de marketing, necesitas saber quiénes son tus mejores clientes. Esta consulta calcula para cada cliente: cuántas compras ha hecho, cuánto ha gastado en total, cuándo fue su última compra, y cuántos días han pasado desde entonces. El filtro `WHERE` excluye clientes sin crédito ni saldo (probablemente clientes dados de baja), y el `HAVING` elimina clientes que nunca han comprado (para quedarse solo con los que tienen actividad).

```sql
SELECT 
    c.nombre,
    c.credito_limite,
    COUNT(f.id_factura) AS compras_realizadas,
    SUM(f.total) AS total_gastado,
    MAX(f.fecha_emision) AS ultima_compra,
    DATEDIFF(CURRENT_DATE, MAX(f.fecha_emision)) AS dias_sin_compra
FROM clientes c
LEFT JOIN facturas f ON c.id_cliente = f.id_cliente AND f.estado = 'activa'
WHERE c.activo = TRUE
    AND (c.credito_limite > 0 OR c.saldo_actual > 0)
GROUP BY c.id_cliente
HAVING dias_sin_compra IS NOT NULL
ORDER BY total_gastado DESC;
```

### Validaciones de Datos

Uno de los usos más valiosos de SQL es la limpieza y validación de datos. Esta consulta usa `UNION ALL` para combinar varias verificaciones en un solo resultado: precios negativos, stocks negativos, facturas activas sin detalle, y cantidades cero en detalle. Es una consulta de auditoría que cualquier negocio debería ejecutar periódicamente para mantener la calidad de sus datos.

```sql
SELECT 
    'Precios negativos' AS tipo_error,
    COUNT(*) AS registros
FROM productos 
WHERE precio_venta <= 0 OR precio_compra <= 0

UNION ALL

SELECT 
    'Stock negativo' AS tipo_error,
    COUNT(*) AS registros
FROM productos 
WHERE stock_actual < 0

UNION ALL

SELECT 
    'Facturas sin detalle' AS tipo_error,
    COUNT(*) AS registros
FROM facturas f
WHERE NOT EXISTS (
    SELECT 1 FROM facturas_detalle fd WHERE fd.id_factura = f.id_factura
)
    AND f.estado = 'activa'

UNION ALL

SELECT 
    'Cantidades cero en detalle' AS tipo_error,
    COUNT(*) AS registros
FROM facturas_detalle 
WHERE cantidad <= 0;
```

### Búsqueda Avanzada de Productos

Esta consulta simula el motor de búsqueda de un catálogo de productos. Permite buscar por nombre, código de barras, SKU o categoría usando `LIKE`. Además tiene filtros opcionales de precio y categoría. Lo interesante es el `ORDER BY` que usa `CASE` para ordenar los resultados por relevancia: primero los que empiezan exactamente con el término de búsqueda, luego los que lo contienen, y finalmente los que coinciden por otros campos.

```sql
SELECT 
    p.codigo_barras,
    p.nombre_producto,
    c.nombre AS categoria,
    p.precio_venta,
    p.stock_actual
FROM productos p
JOIN categorias c ON p.id_categoria = c.id_categoria
WHERE p.activo = TRUE
    AND (
        p.nombre_producto LIKE '%búsqueda%'
        OR p.codigo_barras LIKE '%búsqueda%'
        OR p.sku LIKE '%búsqueda%'
        OR c.nombre LIKE '%búsqueda%'
    )
    AND (p.precio_venta BETWEEN 0 AND 999999)
    AND (c.id_categoria IN (1,2,3,4) OR 1=1)
ORDER BY 
    CASE 
        WHEN p.nombre_producto LIKE 'búsqueda%' THEN 1
        WHEN p.nombre_producto LIKE '%búsqueda%' THEN 2
        WHEN p.codigo_barras LIKE '%búsqueda%' THEN 3
        ELSE 4
    END,
    p.nombre_producto
LIMIT 50;
```

### Conciliación Bancaria

Para el departamento de finanzas, es crucial saber cómo están pagando los clientes. Esta consulta agrupa las ventas por método de pago en un período específico, mostrando cuántas transacciones hubo, el monto total, el ticket promedio, y los valores mínimo y máximo. Es útil para conciliaciones bancarias y para entender las preferencias de pago de los clientes.

```sql
SELECT 
    CASE metodo_pago
        WHEN 'efectivo' THEN 'Efectivo'
        WHEN 'tarjeta_credito' THEN 'Tarjeta Crédito'
        WHEN 'tarjeta_debito' THEN 'Tarjeta Débito'
        WHEN 'transferencia' THEN 'Transferencia'
        ELSE 'Otro'
    END AS forma_cobro,
    COUNT(*) AS cantidad,
    SUM(total) AS monto_total,
    ROUND(AVG(total), 2) AS ticket_promedio,
    MIN(total) AS ticket_minimo,
    MAX(total) AS ticket_maximo
FROM facturas
WHERE fecha_emision BETWEEN '2024-01-01' AND '2024-01-31'
    AND estado = 'activa'
GROUP BY metodo_pago
ORDER BY monto_total DESC;
```

## Errores Comunes con WHERE

### 1. Usar = con NULL

Este es el error más frecuente entre principiantes. Como NULL no es un valor sino la ausencia de él, no puedes preguntar "¿esto es igual a NULL?" porque NULL no es igual a nada, ni siquiera a otro NULL. La forma correcta es preguntar "¿esto **es** NULL?" usando `IS NULL`. Recuerda esta regla de memoria: **NULL nunca se compara con =, siempre con IS**.

```sql
SELECT * FROM clientes WHERE telefono = NULL;

SELECT * FROM clientes WHERE telefono IS NULL;
```

### 2. Olvidar las comillas en strings

En SQL, los valores de texto (strings) deben ir entre comillas simples (`'Juan'`). Si escribes `nombre = Juan` sin comillas, SQL piensa que "Juan" es un nombre de columna o de tabla, no un valor. Recibirás un error como "columna no encontrada". Este error es tan común que incluso los desarrolladores experimentados lo cometen de vez en cuando.

```sql
SELECT * FROM clientes WHERE nombre = Juan;

SELECT * FROM clientes WHERE nombre = 'Juan';
```

### 3. Confundir AND con OR

Un error lógico muy común: cuando quieres productos de dos categorías, podrías pensar "dame productos donde la categoría sea 1 Y también sea 2". Pero eso es imposible: una fila no puede tener dos valores diferentes en la misma columna. Lo correcto es usar `OR`: "dame productos donde la categoría sea 1 **O** sea 2". Piensa en `AND` como filtros que se aplican a diferentes columnas, y `OR` como alternativas para la misma columna.

```sql
SELECT * FROM productos 
WHERE id_categoria = 1 OR id_categoria = 2;

SELECT * FROM productos 
WHERE id_categoria = 1 AND id_categoria = 2;
```

### 4. Usar LIKE sin comodines (es =)

Si usas `LIKE` sin los comodines `%` o `_`, se comporta exactamente igual que `=`. No hay diferencia entre `nombre LIKE 'Juan'` y `nombre = 'Juan'`. El poder de `LIKE` está en los patrones: `LIKE 'Juan%'` te da "Juan", "Juana", "Juanito". Si no usas comodines, mejor usa `=` para que tu intención sea clara.

```sql
SELECT * FROM clientes WHERE nombre LIKE 'Juan';
```

---

## Anterior: [01 Select Basico](../02-fundamentos/01-select-basico.md)
## Siguiente: [03 Join Y Relaciones](../02-fundamentos/03-join-y-relaciones.md)

[Volver al índice](../README.md)
