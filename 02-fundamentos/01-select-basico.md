# 2.1 SELECT Básico

## La Sentencia SELECT

Imagina que tienes una hoja de cálculo gigante con todos los datos de tu negocio: clientes, productos, facturas, etc. Cada una de esas hojas es lo que en SQL llamamos una **tabla**. La sentencia `SELECT` es tu herramienta principal para hacer preguntas a esas tablas y obtener respuestas. Es como decirle a la base de datos: "muéstrame lo que hay aquí". Con `SELECT` puedes elegir qué columnas quieres ver, en qué orden y si quieres calcular cosas nuevas sobre la marcha.

### Sintaxis Básica

La estructura más fundamental de una consulta SQL se compone de dos partes: `SELECT` (qué quieres ver) y `FROM` (de dónde quieres verlo). Piensa en ello como si le dijeras a la base de datos: "Oye, de la tabla tal, enséñame estas columnas". Es tan simple como eso. Todo lo demás que aprenderás después (filtros, joins, agrupaciones) se construye sobre esta base.

```sql
SELECT columna1, columna2, ...
FROM nombre_tabla;
```

## Seleccionar Columnas

### Todas las Columnas

Cuando recién estás explorando una tabla y no sabes exactamente qué columnas contiene, o cuando literalmente necesitas absolutamente todos los campos, puedes usar el asterisco (`*`) como un comodín que significa "todo". Es como abrir un cajón y vaciar todo su contenido sobre la mesa para ver qué hay. En la práctica, esto es muy útil durante la fase de exploración o depuración, pero no se recomienda para sistemas en producción porque traer columnas innecesarias consume recursos.

```sql
SELECT * FROM clientes;
```

### Columnas Específicas

Casi siempre te interesarán solo ciertos campos. Si tienes una tabla de clientes con 20 columnas (nombre, email, teléfono, dirección, fecha de registro, etc.), pero solo necesitas el nombre y el email para enviar una campaña de correos, no tiene sentido traer todo lo demás. Especificar las columnas una por una es más eficiente, más claro para quien lee la consulta, y evita confusiones cuando después trabajes con varias tablas.

```sql
SELECT nombre, email, telefono FROM clientes;
```

### Columnas Calculadas

Una de las capacidades más poderosas de SQL es que puede **hacer operaciones matemáticas en el momento de la consulta**, sin modificar los datos originales. Imagina que tienes una tabla de productos con el precio sin IVA. En tu reporte necesitas mostrar tanto el IVA como el precio final, pero no tienes columnas para eso. No importa: SQL puede calcular `precio_venta * 0.16` para obtener el IVA, o `precio_venta * 1.16` para el precio con IVA. Esto se conoce como **columna calculada**: el resultado existe solo en la consulta, los datos en la tabla permanecen intactos.

```sql
SELECT 
    nombre_producto,
    precio_venta,
    precio_venta * 0.16 AS iva,
    precio_venta * 1.16 AS precio_con_iva
FROM productos;
```

## Alias (AS)

A veces los nombres de las columnas en la base de datos son crípticos (`precio_venta`, `id_categoria_padre`) o el resultado de una operación matemática se muestra con un nombre técnico feo como `precio_venta * 1.16`. Ahí es donde entran los **alias**: son nombres temporales y alternativos que le pones a una columna o tabla para que el resultado sea más legible. Piensa en `AS` como una etiquetadora: tomas algo y le pones una etiqueta que dice cómo quieres que se llame en el resultado. Los alias de tabla (como `FROM productos AS p`) son especialmente útiles cuando trabajas con varias tablas, porque te evitan escribir el nombre completo una y otra vez.

```sql
SELECT 
    nombre_producto AS producto,
    precio_venta AS precio_sin_iva,
    ROUND(precio_venta * 1.16, 2) AS precio_final
FROM productos AS p;

SELECT p.nombre_producto, c.nombre AS categoria
FROM productos p
JOIN categorias c ON p.id_categoria = c.id_categoria;
```

## DISTINCT - Valores Únicos

Pongamos un ejemplo concreto: tienes una tabla de clientes con una columna llamada `ciudad`. Si haces un `SELECT ciudad FROM clientes`, obtendrás una fila por cada cliente, lo que significa que "Ciudad de México" aparecerá cientos de veces, una por cada cliente de allá. Pero si lo que quieres saber es **en qué ciudades tenemos clientes**, sin repeticiones, usas `DISTINCT`. Es como tomar una lista, eliminar todos los duplicados, y quedarte solo con los valores únicos. Puedes aplicarlo a una sola columna (ciudades distintas) o a una combinación de varias (pares únicos de ciudad y estado).

```sql
SELECT DISTINCT ciudad FROM clientes;

SELECT DISTINCT ciudad, estado FROM sucursales;
```

