# 2.5 Subconsultas

## Introducción

Imagina que estás resolviendo un problema en varios pasos. Primero preguntas algo, y con esa respuesta, haces otra pregunta. Eso es exactamente una subconsulta: una consulta SQL que está dentro de otra consulta. La subconsulta se ejecuta primero (o al menos conceptualmente), y su resultado se le pasa a la consulta principal para que termine su trabajo.

Las subconsultas son increíblemente útiles cuando necesitas responder preguntas como: "¿Qué productos tienen un precio mayor al promedio?", "¿Qué clientes han comprado algo en el último mes?", o "¿Qué facturas superan el promedio de gasto de su propio cliente?". Sin subconsultas, tendrías que hacer dos consultas separadas y luego combinar los resultados manualmente (o con código de tu aplicación). Con subconsultas, todo se hace en una sola instrucción SQL.

### Tipos de Subconsultas

Antes de ver ejemplos concretos, es importante que conozcas los tipos de subconsulta que existen. Se clasifican según la cantidad de datos que devuelven. Piensa en esto como diferentes "tamaños" de respuesta:

Una **subconsulta escalar** devuelve un único valor: una sola fila y una sola columna. Es como preguntar "¿Cuál es el promedio de precios?" — la respuesta es un número, nada más. Se usa donde normalmente pondrías un valor simple, como en un SELECT, en un WHERE, o incluso en un SET de un UPDATE.

Una **subconsulta de fila** devuelve una sola fila pero con varias columnas. Es como preguntar "Dame los datos del producto más caro" — obtienes una fila completa con nombre, precio, categoría, etc.

Una **subconsulta de tabla** devuelve múltiples filas y múltiples columnas. Es como preguntar "Dame todos los productos de la categoría electrónicos" — obtienes una lista completa. Se usa con IN, EXISTS, o como tabla derivada en un FROM.

Una **subconsulta correlacionada** es especial: hace referencia a columnas de la consulta exterior. Esto significa que la subconsulta NO se ejecuta una sola vez, sino que se ejecuta UNA VEZ POR CADA FILA de la consulta principal. Es como si para cada cliente preguntaras "¿Este cliente específico ha comprado algo?". Es más lenta pero muy poderosa.

```
Escalar     → Retorna un solo valor (1 fila, 1 columna)
De fila     → Retorna 1 fila con múltiples columnas
De tabla    → Retorna múltiples filas y columnas
Correlacionada → Depende de la consulta exterior
```

## Subconsultas Escalares

Imagina que trabajas en un negocio y necesitas generar un reporte que muestre cada producto, su precio, y además el precio promedio de todos los productos para que se pueda ver la diferencia. Necesitas calcular ese promedio general UNA SOLA VEZ y mostrarlo junto a cada producto. Eso es exactamente lo que hace una subconsulta escalar: calcular un valor único (el promedio general) y usarlo como si fuera una columna más en el SELECT.

La subconsulta `(SELECT AVG(precio_venta) FROM productos)` se ejecuta primero y devuelve un solo número: el precio promedio de todos los productos. Ese número se coloca en cada fila del resultado como si fuera otra columna. Luego, en la cuarta línea, RESTAMOS ese promedio del precio de cada producto para obtener la diferencia. Es como tener una calculadora dentro de tu SELECT.

El segundo ejemplo responde a una pregunta muy común en cualquier negocio: "¿Qué clientes gastaron más que el promedio general?". Primero, la subconsulta `(SELECT AVG(total) FROM facturas WHERE estado = 'activa')` calcula cuánto es el ticket promedio de todas las facturas activas. Luego, la consulta principal selecciona solo aquellas facturas cuyo total supera ese número. Así de simple: en una sola consulta puedes comparar cada fila contra un valor calculado de toda la tabla.

```sql
SELECT 
    nombre_producto,
    precio_venta,
    (SELECT AVG(precio_venta) FROM productos) AS precio_promedio,
    precio_venta - (SELECT AVG(precio_venta) FROM productos) AS diferencia
FROM productos;

-- Clientes con compras superiores al promedio
SELECT 
    c.nombre,
    f.total,
    f.fecha_emision
FROM clientes c
JOIN facturas f ON c.id_cliente = f.id_cliente
WHERE f.total > (SELECT AVG(total) FROM facturas WHERE estado = 'activa')
ORDER BY f.total DESC;
```

