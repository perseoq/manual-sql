# 9.3 Segmentación de Clientes

## Introducción

La segmentación divide a los clientes en grupos con características similares para personalizar estrategias de marketing, ventas y servicio.

## 1. Análisis RFM (Recency, Frequency, Monetary)

### Cálculo RFM Completo
```sql
WITH rfm_calculos AS (
    SELECT 
        c.id_cliente,
        c.nombre,
        DATEDIFF(CURRENT_DATE, MAX(f.fecha_emision)) AS recencia,
        COUNT(DISTINCT f.id_factura) AS frecuencia,
        SUM(f.total) AS monetario,
        NTILE(5) OVER (ORDER BY DATEDIFF(CURRENT_DATE, MAX(f.fecha_emision)) ASC) AS r_score,
        NTILE(5) OVER (ORDER BY COUNT(DISTINCT f.id_factura) DESC) AS f_score,
        NTILE(5) OVER (ORDER BY SUM(f.total) DESC) AS m_score
    FROM clientes c
    JOIN facturas f ON c.id_cliente = f.id_cliente AND f.estado = 'activa'
    GROUP BY c.id_cliente
)
SELECT 
    id_cliente,
    nombre,
    recencia,
    frecuencia,
    ROUND(monetario, 2) AS monetario,
    r_score, f_score, m_score,
    CONCAT(r_score, f_score, m_score) AS rfm_score,
    CASE 
        WHEN r_score >= 4 AND f_score >= 4 AND m_score >= 4 THEN '⭐ Campeones'
        WHEN r_score >= 4 AND f_score >= 3 AND m_score >= 3 THEN '❤️ Clientes Leales'
        WHEN r_score >= 4 AND f_score <= 2 AND m_score <= 2 THEN '🆕 Nuevos'
        WHEN r_score >= 3 AND f_score >= 3 AND m_score >= 3 THEN '🚀 Potenciales'
        WHEN r_score <= 2 AND f_score >= 3 AND m_score >= 3 THEN '⚠️ En Riesgo'
        WHEN r_score <= 2 AND f_score <= 2 AND m_score >= 3 THEN '🔴 Necesitan Atención'
        WHEN r_score <= 2 AND f_score <= 2 AND m_score <= 2 THEN '💀 Perdidos'
        ELSE '📊 Otros'
    END AS segmento,
    CASE 
        WHEN r_score >= 4 AND f_score >= 4 AND m_score >= 4 THEN 1
        WHEN r_score >= 4 AND f_score >= 3 AND m_score >= 3 THEN 2
        WHEN r_score >= 4 AND f_score <= 2 AND m_score <= 2 THEN 3
        WHEN r_score >= 3 AND f_score >= 3 AND m_score >= 3 THEN 4
        WHEN r_score <= 2 AND f_score >= 3 AND m_score >= 3 THEN 5
        WHEN r_score <= 2 AND f_score <= 2 AND m_score >= 3 THEN 6
        WHEN r_score <= 2 AND f_score <= 2 AND m_score <= 2 THEN 7
        ELSE 8
    END AS prioridad
FROM rfm_calculos
ORDER BY prioridad, monetario DESC;
```

