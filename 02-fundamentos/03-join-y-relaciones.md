# 2.3 JOIN y Relaciones

## Introducción a JOIN

Hasta ahora has trabajado con una sola tabla a la vez: clientes, productos o facturas. Pero en una base de datos real, la información está **distribuida** en múltiples tablas por una razón muy importante: la **normalización**. En lugar de repetir el nombre del cliente en cada factura, guardas el cliente una vez en la tabla `clientes` y en la factura solo pones su ID. Esto ahorra espacio y evita inconsistencias (si el cliente cambia de nombre, solo cambia en un lugar).

El problema es que, para responder preguntas de negocio, necesitas juntar esa información dispersa. Por ejemplo, para saber "¿qué productos compró Juan Pérez?" necesitas info de `clientes`, `facturas`, `facturas_detalle` y `productos`. Ahí es donde entra `JOIN`: la instrucción que te permite combinar filas de dos o más tablas basándose en una relación entre ellas.

Piensa en `JOIN` como conectar piezas de un rompecabezas. Cada tabla tiene una pieza de la información, y el `JOIN` las une por una columna común (generalmente un ID). La columna común es la "orilla" donde las piezas encajan.

### Tipos de JOIN

Hay varios tipos de JOIN, y cada uno responde a una pregunta diferente sobre cómo quieres combinar los datos. La elección depende de si quieres solo las coincidencias, o también los datos de una tabla aunque no tengan correspondencia en la otra.

```sql
INNER JOIN   → Solo registros que coinciden en ambas tablas
LEFT JOIN    → Todos los de la izquierda + coincidencias de la derecha
RIGHT JOIN   → Todos los de la derecha + coincidencias de la izquierda
FULL JOIN    → Todos los registros de ambas tablas
CROSS JOIN   → Producto cartesiano
SELF JOIN    → Una tabla unida consigo misma
```

## INNER JOIN

El `INNER JOIN` es el tipo de JOIN más común. Retorna **solo las filas que tienen correspondencia en ambas tablas**. Si un cliente está registrado en la tabla `clientes` pero no tiene ninguna factura en la tabla `facturas`, ese cliente no aparecerá en el resultado. De la misma manera, si hay una factura cuyo `id_cliente` no existe en la tabla `clientes` (lo que sería un error de integridad), tampoco aparecería.

Imagina que tienes dos círculos que se intersectan. El `INNER JOIN` es solo la parte donde se traslapan. Es como decir: "dame los clientes que SÍ han comprado" o "dame los productos que SÍ se han vendido". La sintaxis típica es: pones la primera tabla después de `FROM`, luego `INNER JOIN` seguido de la segunda tabla, y luego `ON` seguido de la condición que relaciona ambas tablas (generalmente una igualdad de IDs).

Presta atención también a los **alias**: escribir `FROM clientes c` te permite después referirte a la tabla como `c` en lugar de escribir `clientes` completo. Esto no solo ahorra tipeo, sino que hace las consultas mucho más legibles, especialmente cuando tienes 4 o 5 tablas unidas.

```sql
SELECT c.nombre, f.folio, f.total
FROM clientes c
INNER JOIN facturas f ON c.id_cliente = f.id_cliente;

SELECT c.nombre, f.folio, f.total
FROM clientes c
JOIN facturas f ON c.id_cliente = f.id_cliente;
```

### Múltiples INNER JOIN

En la vida real, rara vez unes solo dos tablas. Una consulta típica puede unir 4, 5 o más tablas. Por ejemplo, para obtener el detalle completo de una factura (folio, cliente, productos, cantidades, precios), necesitas unir: `facturas` con `clientes`, `facturas` con `facturas_detalle`, y `facturas_detalle` con `productos`. Cada JOIN agrega más columnas al resultado. Es como una cadena: empiezas con una tabla y vas agregando las demás una por una, especificando cómo se relaciona cada nueva tabla con alguna de las que ya tienes.

```sql
SELECT 
    f.folio,
    c.nombre AS cliente,
    p.nombre_producto,
    fd.cantidad,
    fd.precio_unitario,
    fd.total AS importe
FROM facturas f
JOIN clientes c ON f.id_cliente = c.id_cliente
JOIN facturas_detalle fd ON f.id_factura = fd.id_factura
JOIN productos p ON fd.id_producto = p.id_producto
WHERE f.fecha_emision >= '2024-01-01'
ORDER BY f.folio, fd.id_detalle;
```