### En columnas calculadas

Ahora imagina un problema más específico. Tienes un catálogo de productos y quieres saber, para CADA producto, cuántas unidades se han vendido. Pero no tienes una columna "total_vendido" en la tabla productos — esa información está en las tablas de facturas. Necesitas, para cada producto, ir a las facturas, sumar las cantidades vendidas de ese producto, y traer ese número.

Aquí usamos una subconsulta escalar dentro del SELECT que, para cada producto, calcula la suma de cantidades vendidas en facturas activas. La subconsulta se correlaciona con el producto de afuera mediante `fd.id_producto = p.id_producto`. Esto significa que la subconsulta se ejecuta una vez por cada producto, sumando solo las ventas de ESE producto específico. Es como tener un asistente que, por cada producto que ves, corre a revisar las ventas y te trae el total.

```sql
SELECT 
    p.nombre_producto,
    p.stock_actual,
    (SELECT SUM(cantidad) FROM facturas_detalle fd 
     JOIN facturas f ON fd.id_factura = f.id_factura 
     WHERE fd.id_producto = p.id_producto AND f.estado = 'activa'
    ) AS unidades_vendidas
FROM productos p;
```

## Subconsultas con IN

Piensa en IN como en una lista de verificación. Le preguntas a SQL: "¿Este valor está dentro de esta lista?". Normalmente tú escribes la lista manualmente: `WHERE id_cliente IN (1, 2, 3)`. Pero ¿qué pasa si la lista es el resultado de otra consulta? Por ejemplo, "dame todos los clientes que aparecen en la tabla de facturas". En lugar de adivinar los números, le pides a SQL que primero obtenga la lista de clientes que han facturado, y luego seleccione los datos completos de esos clientes.

El primer ejemplo busca todos los productos que ALGUNA VEZ se han vendido. La subconsulta `SELECT DISTINCT id_producto FROM facturas_detalle` obtiene la lista de IDs de productos que aparecen en el detalle de facturas (es decir, que se han vendido). Luego, la consulta principal selecciona los datos completos de esos productos. Es mucho más eficiente que traerte todos los productos y filtrar en tu código.

El segundo ejemplo es muy común en reportes de negocio: "¿Qué clientes han comprado en el último mes?". La subconsulta obtiene los IDs de clientes con facturas emitidas en los últimos 30 días, usando `DATE_SUB(CURRENT_DATE, INTERVAL 1 MONTH)` que calcula la fecha de hace un mes. EXTREMADAMENTE útil para campañas de marketing o para identificar clientes activos.

El tercer ejemplo muestra cómo anidar lógica de negocios: "productos de categorías que han generado más de $10,000 en ventas". La subconsulta primero agrupa por categoría, suma los totales de ventas activas, y filtra con HAVING las categorías que superan los $10,000. Luego, la consulta principal selecciona los productos que pertenecen a esas categorías exitosas. Son dos niveles de análisis: categorías rentables → productos de esas categorías.

```sql
-- Productos que se han vendido
SELECT * FROM productos 
WHERE id_producto IN (
    SELECT DISTINCT id_producto FROM facturas_detalle
);

-- Clientes que han comprado en el último mes
SELECT * FROM clientes 
WHERE id_cliente IN (
    SELECT DISTINCT id_cliente FROM facturas 
    WHERE fecha_emision >= DATE_SUB(CURRENT_DATE, INTERVAL 1 MONTH)
);

-- Productos de categorías que han tenido ventas > $10,000
SELECT * FROM productos 
WHERE id_categoria IN (
    SELECT p.id_categoria
    FROM productos p
    JOIN facturas_detalle fd ON p.id_producto = fd.id_producto
    JOIN facturas f ON fd.id_factura = f.id_factura AND f.estado = 'activa'
    GROUP BY p.id_categoria
    HAVING SUM(fd.total) > 10000
);
```

### NOT IN (con cuidado)

Así como IN te dice "está en la lista", NOT IN te dice "NO está en la lista". El primer ejemplo encuentra productos que NUNCA se han vendido (no aparecen en facturas_detalle). Esto es útil para identificar producto obsoleto o inventario muerto.

