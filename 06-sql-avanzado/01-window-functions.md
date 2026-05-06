# 6.1 Window Functions

## Introducción

Las funciones de ventana (Window Functions) realizan cálculos sobre un conjunto de filas relacionadas con la fila actual, sin agrupar los resultados como GROUP BY.

### Diferencia con GROUP BY
```sql
-- GROUP BY: reduce filas
SELECT categoria, SUM(total) FROM facturas GROUP BY categoria;

-- Window Functions: mantiene filas individuales + cálculo
SELECT fecha, total, SUM(total) OVER () AS gran_total FROM facturas;
```

## Sintaxis General
```sql
funcion_ventana() OVER (
    [PARTITION BY columna(s)]
    [ORDER BY columna(s)]
    [ROWS/RANGE entre]
)
```

## ROW_NUMBER, RANK, DENSE_RANK

### ROW_NUMBER - Numeración correlativa
```sql
SELECT 
    v.nombre_vendedor,
    SUM(f.total) AS ventas,
    ROW_NUMBER() OVER (ORDER BY SUM(f.total) DESC) AS ranking
FROM vendedores v
JOIN facturas f ON v.id_vendedor = f.id_vendedor
WHERE f.estado = 'activa'
GROUP BY v.id_vendedor;
```

### RANK y DENSE_RANK - Ranking con empates
```sql
SELECT 
    v.nombre_vendedor,
    SUM(f.total) AS ventas,
    RANK() OVER (ORDER BY SUM(f.total) DESC) AS ranking_con_empates,
    DENSE_RANK() OVER (ORDER BY SUM(f.total) DESC) AS ranking_sin_saltos,
    ROW_NUMBER() OVER (ORDER BY SUM(f.total) DESC) AS fila_unica
FROM vendedores v
JOIN facturas f ON v.id_vendedor = f.id_vendedor
WHERE f.estado = 'activa'
GROUP BY v.id_vendedor;
```

### Top N por Grupo (el uso más común)
```sql
-- Top 3 productos por categoría
SELECT * FROM (
    SELECT 
        c.nombre AS categoria,
        p.nombre_comercial AS producto,
        SUM(fd.cantidad) AS unidades,
        ROW_NUMBER() OVER (
            PARTITION BY c.id_categoria 
            ORDER BY SUM(fd.cantidad) DESC
        ) AS ranking
    FROM categorias c
    JOIN productos p ON c.id_categoria = p.id_categoria
    JOIN facturas_detalle fd ON p.id_producto = fd.id_producto
    GROUP BY c.id_categoria, p.id_producto
) AS ranked
WHERE ranking <= 3;
```

## LAG y LEAD - Acceso a filas adyacentes

### LAG - Fila anterior
```sql
SELECT 
    DATE_FORMAT(fecha_emision, '%Y-%m') AS mes,
    SUM(total) AS ventas,
    LAG(SUM(total)) OVER (ORDER BY DATE_FORMAT(fecha_emision, '%Y-%m')) AS mes_anterior,
    LAG(SUM(total), 2) OVER (ORDER BY DATE_FORMAT(fecha_emision, '%Y-%m')) AS hace_2_meses,
    ROUND(
        (SUM(total) - LAG(SUM(total)) OVER (ORDER BY DATE_FORMAT(fecha_emision, '%Y-%m'))) 
        / NULLIF(LAG(SUM(total)) OVER (ORDER BY DATE_FORMAT(fecha_emision, '%Y-%m')), 0) * 100, 2
    ) AS variacion_mensual
FROM facturas
WHERE estado = 'activa'
GROUP BY DATE_FORMAT(fecha_emision, '%Y-%m')
ORDER BY mes;
```

### LEAD - Fila siguiente
```sql
SELECT 
    fecha_emision,
    total,
    LEAD(total) OVER (ORDER BY fecha_emision) AS siguiente_venta,
    LEAD(total) OVER (ORDER BY fecha_emision) - total AS diferencia_siguiente
FROM facturas
WHERE estado = 'activa' AND DATE(fecha_emision) = CURRENT_DATE
ORDER BY fecha_emision;
```

## SUM, AVG, MIN, MAX como Window Functions

### Suma Acumulada (Running Total)
```sql
SELECT 
    DATE(fecha_emision) AS dia,
    SUM(total) AS ventas_dia,
    SUM(SUM(total)) OVER (ORDER BY DATE(fecha_emision)) AS ventas_acumuladas,
    SUM(SUM(total)) OVER () AS ventas_totales_periodo
FROM facturas
WHERE YEAR(fecha_emision) = 2024 AND estado = 'activa'
GROUP BY DATE(fecha_emision)
ORDER BY dia;
```