## LIMIT y OFFSET - Paginación

Cuando una tabla tiene millones de registros, hacer `SELECT * FROM productos` traería todo, lo cual sería lentísimo y probablemente congelaría tu herramienta de base de datos. `LIMIT` te permite decir "solo quiero las primeras N filas". Es como decir "enséñame solo la primera página". `OFFSET` es el complemento que te permite saltarte cierta cantidad de filas. Juntos, `LIMIT` y `OFFSET` implementan **paginación**: la página 1 son los primeros 10 registros, la página 2 son los siguientes 10 (saltándote los primeros 10), y así sucesivamente.

```sql
SELECT * FROM productos LIMIT 10;

SELECT * FROM productos LIMIT 10 OFFSET 10;

SELECT * FROM productos LIMIT 10, 10;
```

## ORDER BY - Ordenamiento

Por defecto, las consultas SQL no garantizan ningún orden en particular; los registros pueden aparecer en el orden en que se insertaron, en el orden que el motor decida, o incluso en órdenes diferentes entre ejecuciones. Si quieres un orden específico, debes decirlo explícitamente con `ORDER BY`. Puedes ordenar de **menor a mayor** (`ASC`, que es el comportamiento por defecto) o de **mayor a menor** (`DESC`). También puedes ordenar por múltiples criterios: por ejemplo, primero por precio descendente (los más caros primero) y, en caso de empate, por nombre ascendente (orden alfabético).

```sql
SELECT nombre_producto, precio_venta 
FROM productos 
ORDER BY precio_venta;

SELECT nombre_producto, precio_venta 
FROM productos 
ORDER BY precio_venta DESC;

SELECT nombre_producto, precio_venta, stock_actual 
FROM productos 
ORDER BY precio_venta DESC, nombre_producto ASC;
```

## Operaciones Aritméticas

SQL entiende las operaciones matemáticas básicas: suma (`+`), resta (`-`), multiplicación (`*`) y división (`/`). Puedes combinar estas operaciones en una sola consulta para crear columnas calculadas que respondan preguntas de negocio específicas. Por ejemplo, en una tabla de detalle de factura tienes `cantidad`, `precio_unitario` y `descuento`. Con una simple multiplicación obtienes el subtotal bruto, y restando el descuento obtienes el subtotal neto. Todo esto se calcula al vuelo, sin necesidad de almacenar esos valores en la base de datos.

```sql
SELECT 
    id_detalle,
    cantidad,
    precio_unitario,
    descuento,
    cantidad * precio_unitario AS subtotal_bruto,
    (cantidad * precio_unitario) - descuento AS subtotal_neto
FROM facturas_detalle;
```

## CASE - Condicionales en SELECT

El `CASE` es el equivalente SQL de la estructura "si esto, entonces aquello; si no, esto otro". Es como tomar decisiones dentro de la consulta misma. En lugar de tener que exportar los datos a Excel y ahí aplicar colores o categorías, puedes hacerlo directamente en SQL. Por ejemplo, imagina que quieres clasificar tus productos según su nivel de stock: sin stock, stock bajo, stock medio, stock suficiente. Con `CASE` evalúas condición por condición (en orden) y cuando una se cumple, asignas la etiqueta correspondiente.

```sql
SELECT 
    nombre_producto,
    stock_actual,
    stock_minimo,
    CASE 
        WHEN stock_actual <= 0 THEN 'SIN STOCK'
        WHEN stock_actual < stock_minimo THEN 'STOCK BAJO'
        WHEN stock_actual < stock_minimo * 3 THEN 'STOCK MEDIO'
        ELSE 'STOCK SUFICIENTE'
    END AS nivel_stock
FROM productos;
```

## Concatenación de Texto

Frecuentemente necesitarás juntar (concatenar) varias columnas de texto en una sola. Por ejemplo, para mostrar "Juan Pérez - juan@email.com" a partir de las columnas `nombre` y `email`. La forma de hacer esto varía ligeramente entre sistemas de bases de datos: en MySQL se usa la función `CONCAT()`, mientras que en PostgreSQL se usa el operador `||`. Ambas hacen exactamente lo mismo: unir textos.

```sql
SELECT CONCAT(nombre, ' - ', email) AS cliente_info FROM clientes;

SELECT nombre || ' - ' || email AS cliente_info FROM clientes;

SELECT CONCAT(nombre, ' - ', email) AS cliente_info FROM clientes;
```

## Fechas en SELECT

Trabajar con fechas es una de las tareas más comunes en SQL. Las bases de datos guardan las fechas en un formato interno (generalmente `YYYY-MM-DD HH:MM:SS`), pero tú puedes extraer partes específicas usando funciones como `YEAR()`, `MONTH()`, `DAY()`, `HOUR()`, etc. También puedes cambiar el formato de presentación con `DATE_FORMAT()` para mostrar, por ejemplo, "25/12/2024 15:30" en lugar del formato estándar. Esto es útil para reportes donde los usuarios quieren ver las fechas en un formato familiar.