### Resumen de Segmentos RFM
```sql
WITH rfm AS (
    SELECT 
        c.id_cliente,
        DATEDIFF(CURRENT_DATE, MAX(f.fecha_emision)) AS recencia,
        COUNT(DISTINCT f.id_factura) AS frecuencia,
        SUM(f.total) AS monetario,
        NTILE(5) OVER (ORDER BY DATEDIFF(CURRENT_DATE, MAX(f.fecha_emision)) ASC) AS r,
        NTILE(5) OVER (ORDER BY COUNT(DISTINCT f.id_factura) DESC) AS f,
        NTILE(5) OVER (ORDER BY SUM(f.total) DESC) AS m
    FROM clientes c
    JOIN facturas f ON c.id_cliente = f.id_cliente AND f.estado = 'activa'
    GROUP BY c.id_cliente
)
SELECT 
    CASE 
        WHEN r >= 4 AND f >= 4 AND m >= 4 THEN 'Campeones'
        WHEN r >= 4 AND f >= 3 AND m >= 3 THEN 'Leales'
        WHEN r >= 4 AND f <= 2 AND m <= 2 THEN 'Nuevos'
        WHEN r >= 3 AND f >= 3 AND m >= 3 THEN 'Potenciales'
        WHEN r <= 2 AND f >= 3 AND m >= 3 THEN 'En Riesgo'
        WHEN r <= 2 AND f <= 2 AND m >= 3 THEN 'Atención'
        WHEN r <= 2 AND f <= 2 AND m <= 2 THEN 'Perdidos'
        ELSE 'Otros'
    END AS segmento,
    COUNT(*) AS clientes,
    ROUND(AVG(monetario), 2) AS gasto_promedio,
    ROUND(SUM(monetario), 2) AS ingresos_totales,
    ROUND(SUM(monetario) / SUM(SUM(monetario)) OVER () * 100, 2) AS porcentaje_ingresos,
    ROUND(AVG(recencia), 0) AS recencia_promedio,
    ROUND(AVG(frecuencia), 1) AS frecuencia_promedio
FROM rfm
GROUP BY segmento
ORDER BY ingresos_totales DESC;
```

## 2. Segmentación por Valor de Vida (CLV)

```sql
WITH clv AS (
    SELECT 
        c.id_cliente,
        c.nombre,
        COUNT(f.id_factura) AS num_compras,
        SUM(f.total) AS valor_total,
        AVG(f.total) AS ticket_promedio,
        DATEDIFF(CURRENT_DATE, MIN(f.fecha_emision)) AS antiguedad_dias,
        SUM(f.total) / NULLIF(DATEDIFF(CURRENT_DATE, MIN(f.fecha_emision)), 0) * 365 AS clv_anual_estimado,
        COUNT(f.id_factura) / NULLIF(DATEDIFF(CURRENT_DATE, MIN(f.fecha_emision)), 0) * 365 AS compras_anuales_estimadas
    FROM clientes c
    JOIN facturas f ON c.id_cliente = f.id_cliente AND f.estado = 'activa'
    GROUP BY c.id_cliente
    HAVING DATEDIFF(CURRENT_DATE, MIN(f.fecha_emision)) > 30
)
SELECT 
    nombre,
    num_compras,
    ROUND(valor_total, 2) AS valor_total,
    ROUND(ticket_promedio, 2) AS ticket_promedio,
    ROUND(antiguedad_dias / 30, 1) AS antiguedad_meses,
    ROUND(clv_anual_estimado, 2) AS clv_anual,
    NTILE(4) OVER (ORDER BY clv_anual_estimado DESC) AS cuartil_valor,
    CASE 
        WHEN clv_anual_estimado > 100000 THEN 'VIP'
        WHEN clv_anual_estimado > 50000 THEN 'Premium'
        WHEN clv_anual_estimado > 10000 THEN 'Medio'
        ELSE 'Bajo'
    END AS categoria_clv
FROM clv
ORDER BY clv_anual_estimado DESC;
```

## 3. Segmentación por Comportamiento

### Por Canal de Compra
```sql
SELECT 
    c.id_cliente,
    c.nombre,
    SUM(CASE WHEN f.metodo_pago = 'efectivo' THEN f.total ELSE 0 END) AS compras_efectivo,
    SUM(CASE WHEN f.metodo_pago = 'tarjeta_credito' THEN f.total ELSE 0 END) AS compras_tc,
    SUM(CASE WHEN f.metodo_pago = 'tarjeta_debito' THEN f.total ELSE 0 END) AS compras_td,
    SUM(CASE WHEN f.metodo_pago = 'transferencia' THEN f.total ELSE 0 END) AS compras_transferencia,
    SUM(CASE WHEN f.metodo_pago = 'credito' THEN f.total ELSE 0 END) AS compras_credito,
    CASE 
        WHEN SUM(CASE WHEN f.metodo_pago = 'efectivo' THEN f.total ELSE 0 END) > 
             SUM(f.total) * 0.7 THEN 'Efectivo'
        WHEN SUM(CASE WHEN f.metodo_pago IN ('tarjeta_credito', 'tarjeta_debito') THEN f.total ELSE 0 END) > 
             SUM(f.total) * 0.7 THEN 'Tarjeta'
        WHEN SUM(CASE WHEN f.metodo_pago = 'transferencia' THEN f.total ELSE 0 END) > 
             SUM(f.total) * 0.7 THEN 'Transferencia'
        WHEN SUM(CASE WHEN f.metodo_pago = 'credito' THEN f.total ELSE 0 END) > 
             SUM(f.total) * 0.7 THEN 'Crédito'
        ELSE 'Mixto'
    END AS metodo_preferido
FROM clientes c
JOIN facturas f ON c.id_cliente = f.id_cliente AND f.estado = 'activa'
GROUP BY c.id_cliente;
```