PERO CUIDADO: NOT IN tiene una trampa muy peligrosa con los valores NULL. Mira el segundo ejemplo: `WHERE id_producto NOT IN (1, 2, NULL)`. Intuitivamente pensarías que esto devuelve los productos que no son 1, 2, o NULL. Pero en SQL, `NULL` significa "valor desconocido". Comparar cualquier cosa con NULL da como resultado "desconocido", no TRUE ni FALSE. Entonces, para cada producto, SQL pregunta: "¿Es este producto diferente de NULL?". La respuesta es "no lo sé", así que la fila NO se incluye. El resultado SIEMPRE será vacío si hay al menos un NULL en la lista de NOT IN.

**Regla de oro**: Si existe la menor posibilidad de que haya NULLs en la subconsulta, usa NOT EXISTS en lugar de NOT IN. Es más seguro y además suele ser más rápido.

```sql
-- Productos que NUNCA se han vendido
SELECT * FROM productos 
WHERE id_producto NOT IN (
    SELECT DISTINCT id_producto FROM facturas_detalle
);

-- ⚠️ NOT IN con NULL: Si la subconsulta retorna NULL, el resultado es siempre vacío
SELECT * FROM productos 
WHERE id_producto NOT IN (1, 2, NULL);  -- Retorna 0 filas siempre
-- Usar NOT EXISTS en su lugar
```

## Subconsultas con EXISTS / NOT EXISTS

EXISTS es como preguntar "¿Existe al menos un registro que cumpla esta condición?". A diferencia de IN, que primero recolecta TODOS los valores de la subconsulta y luego los compara, EXISTS funciona con un mecanismo de "cortocircuito": en cuanto encuentra UNA sola fila que cumpla la condición, se detiene y retorna TRUE. No necesita seguir buscando ni construir una lista completa.

El primer ejemplo es el clásico: "clientes que han realizado al menos una compra". Observa que en la subconsulta escribimos `SELECT 1`, no `SELECT *` ni `SELECT id_cliente`. Esto es importante: a EXISTS solo le importa si HAY filas, no QUÉ hay en esas filas. Podríamos escribir `SELECT *` o `SELECT 1` o `SELECT NULL` y daría igual — lo único que importa es si la subconsulta devuelve al menos una fila. Usar `SELECT 1` es una convención que deja claro que solo nos interesa la existencia, no los datos.

El segundo ejemplo, con NOT EXISTS, encuentra productos que NO tienen movimientos de inventario (probablemente productos nuevos o datos incorrectos). Esto es útil para auditoría o limpieza de datos.

El tercer ejemplo es muy de negocio: "proveedores con órdenes de compra pendientes". La subconsulta verifica si existe al menos una orden cuyo estado sea 'enviada' o 'confirmada' para ese proveedor. Si existe, el proveedor aparece en el resultado.

```sql
-- Clientes que han realizado al menos una compra
SELECT c.*
FROM clientes c
WHERE EXISTS (
    SELECT 1 FROM facturas f 
    WHERE f.id_cliente = c.id_cliente AND f.estado = 'activa'
);

-- Productos sin movimientos de inventario
SELECT p.*
FROM productos p
WHERE NOT EXISTS (
    SELECT 1 FROM inventario_movimientos m 
    WHERE m.id_producto = p.id_producto
);

-- Proveedores con órdenes pendientes de entrega
SELECT p.*
FROM proveedores p
WHERE EXISTS (
    SELECT 1 FROM ordenes_compra oc 
    WHERE oc.id_proveedor = p.id_proveedor 
    AND oc.estado IN ('enviada', 'confirmada')
);
```

### EXISTS vs IN

Esta es una de las decisiones más importantes que tomarás como desarrollador SQL. Ambas pueden usarse para resolver problemas similares, pero su funcionamiento interno es muy diferente.

**IN** recolecta TODOS los resultados de la subconsulta primero, los almacena en memoria, y luego compara cada fila de la consulta principal contra esa lista. Si la subconsulta devuelve 100,000 IDs de facturas, IN construye una lista en memoria con esos 100,000 valores. Esto consume memoria y tiempo.

**EXISTS**, en cambio, usa cortocircuito. Para cada fila de la consulta principal, ejecuta la subconsulta y se detiene en cuanto encuentra una coincidencia. No construye ninguna lista en memoria. En cuanto encuentra la primera factura de un cliente, dice "sí, existe" y pasa al siguiente cliente.