```sql
SELECT 
    id_factura,
    folio,
    fecha_emision,
    DATE(fecha_emision) AS solo_fecha,
    TIME(fecha_emision) AS solo_hora,
    YEAR(fecha_emision) AS año,
    MONTH(fecha_emision) AS mes,
    DAY(fecha_emision) AS dia,
    DAYNAME(fecha_emision) AS nombre_dia,
    DATE_FORMAT(fecha_emision, '%d/%m/%Y %H:%i') AS fecha_formateada
FROM facturas;
```

## Funciones de Cadena Útiles

SQL trae muchas funciones para manipular texto. ¿Necesitas que todos los nombres de producto aparezcan en mayúsculas? Usa `UPPER()`. ¿Quieres saber cuántos caracteres tiene una descripción? Usa `LENGTH()`. ¿Necesitas extraer los primeros 10 caracteres de un campo? Usa `LEFT()`. Estas funciones son especialmente útiles para limpiar datos, preparar reportes, o crear campos derivados como códigos SKU combinando texto de otras columnas.

```sql
SELECT 
    nombre_producto,
    UPPER(nombre_producto) AS mayusculas,
    LOWER(nombre_producto) AS minusculas,
    LENGTH(nombre_producto) AS longitud,
    LEFT(nombre_producto, 10) AS primeros_10,
    RIGHT(nombre_producto, 5) AS ultimos_5,
    SUBSTRING(nombre_producto, 1, 10) AS subcadena,
    TRIM(nombre_producto) AS sin_espacios,
    REPLACE(nombre_producto, ' ', '_') AS con_guiones,
    CONCAT('SKU-', codigo_barras) AS codigo_completo
FROM productos;
```

## COALESCE y NULLIF

En las bases de datos, `NULL` significa "valor desconocido o ausente", no es lo mismo que cero o que una cadena vacía. Esto puede causarte problemas: si haces una suma y un valor es `NULL`, el resultado será `NULL`, no cero. `COALESCE()` es una función que toma una lista de valores y retorna el **primero que no sea NULL**. Es perfecta para poner valores por defecto: "si la descripción es NULL, entonces muestra 'Sin descripción'". Por otro lado, `NULLIF()` hace lo contrario: si dos valores son iguales, retorna `NULL`. Es útil para evitar divisiones entre cero: `NULLIF(stock_actual, 0)` retorna NULL si el stock es 0, evitando un error matemático.

```sql
SELECT 
    nombre_producto,
    COALESCE(descripcion, 'Sin descripción') AS descripcion,
    COALESCE(precio_venta, 0) AS precio
FROM productos;

SELECT 
    nombre_producto,
    NULLIF(stock_actual, 0) AS stock_actual
FROM productos;
```

## 📊 Ejemplos Aplicados a Ventas

### Listado básico de productos

Este es un ejemplo real de cómo se vería un listado de productos para un sistema de punto de venta. Necesitas mostrar el código de barras, el nombre, la categoría (que viene de otra tabla), el precio formateado con signo de pesos, y el stock actual. Solo quieres productos activos, ordenados primero por categoría y luego alfabéticamente. Nota cómo combinamos varias de las técnicas que aprendiste: JOIN para traer el nombre de la categoría, alias para renombrar columnas, CONCAT y FORMAT para dar formato al precio, y WHERE para filtrar solo activos.

```sql
SELECT 
    p.codigo_barras,
    p.nombre_producto,
    c.nombre AS categoria,
    CONCAT('$', FORMAT(p.precio_venta, 2)) AS precio,
    p.stock_actual
FROM productos p
JOIN categorias c ON p.id_categoria = c.id_categoria
WHERE p.activo = TRUE
ORDER BY c.nombre, p.nombre_producto;
```

### Últimas ventas realizadas

Cuando entras a un sistema de ventas, lo primero que quieres ver son las transacciones más recientes. Esta consulta te trae las últimas 20 facturas emitidas, ordenadas de la más reciente a la más antigua. Combinas información de la tabla `facturas` con el nombre del cliente (de la tabla `clientes`) para tener un reporte legible. El `ORDER BY ... DESC` asegura que la venta más reciente aparezca primero.

```sql
SELECT 
    f.folio,
    c.nombre AS cliente,
    f.fecha_emision,
    f.total,
    f.estado
FROM facturas f
JOIN clientes c ON f.id_cliente = c.id_cliente
ORDER BY f.fecha_emision DESC
LIMIT 20;
```

### Detalle de una venta específica