### Por Categoría de Producto
```sql
SELECT 
    c.id_cliente,
    c.nombre,
    SUM(CASE WHEN cat.nombre = 'Electrónicos' THEN fd.total ELSE 0 END) AS electronica,
    SUM(CASE WHEN cat.nombre = 'Ropa' THEN fd.total ELSE 0 END) AS ropa,
    SUM(CASE WHEN cat.nombre = 'Hogar' THEN fd.total ELSE 0 END) AS hogar,
    SUM(CASE WHEN cat.nombre = 'Alimentos' THEN fd.total ELSE 0 END) AS alimentos,
    CASE 
        WHEN SUM(CASE WHEN cat.nombre = 'Electrónicos' THEN fd.total ELSE 0 END) > 
             SUM(fd.total) * 0.5 THEN 'Tecnología'
        WHEN SUM(CASE WHEN cat.nombre = 'Ropa' THEN fd.total ELSE 0 END) > 
             SUM(fd.total) * 0.5 THEN 'Moda'
        WHEN SUM(CASE WHEN cat.nombre = 'Hogar' THEN fd.total ELSE 0 END) > 
             SUM(fd.total) * 0.5 THEN 'Hogar'
        WHEN SUM(CASE WHEN cat.nombre = 'Alimentos' THEN fd.total ELSE 0 END) > 
             SUM(fd.total) * 0.5 THEN 'Alimentos'
        ELSE 'Generalista'
    END AS perfil_compra
FROM clientes c
JOIN facturas f ON c.id_cliente = f.id_cliente AND f.estado = 'activa'
JOIN facturas_detalle fd ON f.id_factura = fd.id_factura
JOIN productos p ON fd.id_producto = p.id_producto
JOIN categorias cat ON p.id_categoria = cat.id_categoria
GROUP BY c.id_cliente;
```

## 4. Análisis de Cohortes

### Cohortes por Mes de Primera Compra
```sql
WITH cohortes AS (
    SELECT 
        c.id_cliente,
        DATE_FORMAT(MIN(f.fecha_emision), '%Y-%m') AS cohorte_mes,
        TIMESTAMPDIFF(MONTH, MIN(f.fecha_emision), f.fecha_emision) AS mes_desde_inicio,
        f.total
    FROM clientes c
    JOIN facturas f ON c.id_cliente = f.id_cliente AND f.estado = 'activa'
    GROUP BY c.id_cliente, f.id_factura, f.fecha_emision, f.total
)
SELECT 
    cohorte_mes,
    COUNT(DISTINCT CASE WHEN mes_desde_inicio = 0 THEN id_cliente END) AS mes_0,
    COUNT(DISTINCT CASE WHEN mes_desde_inicio = 1 THEN id_cliente END) AS mes_1,
    COUNT(DISTINCT CASE WHEN mes_desde_inicio = 2 THEN id_cliente END) AS mes_2,
    COUNT(DISTINCT CASE WHEN mes_desde_inicio = 3 THEN id_cliente END) AS mes_3,
    COUNT(DISTINCT CASE WHEN mes_desde_inicio = 4 THEN id_cliente END) AS mes_4,
    COUNT(DISTINCT CASE WHEN mes_desde_inicio = 5 THEN id_cliente END) AS mes_5
FROM cohortes
GROUP BY cohorte_mes
ORDER BY cohorte_mes;
```