¿Cuándo usar cada uno? La regla práctica es: si la subconsulta es grande y solo necesitas verificar existencia, usa EXISTS. Si la subconsulta es pequeña o necesitas los valores para otra cosa, IN está bien. Además, EXISTS maneja NULLs correctamente, mientras que NOT IN falla con NULLs.

```sql
-- EXISTS es generalmente más eficiente que IN cuando:
-- 1. La subconsulta es grande
-- 2. Se necesita evaluar existencia, no valores específicos

-- IN (evaluación perezosa)
SELECT * FROM clientes 
WHERE id_cliente IN (SELECT id_cliente FROM facturas);

-- EXISTS (mejor performance, cortocircuito)
SELECT * FROM clientes c
WHERE EXISTS (SELECT 1 FROM facturas f WHERE f.id_cliente = c.id_cliente);
```

## Subconsultas Correlacionadas

Aquí es donde las subconsultas se vuelven realmente poderosas. Una subconsulta correlacionada es aquella que hace referencia a columnas de la consulta PRINCIPAL (la de afuera). Esto crea una dependencia: la subconsulta NO puede ejecutarse de forma independiente porque necesita valores de la consulta exterior.

Piensa en esto como un bucle: para CADA producto en la tabla principal, la subconsulta calcula el promedio de precios de la MISMA categoría a la que pertenece ese producto. Luego pregunta: "¿El precio de este producto es mayor que el promedio de su propia categoría?". Si es así, lo incluye en el resultado.

En el primer ejemplo, la subconsulta `WHERE p2.id_categoria = p1.id_categoria` hace referencia a `p1.id_categoria`, que es una columna de la consulta exterior (alias p1). Por eso se llama "correlacionada": la subconsulta está correlacionada (relacionada) con la fila actual de la consulta principal.

El segundo ejemplo muestra clientes que compraron en enero de 2024. La subconsulta busca facturas de ese cliente específico en ese rango de fechas. NOTA: esta consulta también se podría escribir con un JOIN, pero EXISTS es más intuitivo cuando solo te importa la existencia.

Un detalle importante: las subconsultas correlacionadas suelen ser más lentas que sus equivalentes con JOIN, porque la subconsulta se ejecuta una vez por cada fila de la tabla exterior. Sin embargo, son más fáciles de leer en ciertos casos, especialmente con EXISTS.

```sql
-- Productos cuyo precio es mayor al promedio de su categoría
SELECT p1.nombre_producto, p1.precio_venta, c.nombre AS categoria
FROM productos p1
JOIN categorias c ON p1.id_categoria = c.id_categoria
WHERE p1.precio_venta > (
    SELECT AVG(p2.precio_venta) 
    FROM productos p2 
    WHERE p2.id_categoria = p1.id_categoria
)
ORDER BY c.nombre, p1.precio_venta DESC;

-- Clientes con compras en fechas específicas
SELECT c.nombre, c.email
FROM clientes c
WHERE EXISTS (
    SELECT 1 FROM facturas f 
    WHERE f.id_cliente = c.id_cliente
    AND f.fecha_emision BETWEEN '2024-01-01' AND '2024-01-31'
    AND f.estado = 'activa'
);
```

### Actualización con Subconsulta Correlacionada

Las subconsultas correlacionadas no solo sirven para SELECT, también funcionan en UPDATE. Aquí tienes un ejemplo práctico de negocio: marcar como inactivos aquellos productos que no se han vendido en los últimos 2 años.

La lógica es: para cada producto, verifica si existe al menos un registro en facturas_detalle (unido con facturas activas) de los últimos 2 años donde ese producto aparezca. Si NO existe (NOT EXISTS), el producto se marca como inactivo. Esto es un proceso típico de limpieza de catálogo que se ejecutaría periódicamente.

```sql
-- Marcar productos como inactivos si no se han vendido en 2 años
UPDATE productos p
SET p.activo = FALSE
WHERE NOT EXISTS (
    SELECT 1 FROM facturas_detalle fd
    JOIN facturas f ON fd.id_factura = f.id_factura
    WHERE fd.id_producto = p.id_producto
    AND f.fecha_emision >= DATE_SUB(CURRENT_DATE, INTERVAL 2 YEAR)
);
```

## Subconsultas en FROM (Derived Tables)

