# 2.4 Agrupación y Agregación

## Funciones de Agregación

Hasta ahora, cada fila de tu consulta correspondía a una fila de la tabla (o a una combinación de filas vía JOIN). Pero muchas preguntas de negocio no son sobre filas individuales, sino sobre **conjuntos**: "¿cuántos clientes tenemos?" (no te importa cada cliente, solo el total), "¿cuál es el promedio de venta?" (no te importa cada factura, solo el número resumen), "¿cuál es el producto más caro?" (solo te importa el valor máximo).

Las **funciones de agregación** toman muchas filas, las procesan, y devuelven **un solo resultado**. Son como una calculadora que recibe una columna entera y aplica una operación: contar, sumar, promediar, encontrar el mínimo o el máximo. Estas funciones ignoran los valores `NULL` por defecto (excepto `COUNT(*)`), lo cual es importante de recordar.

### Funciones Principales

```sql
COUNT()  → Cuenta filas
SUM()    → Suma valores
AVG()    → Promedio
MIN()    → Valor mínimo
MAX()    → Valor máximo
```

### COUNT

`COUNT()` cuenta filas. Pero hay matices importantes. `COUNT(*)` cuenta **todas** las filas, sin importar si tienen valores NULL o no. Es como decir "¿cuántas filas hay en total?". `COUNT(columna)` cuenta solo las filas donde esa columna **no es NULL**. Si tienes 100 clientes pero solo 80 tienen email registrado, `COUNT(email)` devolverá 80, no 100. `COUNT(DISTINCT columna)` cuenta los valores únicos no NULL: "¿cuántas ciudades diferentes tienen nuestros clientes?"

Usa `COUNT(*)` cuando quieras saber el total de registros que cumplen una condición. Usa `COUNT(columna)` cuando quieras saber cuántos registros tienen un valor en ese campo. Usa `COUNT(DISTINCT columna)` cuando quieras saber cuántos valores diferentes existen.

```sql
SELECT COUNT(*) AS total_clientes FROM clientes;

SELECT COUNT(email) AS clientes_con_email FROM clientes;

SELECT COUNT(DISTINCT ciudad) AS ciudades_distintas FROM clientes;

SELECT 
    COUNT(*) AS total,
    SUM(CASE WHEN activo = TRUE THEN 1 ELSE 0 END) AS activos,
    SUM(CASE WHEN activo = FALSE THEN 1 ELSE 0 END) AS inactivos
FROM clientes;
```

### SUM

`SUM()` suma todos los valores de una columna numérica. Es la función que necesitas cuando alguien pregunta "¿cuánto vendimos en total?" o "¿cuál es la suma de los saldos pendientes?".

Es importante saber que `SUM()` ignora los valores NULL. Si una factura tiene `total = NULL` (lo que no debería pasar, pero puede), esa fila simplemente no se suma. Si todos los valores fueran NULL, `SUM()` devolvería NULL, no cero. Por eso es común combinar `SUM()` con `COALESCE()` o con `CASE` para manejar estos casos.

Puedes usar `SUM()` con `CASE` para hacer sumas condicionales: "suma los totales de las facturas activas por un lado, y de las canceladas por otro". Esto es más eficiente que hacer dos consultas separadas.

```sql
SELECT SUM(total) AS ventas_totales FROM facturas 
WHERE MONTH(fecha_emision) = MONTH(CURRENT_DATE)
  AND YEAR(fecha_emision) = YEAR(CURRENT_DATE);

SELECT 
    SUM(CASE WHEN estado = 'activa' THEN total ELSE 0 END) AS ventas_activas,
    SUM(CASE WHEN estado = 'cancelada' THEN total ELSE 0 END) AS ventas_canceladas
FROM facturas;
```

### AVG

`AVG()` calcula el promedio (la media aritmética) de una columna numérica. Es la suma de todos los valores dividida entre la cantidad de valores. Es la función para responder "¿cuál es el ticket promedio?" o "¿cuál es el precio promedio de los productos?".

Al igual que `SUM()`, `AVG()` ignora NULL. Si tienes 10 productos y uno tiene `precio_venta = NULL`, el promedio se calcula sobre los 9 productos restantes. Si lo que quieres es tratar NULL como 0 en el promedio, debes usar `COALESCE(columna, 0)` dentro del `AVG()`.