### Media Móvil
```sql
SELECT 
    DATE(fecha_emision) AS dia,
    SUM(total) AS ventas_dia,
    ROUND(AVG(SUM(total)) OVER (
        ORDER BY DATE(fecha_emision) 
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ), 2) AS media_movil_7dias,
    ROUND(AVG(SUM(total)) OVER (
        ORDER BY DATE(fecha_emision) 
        ROWS BETWEEN 29 PRECEDING AND CURRENT ROW
    ), 2) AS media_movil_30dias
FROM facturas
WHERE estado = 'activa'
GROUP BY DATE(fecha_emision)
ORDER BY dia;
```

### Mínimo y Máximo Móvil
```sql
SELECT 
    DATE(fecha_emision) AS dia,
    SUM(total) AS ventas,
    MIN(SUM(total)) OVER (ORDER BY DATE(fecha_emision) ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS min_7dias,
    MAX(SUM(total)) OVER (ORDER BY DATE(fecha_emision) ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS max_7dias
FROM facturas
WHERE YEAR(fecha_emision) = 2024 AND estado = 'activa'
GROUP BY DATE(fecha_emision)
ORDER BY dia;
```

## FIRST_VALUE y LAST_VALUE

```sql
SELECT 
    DATE_FORMAT(fecha_emision, '%Y-%m') AS mes,
    SUM(total) AS ventas,
    FIRST_VALUE(SUM(total)) OVER (ORDER BY DATE_FORMAT(fecha_emision, '%Y-%m')) AS primer_mes,
    LAST_VALUE(SUM(total)) OVER (
        ORDER BY DATE_FORMAT(fecha_emision, '%Y-%m')
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS ultimo_mes,
    ROUND((SUM(total) - FIRST_VALUE(SUM(total)) OVER (ORDER BY DATE_FORMAT(fecha_emision, '%Y-%m'))) 
        / NULLIF(FIRST_VALUE(SUM(total)) OVER (ORDER BY DATE_FORMAT(fecha_emision, '%Y-%m')), 0) * 100, 2
    ) AS crecimiento_desde_inicio
FROM facturas
WHERE YEAR(fecha_emision) = 2024 AND estado = 'activa'
GROUP BY DATE_FORMAT(fecha_emision, '%Y-%m')
ORDER BY mes;
```

## NTILE - Percentiles y Cuartiles

```sql
-- Dividir clientes en 4 cuartiles por gasto
SELECT 
    c.nombre,
    SUM(f.total) AS gasto_total,
    NTILE(4) OVER (ORDER BY SUM(f.total) DESC) AS cuartil,
    NTILE(10) OVER (ORDER BY SUM(f.total) DESC) AS decil
FROM clientes c
JOIN facturas f ON c.id_cliente = f.id_cliente AND f.estado = 'activa'
GROUP BY c.id_cliente
ORDER BY gasto_total DESC;
```

## CUME_DIST y PERCENT_RANK

```sql
SELECT 
    c.nombre,
    SUM(f.total) AS gasto_total,
    ROUND(CUME_DIST() OVER (ORDER BY SUM(f.total) DESC), 4) AS distribucion_acumulada,
    ROUND(PERCENT_RANK() OVER (ORDER BY SUM(f.total) DESC), 4) AS rango_percentil
FROM clientes c
JOIN facturas f ON c.id_cliente = f.id_cliente AND f.estado = 'activa'
GROUP BY c.id_cliente
ORDER BY gasto_total DESC;
```

## PARTITION BY - Agrupación dentro de la ventana

```sql
-- Comparar cada venta con el promedio de su mes
SELECT 
    f.id_factura,
    DATE_FORMAT(f.fecha_emision, '%Y-%m') AS mes,
    f.total,
    ROUND(AVG(f.total) OVER (PARTITION BY DATE_FORMAT(f.fecha_emision, '%Y-%m')), 2) AS promedio_mensual,
    ROUND(f.total - AVG(f.total) OVER (PARTITION BY DATE_FORMAT(f.fecha_emision, '%Y-%m')), 2) AS diferencia_vs_promedio,
    ROW_NUMBER() OVER (
        PARTITION BY DATE_FORMAT(f.fecha_emision, '%Y-%m') 
        ORDER BY f.total DESC
    ) AS ranking_mensual
FROM facturas f
WHERE f.estado = 'activa' AND YEAR(f.fecha_emision) = 2024;
```

## 📊 Ejemplos del Mundo Real

### Crecimiento Mensual con LAG
```sql
WITH ventas_mensuales AS (
    SELECT 
        DATE_FORMAT(fecha_emision, '%Y-%m') AS mes,
        SUM(total) AS ventas
    FROM facturas
    WHERE estado = 'activa'
    GROUP BY DATE_FORMAT(fecha_emision, '%Y-%m')
)
SELECT 
    mes,
    ventas,
    LAG(ventas) OVER (ORDER BY mes) AS ventas_mes_anterior,
    ROUND((ventas - LAG(ventas) OVER (ORDER BY mes)) / NULLIF(LAG(ventas) OVER (ORDER BY mes), 0) * 100, 2) AS crecimiento,
    ROUND(AVG(ventas) OVER (ORDER BY mes ROWS BETWEEN 2 PRECEDING AND CURRENT ROW), 2) AS media_movil_3m,
    ROUND(ventas / AVG(ventas) OVER (ORDER BY mes ROWS BETWEEN 11 PRECEDING AND CURRENT ROW) * 100, 2) AS indice_estacional
FROM ventas_mensuales
ORDER BY mes DESC;
```