## LEFT JOIN (LEFT OUTER JOIN)

El `LEFT JOIN` es para cuando quieres **todos** los registros de la tabla izquierda (la que va después de `FROM`), tengan o no correspondencia en la tabla derecha (la que va después de `LEFT JOIN`). Si no hay correspondencia, las columnas de la tabla derecha aparecen como `NULL`.

¿Cuándo usarías esto? Por ejemplo: "dame TODOS los clientes, incluso los que nunca han comprado." Con un `INNER JOIN`, los clientes sin facturas se pierden. Con `LEFT JOIN`, aparecen, y en las columnas de facturas (folio, total) verás `NULL`. Esto te permite identificar clientes inactivos.

Piensa en la tabla izquierda como "la protagonista" y la tabla derecha como "información adicional que puede o no existir". Es como una libreta de direcciones: tienes a todas tus personas (tabla izquierda), y algunas de ellas tienen teléfono (tabla derecha). Quieres ver la lista completa, con teléfono si existe, sin él si no.

```sql
SELECT 
    c.nombre AS cliente,
    COUNT(f.id_factura) AS numero_compras,
    COALESCE(SUM(f.total), 0) AS total_gastado
FROM clientes c
LEFT JOIN facturas f ON c.id_cliente = f.id_cliente
    AND f.estado = 'activa'
GROUP BY c.id_cliente
ORDER BY total_gastado DESC;
```

### LEFT JOIN para encontrar registros sin correspondencia

Un patrón muy común con `LEFT JOIN` es usarlo para **encontrar lo que NO existe**. La idea es unir las tablas y luego, en el `WHERE`, buscar los casos donde la tabla derecha es `NULL`. Eso significa que no hubo correspondencia. Por ejemplo: productos que nunca se han vendido, proveedores sin órdenes de compra, clientes que nunca han comprado. Es la forma más común de responder preguntas como "¿qué nos falta?".

```sql
SELECT p.*
FROM productos p
LEFT JOIN facturas_detalle fd ON p.id_producto = fd.id_producto
WHERE fd.id_detalle IS NULL;

SELECT pr.*
FROM proveedores pr
LEFT JOIN ordenes_compra oc ON pr.id_proveedor = oc.id_proveedor
WHERE oc.id_orden IS NULL;

SELECT c.*
FROM clientes c
LEFT JOIN facturas f ON c.id_cliente = f.id_cliente
WHERE f.id_factura IS NULL;
```

## RIGHT JOIN (RIGHT OUTER JOIN)

El `RIGHT JOIN` es el espejo del `LEFT JOIN`: toma **todos** los registros de la tabla derecha (después de `RIGHT JOIN`) y solo las coincidencias de la izquierda. En la práctica, el `RIGHT JOIN` se usa mucho menos porque la mayoría de los programadores prefieren `LEFT JOIN`, que es más intuitivo (de izquierda a derecha, como leemos). De hecho, cualquier `RIGHT JOIN` se puede convertir a `LEFT JOIN` simplemente invirtiendo el orden de las tablas.

```sql
SELECT p.nombre_producto, fd.cantidad
FROM facturas_detalle fd
RIGHT JOIN productos p ON fd.id_producto = p.id_producto;
```

## FULL JOIN (FULL OUTER JOIN)

El `FULL JOIN` es la combinación de LEFT y RIGHT: te da **todos** los registros de ambas tablas, haya o no coincidencia. Es como poner los dos círculos completos del diagrama de Venn, incluyendo lo que está fuera de la intersección. Es útil cuando quieres ver TODO: los clientes sin facturas Y las facturas sin clientes (si las hay).

Una limitación importante: MySQL/MariaDB no soporta `FULL JOIN` directamente. PostgreSQL sí.

```sql
SELECT c.nombre, f.folio, f.total
FROM clientes c
FULL JOIN facturas f ON c.id_cliente = f.id_cliente;
```

### Simular FULL JOIN en MySQL