```sql
SELECT AVG(total) AS ticket_promedio FROM facturas WHERE estado = 'activa';

SELECT AVG(p.precio_venta) AS precio_promedio FROM productos p;
```

### MIN y MAX

`MIN()` y `MAX()` encuentran, respectivamente, el valor más pequeño y el más grande de una columna. Funcionan con números, fechas y texto. Con fechas, `MIN(fecha)` te da la fecha más antigua y `MAX(fecha)` la más reciente. Con texto, el orden es alfabético.

Es muy común usar `MIN` y `MAX` juntos para obtener el rango de valores: "la venta más baja fue de $50, la más alta de $50,000, y el promedio fue de $1,200". También se usan para determinar períodos: "la primera venta fue el 1 de enero, la última fue el 31 de diciembre".

```sql
SELECT 
    MIN(total) AS venta_minima,
    MAX(total) AS venta_maxima,
    AVG(total) AS venta_promedio,
    MIN(fecha_emision) AS primera_venta,
    MAX(fecha_emision) AS ultima_venta
FROM facturas WHERE estado = 'activa';
```

## GROUP BY - Agrupar Filas

Aquí llegamos a uno de los conceptos más importantes de SQL. Las funciones de agregación (`COUNT`, `SUM`, `AVG`, etc.) por sí solas operan sobre **todas** las filas y devuelven una sola fila de resultado. Pero casi siempre quieres algo como: "dame el total de ventas **por vendedor**" o "el promedio de precio **por categoría**". Es decir, quieres aplicar la agregación **por grupos**.

Ahí entra `GROUP BY`. Piensa en ello como: "toma todas las filas, sepáralas en grupos según el valor de esta columna, y luego aplica la función de agregación a cada grupo por separado". Si agrupas por `id_categoria`, todas las filas con `id_categoria = 1` forman un grupo, las de `id_categoria = 2` otro, etc. Luego `COUNT(*)` te dice cuántos productos hay en cada grupo.

La regla de oro: **Toda columna que aparezca en el `SELECT` y no sea una función de agregación debe estar en el `GROUP BY`**. Si pones `SELECT categoria, SUM(total)` sin `GROUP BY categoria`, SQL te marcará error porque no sabe a qué grupo pertenece cada valor.

```sql
SELECT columna_agrupada, funcion_agregacion()
FROM tabla
GROUP BY columna_agrupada;
```

### Ejemplos Básicos

Estos tres ejemplos muestran el poder de `GROUP BY`. En el primero, agrupas ventas por vendedor para ver quién vende más. En el segundo, agrupas productos por categoría para ver qué categoría genera más ingresos. En el tercero, agrupas por múltiples niveles (año, mes, vendedor) para un reporte temporal detallado. Nota que en el tercer ejemplo, todas las columnas del `SELECT` que no son agregaciones (año, mes, `v.id_vendedor`) están en el `GROUP BY`.

```sql
SELECT 
    v.nombre_vendedor,
    COUNT(f.id_factura) AS facturas,
    SUM(f.total) AS total_vendido
FROM facturas f
JOIN vendedores v ON f.id_vendedor = v.id_vendedor
GROUP BY v.id_vendedor;

SELECT 
    c.nombre AS categoria,
    COUNT(DISTINCT p.id_producto) AS productos_vendidos,
    SUM(fd.cantidad) AS unidades,
    SUM(fd.total) AS ingresos
FROM categorias c
JOIN productos p ON c.id_categoria = p.id_categoria
JOIN facturas_detalle fd ON p.id_producto = fd.id_producto
GROUP BY c.id_categoria;

SELECT 
    YEAR(f.fecha_emision) AS año,
    MONTH(f.fecha_emision) AS mes,
    v.nombre_vendedor,
    COUNT(*) AS facturas,
    SUM(f.total) AS total
FROM facturas f
JOIN vendedores v ON f.id_vendedor = v.id_vendedor
GROUP BY año, mes, v.id_vendedor
ORDER BY año, mes, total DESC;
```

### GROUP BY con Expresiones

No solo puedes agrupar por columnas existentes, sino también por el resultado de expresiones. Por ejemplo, puedes agrupar por año-mes usando `DATE_FORMAT()` para tener un reporte mensual. O puedes agrupar por rangos de precio usando un `CASE`: defines categorías como "Económico", "Medio", "Premium" y luego cuentas cuántos productos caen en cada rango. Esto es extremadamente útil para segmentación y análisis.