### Ranking de Vendedores con Partición por Sucursal
```sql
SELECT 
    s.nombre AS sucursal,
    v.nombre_vendedor,
    COUNT(f.id_factura) AS ventas,
    SUM(f.total) AS total_vendido,
    ROW_NUMBER() OVER (PARTITION BY s.id_sucursal ORDER BY SUM(f.total) DESC) AS ranking_sucursal,
    RANK() OVER (PARTITION BY s.id_sucursal ORDER BY COUNT(f.id_factura) DESC) AS ranking_cantidad,
    ROUND(SUM(f.total) / SUM(SUM(f.total)) OVER (PARTITION BY s.id_sucursal) * 100, 2) AS participacion_sucursal
FROM sucursales s
JOIN vendedores v ON s.id_sucursal = v.id_sucursal
LEFT JOIN facturas f ON v.id_vendedor = f.id_vendedor AND f.estado = 'activa'
GROUP BY s.id_sucursal, v.id_vendedor
ORDER BY s.nombre, ranking_sucursal;
```

### Diferencia entre Valor Actual y Promedio del Grupo
```sql
SELECT 
    p.nombre_comercial,
    c.nombre AS categoria,
    p.precio_venta,
    ROUND(AVG(p.precio_venta) OVER (PARTITION BY p.id_categoria), 2) AS precio_promedio_categoria,
    ROUND(p.precio_venta - AVG(p.precio_venta) OVER (PARTITION BY p.id_categoria), 2) AS diferencia_vs_promedio,
    ROUND((p.precio_venta - AVG(p.precio_venta) OVER (PARTITION BY p.id_categoria)) 
        / NULLIF(AVG(p.precio_venta) OVER (PARTITION BY p.id_categoria), 0) * 100, 2) AS porcentaje_vs_promedio
FROM productos p
JOIN categorias c ON p.id_categoria = c.id_categoria
WHERE p.activo = TRUE
ORDER BY c.nombre, diferencia_vs_promedio DESC;
```

### Detección de Outliers con Desviación Estándar
```sql
WITH ventas_diarias AS (
    SELECT 
        DATE(fecha_emision) AS dia,
        SUM(total) AS ventas
    FROM facturas
    WHERE YEAR(fecha_emision) = 2024 AND estado = 'activa'
    GROUP BY DATE(fecha_emision)
)
SELECT 
    dia,
    ventas,
    ROUND(AVG(ventas) OVER (), 2) AS media,
    ROUND(STDDEV(ventas) OVER (), 2) AS desviacion,
    ROUND((ventas - AVG(ventas) OVER ()) / NULLIF(STDDEV(ventas) OVER (), 0), 2) AS z_score,
    CASE 
        WHEN ABS((ventas - AVG(ventas) OVER ()) / NULLIF(STDDEV(ventas) OVER (), 0)) > 2 THEN 'OUTLIER'
        ELSE 'NORMAL'
    END AS tipo
FROM ventas_diarias
ORDER BY dia;
```

### Análisis de Retención con Window Functions
```sql
WITH compras_cliente AS (
    SELECT 
        id_cliente,
        fecha_emision,
        LAG(fecha_emision) OVER (PARTITION BY id_cliente ORDER BY fecha_emision) AS fecha_compra_anterior,
        ROW_NUMBER() OVER (PARTITION BY id_cliente ORDER BY fecha_emision) AS numero_compra
    FROM facturas
    WHERE estado = 'activa'
)
SELECT 
    COUNT(DISTINCT id_cliente) AS total_clientes,
    SUM(CASE WHEN numero_compra = 1 THEN 1 ELSE 0 END) AS primera_compra,
    SUM(CASE WHEN numero_compra = 2 THEN 1 ELSE 0 END) AS segunda_compra,
    SUM(CASE WHEN numero_compra = 3 THEN 1 ELSE 0 END) AS tercera_compra,
    SUM(CASE WHEN numero_compra = 4 THEN 1 ELSE 0 END) AS cuarta_compra,
    SUM(CASE WHEN numero_compra >= 5 THEN 1 ELSE 0 END) AS cinco_mas_compras,
    ROUND(SUM(CASE WHEN numero_compra = 2 THEN 1 ELSE 0 END) / NULLIF(SUM(CASE WHEN numero_compra = 1 THEN 1 ELSE 0 END), 0) * 100, 2) AS tasa_recompra
FROM compras_cliente;
```

---

## Anterior: [04 Kardex Y Valorizacion](../05-inventarios/04-kardex-y-valorizacion.md)
## Siguiente: [02 Cte Y Recursividad](../06-sql-avanzado/02-cte-y-recursividad.md)

[Volver al índice](../README.md)