Cuando un cliente pregunta "¿qué compré en la factura 1001?", esta consulta te da la respuesta. Aquí filtrar por un folio específico usando `WHERE` y luego muestras cada línea del ticket: qué producto, a qué precio, con qué descuento, y los totales por línea. Es la consulta típica del botón "ver detalle" en cualquier sistema de facturación.

```sql
SELECT 
    fd.cantidad,
    p.nombre_producto,
    fd.precio_unitario,
    fd.descuento,
    fd.subtotal,
    fd.iva,
    fd.total
FROM facturas_detalle fd
JOIN productos p ON fd.id_producto = p.id_producto
WHERE fd.id_factura = 1001
ORDER BY fd.id_detalle;
```

### Productos con precios y márgenes

Una consulta muy útil para el área de finanzas: calcular el margen de ganancia de cada producto. Restas el precio de compra al precio de venta para obtener el margen bruto en pesos, y luego calculas qué porcentaje representa ese margen sobre el precio de venta. El `ROUND(..., 2)` redondea a dos decimales. Ordenas de mayor a menor margen para identificar rápidamente tus productos más rentables.

```sql
SELECT 
    p.nombre_producto,
    p.precio_compra,
    p.precio_venta,
    (p.precio_venta - p.precio_compra) AS margen_bruto,
    ROUND((p.precio_venta - p.precio_compra) / p.precio_venta * 100, 2) AS margen_porcentaje
FROM productos p
WHERE p.activo = TRUE
ORDER BY margen_porcentaje DESC;
```

## Ejercicios Prácticos

### Ejercicio 1: Consultas básicas

Estos cuatro ejercicios cubren las operaciones más comunes que harás día con día. El primero es simplemente explorar una tabla completa. El segundo agrega un filtro básico con `WHERE`. El tercero combina `ORDER BY` y `LIMIT` para obtener un top. El cuarto usa funciones de fecha para filtrar un período específico (los últimos 7 días). Practica estos patrones hasta que te salgan sin pensar.

```sql
SELECT * FROM clientes;

SELECT nombre, email FROM clientes WHERE activo = TRUE;

SELECT nombre_producto, precio_venta 
FROM productos 
ORDER BY precio_venta DESC 
LIMIT 5;

SELECT folio, fecha_emision, total 
FROM facturas 
WHERE fecha_emision >= DATE_SUB(CURRENT_DATE, INTERVAL 7 DAY)
ORDER BY fecha_emision DESC;
```

### Ejercicio 2: Crear un reporte de productos

Este ejercicio combina casi todo lo que aprendiste: JOIN entre dos tablas, CASE para clasificar, alias descriptivos, y un ORDER BY un poco más avanzado que ordena por una columna calculada (la prioridad del estado). El objetivo es generar un reporte de inventario donde los productos con estado "Urgente" (sin stock) aparezcan primero, seguidos de "Reordenar", y al final los que están "OK". Es exactamente el tipo de reporte que usarías en un almacén para saber qué productos necesitan atención inmediata.

```sql
SELECT 
    p.codigo_barras AS 'Código',
    p.nombre_producto AS 'Producto',
    c.nombre AS 'Categoría',
    p.precio_venta AS 'Precio Venta',
    p.stock_actual AS 'Stock',
    CASE 
        WHEN p.stock_actual <= 0 THEN 'Urgente'
        WHEN p.stock_actual < p.stock_minimo THEN 'Reordenar'
        ELSE 'OK'
    END AS 'Estado'
FROM productos p
JOIN categorias c ON p.id_categoria = c.id_categoria
ORDER BY 
    CASE estado
        WHEN 'Urgente' THEN 1
        WHEN 'Reordenar' THEN 2
        ELSE 3
    END;
```

## Buenas Prácticas

A lo largo de tu carrera con SQL, repetirás ciertos patrones. Aquí hay cuatro reglas de oro que te ahorrarán dolores de cabeza. La primera: nunca uses `SELECT *` en producción, siempre especifica las columnas que necesitas; esto hace que tu consulta sea más rápida y predecible. La segunda: cuando calcules una columna, ponle un alias descriptivo usando `AS`, para que quien lea el resultado entienda qué significa ese número. La tercera: durante desarrollo, siempre pon un `LIMIT` para evitar traer millones de registros accidentalmente. La cuarta: cuando el orden importe, no asumas nada, ordénalo explícitamente con `ORDER BY`.

```sql
SELECT * FROM clientes WHERE ...;

SELECT id_cliente, nombre, email FROM clientes WHERE ...;

SELECT * FROM facturas;

SELECT * FROM facturas LIMIT 100;
```

---

## Anterior: [03 Configuracion Entorno](../01-introduccion/03-configuracion-entorno.md)
## Siguiente: [02 Where Y Filtros](../02-fundamentos/02-where-y-filtros.md)

[Volver al índice](../README.md)