A veces necesitas hacer una consulta sobre los resultados de OTRA consulta. Es decir, primero calculas algo (como ventas diarias), y luego usas ese resultado como si fuera una tabla para hacer más cálculos. Esto se llama "tabla derivada" (derived table) y se escribe poniendo una subconsulta dentro del FROM.

La subconsulta en FROM debe tener SIEMPRE un alias (en este caso `AS ventas_por_dia`), porque SQL necesita un nombre para referirse a ella como si fuera una tabla.

En el ejemplo, la subconsulta interna calcula el total de ventas por día para el año 2024. El resultado es una tabla temporal con dos columnas: `dia` y `total_diario`. Luego, la consulta principal selecciona de esa tabla temporal y además calcula qué porcentaje representa cada día del total anual. Para obtener el total anual, usa OTRA subconsulta escalar `(SELECT SUM(total) FROM facturas WHERE YEAR(fecha_emision) = 2024)`.

Es como construir un reporte en capas: primero preparas los datos base (ventas por día), y luego agregas más análisis sobre esos datos (porcentaje del total).

```sql
SELECT 
    ventas_por_dia.dia,
    ventas_por_dia.total_diario,
    ventas_por_dia.total_diario / (SELECT SUM(total) FROM facturas WHERE YEAR(fecha_emision) = 2024) * 100 AS porcentaje_diario
FROM (
    SELECT 
        DATE(fecha_emision) AS dia,
        SUM(total) AS total_diario
    FROM facturas
    WHERE estado = 'activa' AND YEAR(fecha_emision) = 2024
    GROUP BY DATE(fecha_emision)
) AS ventas_por_dia
ORDER BY ventas_por_dia.dia;
```

### Top N por grupo con subconsulta en FROM

Este es uno de los patrones más útiles en análisis de datos: obtener los "Top N" elementos dentro de cada grupo. Por ejemplo, "los 3 productos más vendidos de cada categoría".

El truco está en usar la función de ventana `ROW_NUMBER()`, que asigna un número de ranking a cada producto dentro de su categoría (gracias a `PARTITION BY c.id_categoria`), ordenado por unidades vendidas de mayor a menor. La subconsulta en FROM prepara estos datos con el ranking, y la consulta principal simplemente filtra `WHERE ranking <= 3`.

Sin una subconsulta en FROM, no podrías filtrar por ranking porque `ROW_NUMBER()` se calcula en el momento del SELECT y no puedes usarlo en el WHERE de la misma consulta (el WHERE se ejecuta ANTES que las funciones de ventana). Por eso necesitas la subconsulta: primero calculas el ranking, luego filtras.

```sql
-- Top 3 productos más vendidos por categoría
SELECT categoria, producto, unidades_vendidas
FROM (
    SELECT 
        c.nombre AS categoria,
        p.nombre_producto AS producto,
        SUM(fd.cantidad) AS unidades_vendidas,
        ROW_NUMBER() OVER (
            PARTITION BY c.id_categoria 
            ORDER BY SUM(fd.cantidad) DESC
        ) AS ranking
    FROM categorias c
    JOIN productos p ON c.id_categoria = p.id_categoria
    JOIN facturas_detalle fd ON p.id_producto = fd.id_producto
    JOIN facturas f ON fd.id_factura = f.id_factura AND f.estado = 'activa'
    GROUP BY c.id_categoria, p.id_producto
) AS ranked
WHERE ranking <= 3
ORDER BY categoria, ranking;
```

## Subconsultas con ANY y ALL

Estos operadores son menos conocidos pero muy útiles en ciertos escenarios. Funcionan con operadores de comparación (`>`, `<`, `=`, etc.) combinados con una subconsulta que devuelve múltiples valores.

**ANY** significa "compara contra cualquier valor de la lista". `precio > ANY (...)` es TRUE si el precio es mayor que AL MENOS UNO de los valores devueltos por la subconsulta. En el ejemplo, busca productos cuyo precio sea mayor que el precio de al menos un producto de electrónica. En otras palabras, encuentra productos que son más caros que el producto más barato de electrónica.

**ALL** significa "compara contra TODOS los valores de la lista". `precio > ALL (...)` es TRUE solo si el precio es mayor que CADA UNO de los valores devueltos. En el ejemplo, busca productos cuyo precio sea mayor que TODOS los productos de electrónica. Esto equivale a "más caro que el producto más caro de electrónica".