```sql
SELECT 
    DATE_FORMAT(fecha_emision, '%Y-%m') AS mes,
    COUNT(*) AS facturas,
    SUM(total) AS total
FROM facturas
GROUP BY DATE_FORMAT(fecha_emision, '%Y-%m')
ORDER BY mes;

SELECT 
    CASE 
        WHEN precio_venta < 100 THEN 'Económico'
        WHEN precio_venta BETWEEN 100 AND 1000 THEN 'Medio'
        ELSE 'Premium'
    END AS rango_precio,
    COUNT(*) AS productos,
    AVG(precio_venta) AS precio_promedio
FROM productos
GROUP BY rango_precio;
```

## HAVING - Filtrar Grupos

Ya conoces `WHERE` para filtrar filas individuales. Pero ¿cómo filtras después de agrupar? Por ejemplo: "quiero solo los vendedores que hayan hecho más de 50 ventas". El problema es que `COUNT(*) > 50` no puede ir en el `WHERE` porque `WHERE` se evalúa **antes** de agrupar, y en ese momento SQL aún no sabe cuántas ventas tiene cada vendedor.

Para eso existe `HAVING`. Si `WHERE` filtra filas, `HAVING` filtra **grupos**. Se escribe después de `GROUP BY` y puede usar funciones de agregación como `COUNT()`, `SUM()`, `AVG()`, etc. La secuencia lógica es:

1. `WHERE` filtra las filas individuales
2. `GROUP BY` agrupa las filas restantes
3. `HAVING` filtra los grupos resultantes

```sql
SELECT columna_agrupada, funcion_agregacion()
FROM tabla
GROUP BY columna_agrupada
HAVING condicion_del_grupo;
```

### Ejemplos

En el primer ejemplo, agrupas ventas por vendedor y luego con `HAVING COUNT(*) > 50` te quedas solo con los vendedores que superan las 50 ventas. En el segundo, te quedas con categorías cuyos ingresos superan $100,000. En el tercero, buscas clientes que hayan comprado más de 10 veces pero con un ticket promedio bajo (menos de $500) — clientes frecuentes pero de bajo gasto, ideales para campañas de upselling.

```sql
SELECT 
    v.nombre_vendedor,
    COUNT(*) AS ventas,
    SUM(f.total) AS total
FROM facturas f
JOIN vendedores v ON f.id_vendedor = v.id_vendedor
GROUP BY v.id_vendedor
HAVING COUNT(*) > 50;

SELECT 
    c.nombre,
    SUM(fd.total) AS ingresos,
    COUNT(DISTINCT fd.id_producto) AS productos_vendidos
FROM categorias c
JOIN productos p ON c.id_categoria = p.id_categoria
JOIN facturas_detalle fd ON p.id_producto = fd.id_producto
JOIN facturas f ON fd.id_factura = f.id_factura AND f.estado = 'activa'
GROUP BY c.id_categoria
HAVING SUM(fd.total) > 100000;

SELECT 
    c.nombre,
    COUNT(f.id_factura) AS compras,
    AVG(f.total) AS ticket_promedio,
    SUM(f.total) AS total_gastado
FROM clientes c
JOIN facturas f ON c.id_cliente = f.id_cliente AND f.estado = 'activa'
GROUP BY c.id_cliente
HAVING COUNT(f.id_factura) > 10 AND AVG(f.total) < 500;
```

## Diferencia WHERE vs HAVING

Esta es una de las confusiones más comunes entre principiantes. La diferencia clave es el **momento** en que se aplican:

- `WHERE` se aplica **antes** de agrupar, a las filas individuales. No puede usar funciones de agregación (`COUNT`, `SUM`, etc.) porque los grupos aún no existen.
- `HAVING` se aplica **después** de agrupar, a los grupos. Puede usar funciones de agregación.

En la práctica, siempre que puedas filtrar con `WHERE`, hazlo. Es más eficiente porque reduces la cantidad de datos que luego se agrupan. Usa `HAVING` solo para condiciones que requieran funciones de agregación.

| WHERE | HAVING |
|-------|--------|
| Filtra FILAS antes de agrupar | Filtra GRUPOS después de agrupar |
| No puede usar funciones de agregación | Puede usar funciones de agregación |
| Se aplica a registros individuales | Se aplica a grupos |
| Mayor performance (reduce datos antes) | Menor performance |