## 5. Clientes en Riesgo de Abandono

```sql
WITH compras_cliente AS (
    SELECT 
        c.id_cliente,
        c.nombre,
        c.email,
        MAX(f.fecha_emision) AS ultima_compra,
        COUNT(f.id_factura) AS total_compras,
        SUM(f.total) AS total_gastado,
        AVG(DATEDIFF(f.fecha_emision, LAG(f.fecha_emision) OVER (
            PARTITION BY c.id_cliente ORDER BY f.fecha_emision
        ))) AS intervalo_promedio_dias
    FROM clientes c
    JOIN facturas f ON c.id_cliente = f.id_cliente AND f.estado = 'activa'
    GROUP BY c.id_cliente
)
SELECT 
    nombre,
    email,
    DATEDIFF(CURRENT_DATE, ultima_compra) AS dias_sin_compra,
    total_compras,
    ROUND(total_gastado, 2) AS total_gastado,
    ROUND(COALESCE(intervalo_promedio_dias, 0), 0) AS intervalo_promedio,
    CASE 
        WHEN DATEDIFF(CURRENT_DATE, ultima_compra) > COALESCE(intervalo_promedio_dias * 3, 180) 
             AND ultima_compra < DATE_SUB(CURRENT_DATE, INTERVAL 90 DAY) THEN '🔴 ALTO RIESGO'
        WHEN DATEDIFF(CURRENT_DATE, ultima_compra) > COALESCE(intervalo_promedio_dias * 2, 90) 
             THEN '🟡 RIESGO MEDIO'
        WHEN DATEDIFF(CURRENT_DATE, ultima_compra) > COALESCE(intervalo_promedio_dias, 45) 
             THEN '🟢 BAJO RIESGO'
        ELSE '✅ ACTIVO'
    END AS riesgo_abandono
FROM compras_cliente
HAVING riesgo_abandono IN ('🔴 ALTO RIESGO', '🟡 RIESGO MEDIO')
ORDER BY dias_sin_compra DESC;
```

## Dashboard de Segmentación

```sql
-- Resumen ejecutivo de segmentos
SELECT 
    segmento,
    COUNT(*) AS clientes,
    ROUND(AVG(ingresos), 2) AS ingreso_promedio,
    ROUND(SUM(ingresos), 2) AS ingreso_total,
    ROUND(SUM(ingresos) / SUM(SUM(ingresos)) OVER () * 100, 2) AS porcentaje_ingresos,
    ROUND(AVG(frecuencia), 1) AS frecuencia_promedio,
    ROUND(AVG(recencia), 0) AS recencia_promedio
FROM (
    SELECT 
        c.id_cliente,
        CASE 
            WHEN r >= 4 AND f >= 4 AND m >= 4 THEN 'Campeones'
            WHEN r >= 4 AND f >= 3 AND m >= 3 THEN 'Leales'
            ELSE 'Regular'
        END AS segmento,
        monetario AS ingresos,
        frecuencia,
        recencia
    FROM (
        SELECT 
            c.id_cliente,
            DATEDIFF(CURRENT_DATE, MAX(f.fecha_emision)) AS recencia,
            COUNT(DISTINCT f.id_factura) AS frecuencia,
            SUM(f.total) AS monetario,
            NTILE(5) OVER (ORDER BY DATEDIFF(CURRENT_DATE, MAX(f.fecha_emision)) ASC) AS r,
            NTILE(5) OVER (ORDER BY COUNT(DISTINCT f.id_factura) DESC) AS f,
            NTILE(5) OVER (ORDER BY SUM(f.total) DESC) AS m
        FROM clientes c
        JOIN facturas f ON c.id_cliente = f.id_cliente AND f.estado = 'activa'
        GROUP BY c.id_cliente
    ) AS rfm_sub
) AS segmentos
GROUP BY segmento
ORDER BY ingreso_total DESC;
```

---

## Anterior: [02 Series Temporales](../09-analisis-datos/02-series-temporales.md)
## Siguiente: [04 Dashboards Sql](../09-analisis-datos/04-dashboards-sql.md)

[Volver al índice](../README.md)