Puedes pensar en ANY como un "OR" (si cumple con cualquiera, es suficiente) y ALL como un "AND" (debe cumplir con todos).

```sql
-- ANY: TRUE si la condición se cumple para ALGÚN valor
-- Productos más caros que al menos un producto de electrónica
SELECT * FROM productos 
WHERE precio_venta > ANY (
    SELECT precio_venta FROM productos WHERE id_categoria = 1
);

-- ALL: TRUE si la condición se cumple para TODOS los valores
-- Productos más caros que TODOS los de electrónica
SELECT * FROM productos 
WHERE precio_venta > ALL (
    SELECT precio_venta FROM productos WHERE id_categoria = 1
);
```

## Subconsultas con Comparación de Múltiples Columnas

SQL te permite comparar múltiples columnas a la vez usando paréntesis. En lugar de escribir `WHERE col1 IN (SELECT col1 FROM ...) AND col2 IN (SELECT col2 FROM ...)`, puedes escribir `WHERE (col1, col2) IN (SELECT col1, col2 FROM ...)`.

El ejemplo encuentra pares (categoría, precio) que aparecen más de una vez en la tabla de productos. Esto es útil para detectar duplicados o productos con el mismo precio dentro de una categoría. La subconsulta agrupa por categoría y precio, y con HAVING COUNT(*) > 1 encuentra las combinaciones que se repiten.

```sql
-- Encontrar los pares (categoría, precio) más comunes
SELECT DISTINCT p1.categoria, p1.precio
FROM productos p1
WHERE (p1.id_categoria, p1.precio_venta) IN (
    SELECT p2.id_categoria, p2.precio_venta
    FROM productos p2
    GROUP BY p2.id_categoria, p2.precio_venta
    HAVING COUNT(*) > 1
);
```

## LATERAL (PostgreSQL)

LATERAL es una característica avanzada de PostgreSQL (y otros motores) que permite que una subconsulta en FROM haga referencia a columnas de tablas que aparecieron ANTES en la misma cláusula FROM.

Normalmente, una subconsulta en FROM no puede ver las columnas de otras tablas en el mismo FROM. Pero con LATERAL, sí puede. Esto es útil cuando necesitas calcular algo diferente para cada fila de la tabla anterior.

El ejemplo clásico: "las últimas 3 facturas de cada cliente". La subconsulta con LATERAL recibe el `c.id_cliente` de la consulta exterior y, para cada cliente, obtiene sus últimas 3 facturas. Sin LATERAL, esto sería mucho más difícil de lograr, requeriría funciones de ventana o subconsultas correlacionadas más complejas.

```sql
-- PostgreSQL: Últimas 3 facturas de cada cliente
SELECT 
    c.nombre AS cliente,
    ventas_recientes.folio,
    ventas_recientes.total,
    ventas_recientes.fecha_emision
FROM clientes c
CROSS JOIN LATERAL (
    SELECT folio, total, fecha_emision
    FROM facturas f
    WHERE f.id_cliente = c.id_cliente
    ORDER BY f.fecha_emision DESC
    LIMIT 3
) AS ventas_recientes
WHERE c.activo = TRUE;
```

## 📊 Ejemplos del Mundo Real

### Detección de Anomalías

En cualquier negocio, es crucial detectar situaciones anómalas. Por ejemplo, una factura cuyo monto sea mucho mayor de lo que el cliente suele gastar. Esto podría indicar un error, un fraude, o una venta inusualmente grande que merece atención.

La consulta calcula, para cada factura, cuál es el promedio de gasto histórico de ese cliente (usando una subconsulta escalar correlacionada). Luego calcula cuántas veces el total de la factura supera ese promedio (`f.total / NULLIF(promedio_cliente, 0)`). `NULLIF` evita la división entre cero si el cliente no tiene facturas previas.

Finalmente, filtra solo aquellas facturas cuyo total es más de 3 VECES el promedio del cliente. El `3 * (SELECT AVG...)` en el WHERE es un umbral de anomalía que puedes ajustar según tu negocio.