```sql
SELECT 
    v.nombre_vendedor,
    COUNT(*) AS ventas
FROM facturas f
JOIN vendedores v ON f.id_vendedor = v.id_vendedor
WHERE f.fecha_emision >= '2024-01-01'
GROUP BY v.id_vendedor;

SELECT 
    v.nombre_vendedor,
    COUNT(*) AS ventas
FROM facturas f
JOIN vendedores v ON f.id_vendedor = v.id_vendedor
GROUP BY v.id_vendedor
HAVING COUNT(*) > 30;
```

## GROUP_CONCAT (MySQL) / STRING_AGG (PostgreSQL)

A veces no quieres contar o sumar, sino **listar** los valores que forman parte de cada grupo. Por ejemplo, para cada cliente, quieres una lista de los productos que ha comprado. `GROUP_CONCAT()` (en MySQL) y `STRING_AGG()` (en PostgreSQL) hacen exactamente eso: toman los valores de una columna dentro de cada grupo y los concatenan en una sola cadena, separados por comas (o el separador que elijas). Puedes incluso ordenarlos dentro de la concatenación.

Imagina que es como si para cada cliente, coleccionaras todos los nombres de producto que ha comprado, los escribieras en una sola línea separados por comas, y el resultado fuera algo como "Laptop, Mouse, Teclado".

```sql
SELECT 
    c.nombre,
    GROUP_CONCAT(p.nombre_producto ORDER BY p.nombre_producto SEPARATOR ', ') AS productos_comprados
FROM clientes c
JOIN facturas f ON c.id_cliente = f.id_cliente
JOIN facturas_detalle fd ON f.id_factura = fd.id_factura
JOIN productos p ON fd.id_producto = p.id_producto
GROUP BY c.id_cliente;

SELECT 
    c.nombre,
    STRING_AGG(p.nombre_producto, ', ' ORDER BY p.nombre_producto) AS productos_comprados
FROM clientes c
JOIN facturas f ON c.id_cliente = f.id_cliente
JOIN facturas_detalle fd ON f.id_factura = fd.id_factura
JOIN productos p ON fd.id_producto = p.id_producto
GROUP BY c.id_cliente;
```

## ROLLUP - Subtotales Jerárquicos

`ROLLUP` es una extensión de `GROUP BY` que genera **subtotales en múltiples niveles**. Si agrupas por `categoria, producto`, el `GROUP BY` normal te da el total por cada producto dentro de cada categoría. `ROLLUP` además agrega una fila con el subtotal por cada categoría, y al final una fila con el gran total.

Es como si pidieras un reporte, luego "con subtotales", y luego "con gran total". El `COALESCE` se usa para mostrar textos descriptivos ("Subtotal", "Todas") en lugar de NULL en las columnas de agrupación.

```sql
SELECT 
    COALESCE(c.nombre, 'Todas') AS categoria,
    COALESCE(p.nombre_producto, 'Subtotal') AS producto,
    SUM(fd.cantidad) AS unidades_vendidas
FROM productos p
JOIN categorias c ON p.id_categoria = c.id_categoria
JOIN facturas_detalle fd ON p.id_producto = fd.id_producto
GROUP BY c.nombre, p.nombre_producto WITH ROLLUP;

SELECT 
    c.nombre AS categoria,
    p.nombre_producto AS producto,
    SUM(fd.cantidad) AS unidades_vendidas
FROM productos p
JOIN categorias c ON p.id_categoria = c.id_categoria
JOIN facturas_detalle fd ON p.id_producto = fd.id_producto
GROUP BY ROLLUP (c.nombre, p.nombre_producto);
```

## CUBE - Todas las Combinaciones

`CUBE` va un paso más allá que `ROLLUP`. Mientras que `ROLLUP` genera subtotales en una jerarquía (categoría → producto), `CUBE` genera subtotales para **todas las combinaciones posibles** de las columnas de agrupación. Si agrupas por `categoria` y `almacen`, `CUBE` te da: total por cada categoría, total por cada almacén, total por cada combinación específica, y el gran total. Es útil para análisis multidimensionales.