Si usas MySQL y necesitas un `FULL JOIN`, puedes simularlo con `UNION` de un `LEFT JOIN` y un `RIGHT JOIN`. `UNION` combina los resultados de dos consultas eliminando duplicados. El `LEFT JOIN` te da todos los clientes (con o sin facturas) y el `RIGHT JOIN` te da todas las facturas (con o sin cliente). Al unirlos, obtienes el equivalente a un `FULL JOIN`.

```sql
SELECT c.nombre, f.folio, f.total
FROM clientes c
LEFT JOIN facturas f ON c.id_cliente = f.id_cliente

UNION

SELECT c.nombre, f.folio, f.total
FROM clientes c
RIGHT JOIN facturas f ON c.id_cliente = f.id_cliente;
```

## CROSS JOIN (Producto Cartesiano)

El `CROSS JOIN` es especial: no usa una condición `ON`. En lugar de emparejar filas relacionadas, combina **cada fila de la primera tabla con cada fila de la segunda tabla**. Si tienes 10 productos y 5 almacenes, el resultado tendrá 50 filas (10 x 5). Esto se llama **producto cartesiano**.

¿Cuándo usarías esto? Cuando necesitas generar combinaciones de todos los elementos de dos conjuntos. Por ejemplo, para crear un reporte de stock donde quieres ver cada producto en cada almacén, incluso si algunos productos no tienen registro en algunos almacenes. El `CROSS JOIN` genera la "plantilla" de todas las combinaciones posibles, y luego haces un `LEFT JOIN` para traer los datos reales.

```sql
SELECT 
    p.nombre_producto,
    a.nombre AS almacen
FROM productos p
CROSS JOIN almacenes a;

SELECT p.nombre_producto, a.nombre
FROM productos p, almacenes a;
```

### Uso práctico: Generar reportes por defecto

Este es el caso práctico más común de `CROSS JOIN`: generar una matriz completa de todas las combinaciones posibles para después llenarla con datos reales. Primero cruzas productos con almacenes para tener todas las combinaciones, luego haces un `LEFT JOIN` con los movimientos de inventario para traer las cantidades reales, y usas `COALESCE` para mostrar 0 donde no hay movimientos.

```sql
SELECT 
    p.nombre_producto,
    a.nombre AS almacen,
    COALESCE(SUM(m.cantidad), 0) AS stock_actual
FROM productos p
CROSS JOIN almacenes a
LEFT JOIN inventario_movimientos m ON p.id_producto = m.id_producto 
    AND a.id_almacen = m.id_almacen
GROUP BY p.id_producto, a.id_almacen;
```

## SELF JOIN

Un `SELF JOIN` es cuando una tabla se une **consigo misma**. Esto suena extraño al principio, pero tiene usos muy prácticos. Imagina una tabla `categorias` donde cada categoría puede tener una categoría padre (por ejemplo: "Lácteos" es hija de "Alimentos", que es hija de "Comestibles"). La columna `id_categoria_padre` apunta a otra fila de la misma tabla.

Para obtener un listado que muestre cada categoría junto con el nombre de su categoría padre, necesitas unir la tabla consigo misma: la tratas como si fueran dos tablas separadas, cada una con su propio alias.

```sql
SELECT 
    c.nombre AS categoria,
    p.nombre AS categoria_padre
FROM categorias c
LEFT JOIN categorias p ON c.id_categoria_padre = p.id_categoria;

SELECT 
    p.nombre_producto AS producto,
    c.nombre_producto AS componente
FROM productos p
JOIN productos_componentes pc ON p.id_producto = pc.id_producto_padre
JOIN productos c ON pc.id_producto_componente = c.id_producto;
```

## USING vs ON

Cuando las columnas de enlace tienen exactamente el mismo nombre en ambas tablas (por ejemplo, ambas se llaman `id_cliente`), puedes usar `USING` en lugar de `ON`. Es más limpio y corto, y evita tener que escribir los alias de tabla para la columna de unión. Sin embargo, `ON` es más explícito y funciona siempre, incluso cuando los nombres de columna difieren entre tablas.

```sql
SELECT * FROM clientes c 
JOIN facturas f ON c.id_cliente = f.id_cliente;

SELECT * FROM clientes 
JOIN facturas USING(id_cliente);
```

## NATURAL JOIN

