# 6.2 CTE y Recursividad

## Introducción a CTE (Common Table Expressions)

Un CTE es una tabla temporal con nombre que existe solo durante la ejecución de una consulta. Mejora la legibilidad y permite consultas [recursivas](https://ellibrodepython.com/recursividad).

### Sintaxis Básica
```sql
WITH nombre_cte AS (
    SELECT ... FROM ...
)
SELECT * FROM nombre_cte;
```

## CTE Simples

### CTE Básico
```sql
WITH ventas_2024 AS (
    SELECT id_factura, folio, total, fecha_emision
    FROM facturas
    WHERE YEAR(fecha_emision) = 2024 AND estado = 'activa'
)
SELECT 
    DATE_FORMAT(fecha_emision, '%Y-%m') AS mes,
    COUNT(*) AS facturas,
    SUM(total) AS total
FROM ventas_2024
GROUP BY DATE_FORMAT(fecha_emision, '%Y-%m')
ORDER BY mes;
```

### CTE con JOIN
```sql
WITH ventas_cliente AS (
    SELECT 
        c.id_cliente,
        c.nombre,
        COUNT(f.id_factura) AS total_compras,
        SUM(f.total) AS total_gastado
    FROM clientes c
    LEFT JOIN facturas f ON c.id_cliente = f.id_cliente AND f.estado = 'activa'
    GROUP BY c.id_cliente
)
SELECT * FROM ventas_cliente
WHERE total_gastado > 10000
ORDER BY total_gastado DESC;
```

## Múltiples CTE

```sql
WITH 
ventas_mensuales AS (
    SELECT 
        DATE_FORMAT(fecha_emision, '%Y-%m') AS mes,
        SUM(total) AS ventas
    FROM facturas
    WHERE estado = 'activa'
    GROUP BY DATE_FORMAT(fecha_emision, '%Y-%m')
),
ventas_acumuladas AS (
    SELECT 
        mes,
        ventas,
        SUM(ventas) OVER (ORDER BY mes) AS acumulado
    FROM ventas_mensuales
),
promedio_anual AS (
    SELECT AVG(ventas) AS promedio FROM ventas_mensuales
)
SELECT 
    va.mes,
    va.ventas,
    va.acumulado,
    ROUND(va.ventas - pa.promedio, 2) AS diferencia_promedio
FROM ventas_acumuladas va
CROSS JOIN promedio_anual pa
ORDER BY va.mes;
```

## CTE Recursivo

Permite recorrer estructuras jerárquicas como árboles de categorías, organigramas, etc.

### Sintaxis
```sql
WITH RECURSIVE nombre_cte AS (
    -- Ancla: consulta base (nivel 0)
    SELECT ... WHERE condicion_raiz
    
    UNION ALL
    
    -- Paso recursivo: se une al CTE
    SELECT ... FROM nombre_cte WHERE condicion_recursiva
)
SELECT * FROM nombre_cte;
```

### 1. Jerarquía de Categorías
```sql
-- Obtener toda la jerarquía de categorías desde una raíz
WITH RECURSIVE categorias_tree AS (
    -- Ancla: categorías raíz (sin padre)
    SELECT 
        id_categoria,
        id_categoria_padre,
        nombre,
        1 AS nivel,
        CAST(nombre AS CHAR(500)) AS ruta
    FROM categorias
    WHERE id_categoria_padre IS NULL
    
    UNION ALL
    
    -- Paso recursivo: hijos
    SELECT 
        c.id_categoria,
        c.id_categoria_padre,
        c.nombre,
        ct.nivel + 1,
        CONCAT(ct.ruta, ' > ', c.nombre)
    FROM categorias c
    INNER JOIN categorias_tree ct ON c.id_categoria_padre = ct.id_categoria
)
SELECT * FROM categorias_tree
ORDER BY ruta;
```

### 2. Órdenes de Compra con Historial de Estados
```sql
WITH RECURSIVE estados_orden AS (
    -- Estado inicial
    SELECT 
        id_orden,
        estado,
        fecha_emision AS fecha_cambio,
        1 AS secuencia
    FROM ordenes_compra
    WHERE estado = 'borrador'
    
    UNION ALL
    
    -- Estados siguientes (simulado con fechas)
    SELECT 
        oc.id_orden,
        CASE 
            WHEN eo.secuencia = 1 THEN 'autorizada'
            WHEN eo.secuencia = 2 THEN 'enviada'
            WHEN eo.secuencia = 3 THEN 'confirmada'
            WHEN eo.secuencia = 4 THEN 'recibida_parcial'
            WHEN eo.secuencia = 5 THEN 'recibida_total'
        END,
        DATE_ADD(eo.fecha_cambio, INTERVAL 1 DAY),
        eo.secuencia + 1
    FROM estados_orden eo
    JOIN ordenes_compra oc ON eo.id_orden = oc.id_orden
    WHERE eo.secuencia < 5
)
SELECT * FROM estados_orden
ORDER BY id_orden, secuencia;
```

### 3. Generar Series de Fechas
```sql
-- Generar todos los días de un mes (útil para completar reportes)
WITH RECURSIVE fechas AS (
    SELECT '2024-01-01' AS fecha
    UNION ALL
    SELECT DATE_ADD(fecha, INTERVAL 1 DAY)
    FROM fechas
    WHERE fecha < '2024-01-31'
)
SELECT 
    f.fecha,
    COALESCE(SUM(fac.total), 0) AS ventas
FROM fechas f
LEFT JOIN facturas fac ON DATE(fac.fecha_emision) = f.fecha 
    AND fac.estado = 'activa'
GROUP BY f.fecha
ORDER BY f.fecha;
```

### 4. Organigrama de Vendedores
```sql
WITH RECURSIVE organigrama AS (
    SELECT 
        id_vendedor,
        nombre_vendedor,
        id_jefe,
        nombre_vendedor AS nombre_nivel,
        1 AS nivel,
        CAST(nombre_vendedor AS CHAR(500)) AS ruta_organigrama
    FROM vendedores
    WHERE id_jefe IS NULL
    
    UNION ALL
    
    SELECT 
        v.id_vendedor,
        v.nombre_vendedor,
        v.id_jefe,
        v.nombre_vendedor,
        o.nivel + 1,
        CONCAT(o.ruta_organigrama, ' -> ', v.nombre_vendedor)
    FROM vendedores v
    INNER JOIN organigrama o ON v.id_jefe = o.id_vendedor
)
SELECT * FROM organigrama
ORDER BY ruta_organigrama;
```

### 5. Reporte de Ventas Completando Meses sin Ventas
```sql
WITH RECURSIVE meses AS (
    SELECT '2024-01-01' AS mes_inicio
    UNION ALL
    SELECT DATE_ADD(mes_inicio, INTERVAL 1 MONTH)
    FROM meses
    WHERE mes_inicio < '2024-12-01'
)
SELECT 
    DATE_FORMAT(m.mes_inicio, '%Y-%m') AS mes,
    COALESCE(SUM(f.total), 0) AS ventas,
    COALESCE(COUNT(f.id_factura), 0) AS facturas
FROM meses m
LEFT JOIN facturas f ON MONTH(f.fecha_emision) = MONTH(m.mes_inicio)
    AND YEAR(f.fecha_emision) = YEAR(m.mes_inicio)
    AND f.estado = 'activa'
GROUP BY m.mes_inicio
ORDER BY m.mes_inicio;
```

## CTE vs Subconsultas vs Tablas Temporales

```sql
-- 1. Subconsulta (difícil de leer)
SELECT * FROM (
    SELECT c.nombre, SUM(f.total) as total
    FROM clientes c JOIN facturas f ON c.id_cliente = f.id_cliente
    GROUP BY c.id_cliente
) AS res WHERE total > 10000;

-- 2. CTE (más legible)
WITH resumen AS (
    SELECT c.nombre, SUM(f.total) as total
    FROM clientes c JOIN facturas f ON c.id_cliente = f.id_cliente
    GROUP BY c.id_cliente
)
SELECT * FROM resumen WHERE total > 10000;

-- 3. Tabla temporal (persistente, reusable)
CREATE TEMPORARY TABLE resumen_temp AS
SELECT c.nombre, SUM(f.total) as total
FROM clientes c JOIN facturas f ON c.id_cliente = f.id_cliente
GROUP BY c.id_cliente;

SELECT * FROM resumen_temp WHERE total > 10000;
```

## 📊 Ejemplos del Mundo Real

### Análisis de Canasta con CTE
```sql
WITH productos_vendidos AS (
    SELECT 
        f.id_factura,
        GROUP_CONCAT(p.nombre_comercial ORDER BY p.nombre_comercial) AS canasta
    FROM facturas f
    JOIN facturas_detalle fd ON f.id_factura = fd.id_factura
    JOIN productos p ON fd.id_producto = p.id_producto
    GROUP BY f.id_factora
)
SELECT 
    canasta,
    COUNT(*) AS frecuencia
FROM productos_vendidos
GROUP BY canasta
ORDER BY frecuencia DESC
LIMIT 10;
```

### Segmentación RFM con CTE
```sql
WITH rfm AS (
    SELECT 
        c.id_cliente,
        c.nombre,
        DATEDIFF(CURRENT_DATE, MAX(f.fecha_emision)) AS recencia,
        COUNT(DISTINCT f.id_factura) AS frecuencia,
        SUM(f.total) AS monetario
    FROM clientes c
    JOIN facturas f ON c.id_cliente = f.id_cliente AND f.estado = 'activa'
    GROUP BY c.id_cliente
),
rfm_scores AS (
    SELECT 
        *,
        NTILE(5) OVER (ORDER BY recencia ASC) AS r,
        NTILE(5) OVER (ORDER BY frecuencia DESC) AS f,
        NTILE(5) OVER (ORDER BY monetario DESC) AS m
    FROM rfm
)
SELECT 
    CONCAT(r, f, m) AS rfm_score,
    COUNT(*) AS clientes,
    ROUND(AVG(monetario), 2) AS gasto_promedio,
    GROUP_CONCAT(nombre SEPARATOR ', ') AS ejemplos_clientes
FROM rfm_scores
GROUP BY CONCAT(r, f, m)
ORDER BY rfm_score DESC;
```

### Reemplazo de Subconsultas Correlacionadas con CTE
```sql
-- Sin CTE (subconsulta correlacionada, lenta)
SELECT 
    p1.nombre_producto,
    p1.precio_venta,
    (SELECT AVG(p2.precio_venta) FROM productos p2 
     WHERE p2.id_categoria = p1.id_categoria) AS promedio_categoria
FROM productos p1;

-- Con CTE (más eficiente)
WITH promedio_categorias AS (
    SELECT id_categoria, AVG(precio_venta) AS precio_promedio
    FROM productos
    GROUP BY id_categoria
)
SELECT 
    p.nombre_producto,
    p.precio_venta,
    pc.precio_promedio
FROM productos p
JOIN promedio_categorias pc ON p.id_categoria = pc.id_categoria;
```

---

## Anterior: [01 Window Functions](../06-sql-avanzado/01-window-functions.md)
## Siguiente: [03 Store Procedures](../06-sql-avanzado/03-store-procedures.md)

[Volver al índice](../README.md)