```sql
SELECT 
    COALESCE(c.nombre, 'Todas') AS categoria,
    COALESCE(a.nombre, 'Todos') AS almacen,
    SUM(s.stock_actual) AS stock_total
FROM productos p
JOIN categorias c ON p.id_categoria = c.id_categoria
JOIN almacenes_stock s ON p.id_producto = s.id_producto
JOIN almacenes a ON s.id_almacen = a.id_almacen
GROUP BY CUBE (c.nombre, a.nombre);
```

## GROUPING SETS - Agrupaciones Específicas

`GROUPING SETS` te da control granular sobre qué agrupaciones quieres. En lugar de todos los subtotales (CUBE) o los jerárquicos (ROLLUP), defines exactamente los conjuntos de agrupación que te interesan. Por ejemplo, si quieres solo el total por categoría y el total por vendedor (pero no la combinación de ambos), `GROUPING SETS` te permite especificarlo.

```sql
SELECT 
    COALESCE(c.nombre, 'Todos') AS categoria,
    COALESCE(v.nombre_vendedor, 'Todos') AS vendedor,
    SUM(f.total) AS ventas
FROM facturas f
JOIN vendedores v ON f.id_vendedor = v.id_vendedor
JOIN clientes c ON f.id_cliente = c.id_cliente
GROUP BY GROUPING SETS (
    (c.nombre),
    (v.nombre_vendedor),
    ()
);
```

## 📊 Dashboard Analítico con Agregaciones

Este es un dashboard completo de métricas de ventas mes a mes. Combina múltiples funciones de agregación: `COUNT(DISTINCT)` para contar elementos únicos (facturas, clientes, productos), `SUM()` para ingresos y descuentos, `AVG()` para ticket promedio. También usa `NULLIF()` para evitar división entre cero en el precio promedio unitario. La consulta agrupa por mes y ordena del más reciente al más antiguo. Este es exactamente el tipo de consulta que ves en sistemas de business intelligence.

El segundo bloque es un análisis ABC (Principio de Pareto 80/20) usando **funciones de ventana** (`SUM() OVER`). Clasifica productos en A (el 80% superior de ingresos), B (siguiente 15%), y C (último 5%). Aunque las ventanas son un tema avanzado, este ejemplo muestra hacia dónde te llevarán las agregaciones cuando las domines.

```sql
SELECT 
    DATE_FORMAT(f.fecha_emision, '%Y-%m') AS periodo,
    COUNT(DISTINCT f.id_factura) AS facturas_emitidas,
    COUNT(DISTINCT f.id_cliente) AS clientes_activos,
    SUM(f.total) AS ingresos_brutos,
    SUM(f.descuento) AS descuentos_otorgados,
    SUM(f.total) - SUM(f.descuento) AS ingresos_netos,
    ROUND(AVG(f.total), 2) AS ticket_promedio,
    COUNT(DISTINCT fd.id_producto) AS productos_vendidos,
    SUM(fd.cantidad) AS unidades_vendidas,
    SUM(fd.cantidad * fd.precio_unitario - fd.descuento) / NULLIF(SUM(fd.cantidad), 0) AS precio_promedio_unitario
FROM facturas f
LEFT JOIN facturas_detalle fd ON f.id_factura = fd.id_factura
WHERE f.estado = 'activa'
    AND f.fecha_emision >= DATE_SUB(CURRENT_DATE, INTERVAL 12 MONTH)
GROUP BY DATE_FORMAT(f.fecha_emision, '%Y-%m')
ORDER BY periodo DESC;

SELECT 
    p.nombre_producto,
    SUM(fd.total) AS ingresos,
    SUM(SUM(fd.total)) OVER (ORDER BY SUM(fd.total) DESC) / SUM(SUM(fd.total)) OVER () AS porcentaje_acumulado,
    CASE 
        WHEN SUM(SUM(fd.total)) OVER (ORDER BY SUM(fd.total) DESC) / SUM(SUM(fd.total)) OVER () <= 0.8 THEN 'A'
        WHEN SUM(SUM(fd.total)) OVER (ORDER BY SUM(fd.total) DESC) / SUM(SUM(fd.total)) OVER () <= 0.95 THEN 'B'
        ELSE 'C'
    END AS clasificacion
FROM productos p
JOIN facturas_detalle fd ON p.id_producto = fd.id_producto
JOIN facturas f ON fd.id_factura = f.id_factura AND f.estado = 'activa'
GROUP BY p.id_producto
ORDER BY ingresos DESC;
```

### Análisis de Estacionalidad