El `NATURAL JOIN` es una forma "automática" de JOIN: SQL detecta las columnas que tienen el mismo nombre en ambas tablas y las usa para unir. Suena conveniente, pero **es peligroso**. Si las tablas tienen otras columnas con el mismo nombre que no deberían usarse para la unión (por ejemplo, ambas tienen una columna `fecha_creacion`), el JOIN las incluirá también, causando resultados incorrectos o inesperados. La recomendación general es **evitar NATURAL JOIN** y siempre especificar explícitamente la condición de unión.

```sql
SELECT * FROM clientes NATURAL JOIN facturas;
```

## JOIN con Condiciones Complejas

Las condiciones de JOIN no tienen que ser simples igualdades. Puedes agregar condiciones adicionales con `AND` dentro del `ON`. Por ejemplo, al unir facturas con clientes, puedes pedir que el cliente esté activo y tenga crédito. Esto es diferente a poner esas condiciones en el `WHERE`: las condiciones en el `ON` del JOIN se evalúan **durante** la unión, mientras que las del `WHERE` se evalúan **después**. Esto es especialmente relevante con `LEFT JOIN`, como viste en el capítulo anterior.

```sql
SELECT 
    f.folio,
    c.nombre AS cliente,
    f.total,
    p.nombre_producto,
    fd.cantidad
FROM facturas f
JOIN clientes c ON f.id_cliente = c.id_cliente
    AND c.activo = TRUE
    AND c.credito_limite > 0
JOIN facturas_detalle fd ON f.id_factura = fd.id_factura
JOIN productos p ON fd.id_producto = p.id_producto
    AND p.activo = TRUE
WHERE f.fecha_emision >= '2024-01-01'
    AND f.estado = 'activa'
    AND f.total > (
        SELECT AVG(total) FROM facturas WHERE YEAR(fecha_emision) = 2024
    );
```

## Diagrama Visual de JOINs

Una imagen vale más que mil palabras. Estos diagramas representan visualmente cada tipo de JOIN. Las áreas sombreadas muestran qué registros se incluyen. El círculo A es la tabla izquierda, el B es la tabla derecha, y la intersección es donde coinciden.

```
INNER JOIN:            LEFT JOIN:             RIGHT JOIN:
┌─────┬─────┐         ┌─────┬─────┐          ┌─────┬─────┐
│  A  │  B  │         │  A  │  B  │          │  A  │  B  │
│ ○───│───○ │         │ ○───│───○ │          │ ○───│───○ │
│  ○  │  ○  │         │  ○  │  ○  │          │  ○  │  ○  │
│     │     │         │  ○  │     │          │     │  ○  │
└─────┴─────┘         └─────┴─────┘          └─────┴─────┘

FULL JOIN:             LEFT EXCLUDING:       RIGHT EXCLUDING:
┌─────┬─────┐         ┌─────┬─────┐          ┌─────┬─────┐
│  A  │  B  │         │  A  │  B  │          │  A  │  B  │
│ ○───│───○ │         │  ○  │     │          │     │  ○  │
│  ○  │  ○  │         │  ○  │     │          │     │  ○  │
│     │  ○  │         └─────┴─────┘          └─────┴─────┘
└─────┴─────┘          (LEFT - INNER)         (RIGHT - INNER)
```

## 📊 Ejemplos del Mundo Real

### Reporte de Ventas Completo

Este es probablemente el reporte más común en cualquier negocio: el detalle de ventas en un período. Une 4 tablas (facturas, clientes, vendedores, facturas_detalle) para mostrar toda la información relevante de cada factura: a quién se le vendió, quién vendió, qué artículos, cantidades, montos, y método de pago. Nota el uso de `LEFT JOIN` para vendedores (por si alguna factura no tiene vendedor asignado) y para el detalle (por si alguna factura está sin líneas de detalle).

```sql
SELECT 
    f.folio,
    DATE_FORMAT(f.fecha_emision, '%d/%m/%Y') AS fecha,
    c.nombre AS cliente,
    c.rfc,
    v.nombre_vendedor AS vendedor,
    COUNT(fd.id_detalle) AS articulos,
    SUM(fd.cantidad) AS total_piezas,
    f.subtotal,
    f.descuento,
    f.iva,
    f.total,
    f.metodo_pago,
    f.estado
FROM facturas f
JOIN clientes c ON f.id_cliente = c.id_cliente
LEFT JOIN vendedores v ON f.id_vendedor = v.id_vendedor
LEFT JOIN facturas_detalle fd ON f.id_factura = fd.id_factura
WHERE f.fecha_emision BETWEEN '2024-01-01' AND '2024-12-31'
GROUP BY f.id_factura
ORDER BY f.fecha_emision DESC;
```