```sql
-- Facturas con montos muy superiores al promedio del cliente
SELECT 
    c.nombre,
    f.folio,
    f.total,
    f.fecha_emision,
    (SELECT AVG(f2.total) FROM facturas f2 
     WHERE f2.id_cliente = c.id_cliente AND f2.estado = 'activa') AS promedio_cliente,
    f.total / NULLIF((SELECT AVG(f2.total) FROM facturas f2 
                      WHERE f2.id_cliente = c.id_cliente AND f2.estado = 'activa'), 0) AS veces_promedio
FROM clientes c
JOIN facturas f ON c.id_cliente = f.id_cliente
WHERE f.estado = 'activa'
AND f.total > 3 * (
    SELECT AVG(f2.total) FROM facturas f2 
    WHERE f2.id_cliente = c.id_cliente AND f2.estado = 'activa'
);
```

### Recomendación de Productos

Este es el clásico sistema de "quien compró X también compró Y" que ves en Amazon y otras tiendas. Se basa en un concepto llamado "análisis de cestas de compra" (market basket analysis).

La lógica es: para un producto específico (por ejemplo, el producto con id=100), encuentra todos los productos que aparecen en las MISMAS facturas. La tabla `facturas_detalle` se une dos veces: una para el producto base (fd1) y otra para el producto recomendado (fd2), asegurando que ambas líneas pertenezcan a la misma factura (`f1.id_factura = fd2.id_factura`) pero sean productos diferentes (`fd2.id_producto <> fd1.id_producto`).

Finalmente, cuenta cuántas facturas contienen ambos productos y ordena de mayor a menor. Los productos que más veces aparecen junto al producto base son las mejores recomendaciones.

```sql
-- Productos que suelen comprarse juntos (quien compró X también compró Y)
SELECT 
    p1.nombre_producto AS producto_base,
    p2.nombre_producto AS producto_recomendado,
    COUNT(DISTINCT f1.id_factura) AS veces_juntos
FROM facturas_detalle fd1
JOIN facturas f1 ON fd1.id_factura = f1.id_factura
JOIN productos p1 ON fd1.id_producto = p1.id_producto
JOIN facturas_detalle fd2 ON f1.id_factura = fd2.id_factura AND fd2.id_producto <> fd1.id_producto
JOIN productos p2 ON fd2.id_producto = p2.id_producto
WHERE p1.id_producto = 100  -- Producto específico
GROUP BY p1.id_producto, p2.id_producto
ORDER BY veces_juntos DESC
LIMIT 5;
```

## Buenas Prácticas

Aquí tienes un resumen de consejos prácticos para trabajar con subconsultas:

- **Usa EXISTS en lugar de IN cuando sea posible**: EXISTS usa cortocircuito y es más eficiente con subconsultas grandes. Además, maneja NULLs correctamente.
- **Nombra las subconsultas** (derived tables) con alias descriptivos: en lugar de `AS t`, usa `AS ventas_por_dia` o `AS ranked`. Tu yo del futuro te lo agradecerá.
- **Evita subconsultas correlacionadas cuando se puedan reescribir con JOIN**: aunque son intuitivas, suelen ser más lentas porque se ejecutan una vez por fila. Un JOIN suele ser más eficiente.
- **Cuidado con NULL en NOT IN**: es el error más común con subconsultas. Siempre prefiere NOT EXISTS si hay posibilidad de NULLs.
- **Limita subconsultas anidadas**: más de 3 niveles de anidamiento es una señal de que algo está mal. Probablemente puedas simplificar la consulta o usar una CTE (WITH).

El ejemplo que sigue muestra una subconsulta que se puede reescribir con un CROSS JOIN. La versión con CROSS JOIN calcula el promedio una sola vez (no para cada fila) y por eso suele ser más eficiente.

```sql
-- MAL: Subconsulta evitable
SELECT p.*
FROM productos p
WHERE p.precio_venta > (SELECT AVG(precio_venta) FROM productos);

-- BIEN: Puede ser más eficiente con JOIN
SELECT p.*, t.promedio
FROM productos p
CROSS JOIN (SELECT AVG(precio_venta) AS promedio FROM productos) t
WHERE p.precio_venta > t.promedio;
```

---

## Anterior: [04 Agrupacion Y Agregacion](../02-fundamentos/04-agrupacion-y-agregacion.md)
## Siguiente: [06 Insert Update Delete](../02-fundamentos/06-insert-update-delete.md)

[Volver al índice](../README.md)