¿Sabes qué días de la semana y a qué horas se vende más? Esta consulta te lo dice. Agrupa las ventas por día de la semana y hora, mostrando transacciones, ingresos y ticket promedio. Es útil para decidir horarios de personal, campañas de marketing en horas pico, o para identificar patrones estacionales. El `ORDER BY` usa `FIELD()` para ordenar los días de lunes a domingo en lugar de alfabéticamente.

```sql
SELECT 
    DAYNAME(fecha_emision) AS dia_semana,
    HOUR(fecha_emision) AS hora,
    COUNT(*) AS transacciones,
    SUM(total) AS ingresos,
    AVG(total) AS ticket_promedio
FROM facturas
WHERE YEAR(fecha_emision) = 2024
GROUP BY dia_semana, hora
ORDER BY 
    FIELD(dia_semana, 'Monday', 'Tuesday', 'Wednesday', 'Thursday', 'Friday', 'Saturday', 'Sunday'),
    hora;
```

### Rendimiento de Vendedores

Para una gerencia de ventas, esta consulta es oro puro. Calcula por cada vendedor: cuántas ventas hizo, a cuántos clientes distintos atendió, el monto total vendido, el ticket promedio, la comisión generada, y un ranking (usando `RANK() OVER`). Todo agrupado por vendedor y filtrado al último mes. El ranking ordena automáticamente del mejor al peor desempeño.

```sql
SELECT 
    v.nombre_vendedor,
    COUNT(DISTINCT f.id_factura) AS ventas,
    COUNT(DISTINCT f.id_cliente) AS clientes_atendidos,
    SUM(f.total) AS monto_vendido,
    ROUND(SUM(f.total) / NULLIF(COUNT(DISTINCT f.id_factura), 0), 2) AS ticket_promedio,
    ROUND(COMISION_PORCENTAJE * SUM(f.total) / 100, 2) AS comision_generada,
    RANK() OVER (ORDER BY SUM(f.total) DESC) AS ranking
FROM vendedores v
JOIN facturas f ON v.id_vendedor = f.id_vendedor
WHERE f.estado = 'activa'
    AND f.fecha_emision >= DATE_SUB(CURRENT_DATE, INTERVAL 1 MONTH)
GROUP BY v.id_vendedor
ORDER BY ranking;
```

## Ejercicios Prácticos

Estos tres ejercicios integran todo lo aprendido: JOINs, funciones de agregación, `GROUP BY`, `HAVING`, y manejo de fechas. El primero analiza ventas por día de la semana. El segundo identifica productos problemáticos con muchas devoluciones. El tercero encuentra clientes inactivos (más de 6 meses sin comprar), usando `LEFT JOIN` para incluir también a los que nunca compraron.

Tómate el tiempo para leer cada consulta y entender qué hace cada parte. Son patrones que repetirás constantemente en tu trabajo con SQL.

```sql
SELECT 
    DAYNAME(fecha_emision) AS dia,
    COUNT(*) AS ventas,
    SUM(total) AS ingresos
FROM facturas
GROUP BY DAYNAME(fecha_emision)
ORDER BY ingresos DESC;

SELECT 
    p.nombre_producto,
    COUNT(*) AS veces_devuelto,
    SUM(fd.cantidad) AS unidades_devueltas
FROM facturas f
JOIN facturas_detalle fd ON f.id_factura = fd.id_factura
JOIN productos p ON fd.id_producto = p.id_producto
WHERE f.estado IN ('devuelta_parcial', 'devuelta_total')
GROUP BY p.id_producto
HAVING COUNT(*) > 0
ORDER BY veces_devuelto DESC;

SELECT 
    c.nombre,
    MAX(f.fecha_emision) AS ultima_compra,
    DATEDIFF(CURRENT_DATE, MAX(f.fecha_emision)) AS dias_inactivo
FROM clientes c
LEFT JOIN facturas f ON c.id_cliente = f.id_cliente AND f.estado = 'activa'
GROUP BY c.id_cliente
HAVING dias_inactivo > 180 OR MAX(f.fecha_emision) IS NULL
ORDER BY dias_inactivo DESC;
```

---

## Anterior: [03 Join Y Relaciones](../02-fundamentos/03-join-y-relaciones.md)
## Siguiente: [05 Subconsultas](../02-fundamentos/05-subconsultas.md)

[Volver al índice](../README.md)