### Órdenes de Compra con Estatus

Este reporte muestra órdenes de compra activas con su proveedor, fechas y un cálculo del porcentaje recibido. Es útil para el departamento de compras: pueden ver de un vistazo qué órdenes están completas, cuáles están parcialmente recibidas y cuáles están atrasadas (cuando la fecha estimada ya pasó). La función `NULLIF` evita la división entre cero si no se ha solicitado nada.

```sql
SELECT 
    oc.folio AS orden,
    p.nombre_comercial AS proveedor,
    oc.fecha_emision,
    oc.fecha_estimada_entrega,
    DATEDIFF(oc.fecha_estimada_entrega, CURRENT_DATE) AS dias_restantes,
    oc.estado,
    COUNT(ocd.id_detalle) AS productos_solicitados,
    SUM(ocd.cantidad_recibida) / NULLIF(SUM(ocd.cantidad_solicitada), 0) * 100 AS porcentaje_recibido
FROM ordenes_compra oc
JOIN proveedores p ON oc.id_proveedor = p.id_proveedor
LEFT JOIN ordenes_compra_detalle ocd ON oc.id_orden = ocd.id_orden
WHERE oc.estado NOT IN ('borrador', 'cancelada')
GROUP BY oc.id_orden
ORDER BY oc.fecha_estimada_entrega ASC;
```

### Movimiento de Inventario Detallado

Para el control de inventario, necesitas un registro detallado de cada movimiento: qué producto, en qué almacén, si fue entrada o salida, a qué costo, y quién lo hizo. Esta consulta usa `CASE` para separar entradas y salidas en columnas distintas, lo que facilita la lectura. El `LEFT JOIN` con almacenes es por si algún movimiento no tiene almacén asignado.

```sql
SELECT 
    m.id_movimiento,
    m.fecha_movimiento,
    p.nombre_producto,
    a.nombre AS almacen,
    m.tipo_movimiento,
    CASE 
        WHEN m.tipo_movimiento LIKE 'entrada%' THEN m.cantidad
        ELSE 0
    END AS entrada,
    CASE 
        WHEN m.tipo_movimiento LIKE 'salida%' THEN m.cantidad
        ELSE 0
    END AS salida,
    m.costo_unitario,
    m.cantidad * m.costo_unitario AS costo_total,
    m.referencia_tipo,
    m.referencia_id,
    m.usuario
FROM inventario_movimientos m
JOIN productos p ON m.id_producto = p.id_producto
LEFT JOIN almacenes a ON m.id_almacen = a.id_almacen
WHERE m.fecha_movimiento >= '2024-01-01'
ORDER BY m.fecha_movimiento DESC
LIMIT 100;
```

### KPI de Negocio (Múltiples JOIN)

Este es el tipo de consulta que alimenta los dashboards ejecutivos. Calcula indicadores clave por día: número de facturas, clientes activos, ingresos, ticket promedio, productos vendidos, y líneas de venta. Muestra cómo los JOINs permiten cruzar información de facturación con detalle de productos para obtener métricas que ninguna tabla individual podría proporcionar por sí sola.

```sql
SELECT 
    DATE(f.fecha_emision) AS dia,
    COUNT(DISTINCT f.id_factura) AS facturas,
    COUNT(DISTINCT f.id_cliente) AS clientes_activos,
    SUM(f.total) AS ingresos,
    ROUND(SUM(f.total) / COUNT(DISTINCT f.id_factura), 2) AS ticket_promedio,
    COUNT(DISTINCT fd.id_producto) AS productos_vendidos,
    COUNT(fd.id_detalle) AS lineas_vendidas,
    ROUND(SUM(fd.cantidad * fd.precio_unitario - fd.descuento), 2) AS venta_neta
FROM facturas f
JOIN facturas_detalle fd ON f.id_factura = fd.id_factura
WHERE f.estado = 'activa'
GROUP BY DATE(f.fecha_emision)
ORDER BY dia DESC;
```

### Detección de Fraude (Multi-JOIN Complejo)

Este es un ejemplo avanzado que muestra JOINs para análisis de fraude. La consulta auto-relaciona la tabla `facturas` dos veces (como `f_original` y `f_devolucion`) para comparar compras con devoluciones del mismo cliente. Identifica clientes con más de 3 devoluciones, de las cuales al menos 2 ocurrieron dentro de los 7 días posteriores a la compra (un posible patrón de fraude). Este tipo de consulta muestra el verdadero poder analítico de SQL.

```sql
SELECT 
    c.nombre,
    c.email,
    COUNT(DISTINCT f_original.id_factura) AS compras_realizadas,
    COUNT(DISTINCT f_devolucion.id_factura) AS devoluciones,
    SUM(f_devolucion.total) AS monto_devuelto,
    COUNT(DISTINCT CASE WHEN f_devolucion.fecha_emision > f_original.fecha_emision 
                        AND DATEDIFF(f_devolucion.fecha_emision, f_original.fecha_emision) <= 7 
                   THEN f_devolucion.id_factura END) AS devoluciones_rapidas
FROM clientes c
JOIN facturas f_original ON c.id_cliente = f_original.id_cliente 
    AND f_original.estado = 'activa'
JOIN facturas f_devolucion ON c.id_cliente = f_devolucion.id_cliente 
    AND f_devolucion.estado = 'devuelta_total'
GROUP BY c.id_cliente
HAVING COUNT(DISTINCT f_devolucion.id_factura) > 3
    AND devoluciones_rapidas > 2
ORDER BY devoluciones DESC;
```

### Productos Más Vendidos por Categoría

Un clásico reporte de ventas: ¿cuáles son los productos más vendidos de cada categoría? Une 4 tablas (productos, categorías, detalle de factura, facturas) y usa `GROUP BY` en dos niveles (categoría y producto) para obtener las unidades vendidas y los ingresos por producto. El filtro en el JOIN con facturas (fecha y estado) asegura que solo se consideren ventas activas de los últimos 3 meses.

```sql
SELECT 
    c.nombre AS categoria,
    p.nombre_producto,
    SUM(fd.cantidad) AS unidades_vendidas,
    SUM(fd.total) AS ingresos_totales
FROM productos p
JOIN categorias c ON p.id_categoria = c.id_categoria
JOIN facturas_detalle fd ON p.id_producto = fd.id_producto
JOIN facturas f ON fd.id_factura = f.id_factura
    AND f.estado = 'activa'
    AND f.fecha_emision >= DATE_SUB(CURRENT_DATE, INTERVAL 3 MONTH)
GROUP BY c.id_categoria, p.id_producto
HAVING SUM(fd.cantidad) > 0
ORDER BY c.nombre, SUM(fd.cantidad) DESC;
```

## Buenas Prácticas

Los JOINs son una de las áreas donde más errores se cometen. Estas prácticas te ayudarán a escribirlos correctamente desde el principio. La más importante: siempre usa `JOIN` explícito (`INNER JOIN`, `LEFT JOIN`, etc.) en lugar del JOIN implícito con comas en el `FROM`. El JOIN implícito puede generar productos cartesianos accidentales si olvidas el `WHERE`. También, usa `EXISTS` en lugar de `JOIN` cuando solo necesites verificar la existencia de un registro, no sus datos — es más eficiente y más claro.

```sql
SELECT * FROM clientes c, facturas f WHERE c.id_cliente = f.id_cliente;

SELECT * FROM clientes c JOIN facturas f ON c.id_cliente = f.id_cliente;

SELECT DISTINCT c.* FROM clientes c
JOIN facturas f ON c.id_cliente = f.id_cliente
JOIN facturas_detalle fd ON f.id_factura = fd.id_factura;

SELECT c.* FROM clientes c
WHERE EXISTS (SELECT 1 FROM facturas f WHERE f.id_cliente = c.id_cliente);
```

---

## Anterior: [02 Where Y Filtros](../02-fundamentos/02-where-y-filtros.md)
## Siguiente: [04 Agrupacion Y Agregacion](../02-fundamentos/04-agrupacion-y-agregacion.md)

[Volver al índice](../README.md)
