# 9.2 Series Temporales con SQL

## Introducción

El análisis de series temporales permite identificar patrones, tendencias y estacionalidad en datos a lo largo del tiempo.

## 1. Preparación de Datos Temporales

### Generar Secuencias de Fechas
```sql
-- Completar días faltantes (útil para reporting)
WITH RECURSIVE fechas AS (
    SELECT '2024-01-01' AS fecha
    UNION ALL
    SELECT DATE_ADD(fecha, INTERVAL 1 DAY)
    FROM fechas
    WHERE fecha < '2024-12-31'
)
SELECT 
    f.fecha,
    COALESCE(SUM(v.total), 0) AS ventas,
    COALESCE(COUNT(v.id_factura), 0) AS transacciones
FROM fechas f
LEFT JOIN facturas v ON DATE(v.fecha_emision) = f.fecha 
    AND v.estado = 'activa'
GROUP BY f.fecha
ORDER BY f.fecha;
```

### Crear Tabla de Calendario
```sql
CREATE TABLE calendario (
    fecha DATE PRIMARY KEY,
    dia INT,
    mes INT,
    año INT,
    dia_semana INT,
    nombre_dia VARCHAR(20),
    nombre_mes VARCHAR(20),
    semana_año INT,
    dia_año INT,
    es_fin_semana BOOLEAN,
    es_festivo BOOLEAN DEFAULT FALSE
);

-- Poblar calendario
INSERT INTO calendario (fecha, dia, mes, año, dia_semana, nombre_dia, 
                        nombre_mes, semana_año, dia_año, es_fin_semana)
SELECT 
    fecha,
    DAY(fecha),
    MONTH(fecha),
    YEAR(fecha),
    DAYOFWEEK(fecha),
    DAYNAME(fecha),
    MONTHNAME(fecha),
    WEEK(fecha),
    DAYOFYEAR(fecha),
    DAYOFWEEK(fecha) IN (1, 7)
FROM (
    SELECT DATE_ADD('2020-01-01', INTERVAL n DAY) AS fecha
    FROM (SELECT @rownum := @rownum + 1 AS n 
          FROM facturas, (SELECT @rownum := -1) r 
          LIMIT 3653) AS nums
) AS fechas;
```

## 2. Tendencias

### Tendencia Lineal con Regresión
```sql
WITH ventas_diarias AS (
    SELECT 
        DATE(fecha_emision) AS dia,
        SUM(total) AS ventas,
        DATEDIFF(DATE(fecha_emision), '2024-01-01') AS x
    FROM facturas
    WHERE YEAR(fecha_emision) = 2024 AND estado = 'activa'
    GROUP BY DATE(fecha_emision)
),
regresion AS (
    SELECT 
        COUNT(*) AS n,
        SUM(x) AS sum_x,
        SUM(ventas) AS sum_y,
        SUM(x * ventas) AS sum_xy,
        SUM(x * x) AS sum_xx
    FROM ventas_diarias
)
SELECT 
    'Pendiente' AS parametro,
    ROUND((n * sum_xy - sum_x * sum_y) / (n * sum_xx - sum_x * sum_x), 4) AS valor
FROM regresion

UNION ALL

SELECT 
    'Intersección',
    ROUND((sum_y - ((n * sum_xy - sum_x * sum_y) / (n * sum_xx - sum_x * sum_x)) * sum_x) / n, 2)
FROM regresion

UNION ALL

SELECT 
    'Tendencia diaria ($)',
    ROUND((n * sum_xy - sum_x * sum_y) / (n * sum_xx - sum_x * sum_x), 2)
FROM regresion;
```

### Crecimiento Compuesto
```sql
WITH ventas_mensuales AS (
    SELECT 
        DATE_FORMAT(fecha_emision, '%Y-%m') AS mes,
        SUM(total) AS ventas
    FROM facturas
    WHERE YEAR(fecha_emision) = 2024 AND estado = 'activa'
    GROUP BY DATE_FORMAT(fecha_emision, '%Y-%m')
),
crecimiento AS (
    SELECT 
        mes,
        ventas,
        LAG(ventas) OVER (ORDER BY mes) AS mes_anterior,
        ROUND((ventas - LAG(ventas) OVER (ORDER BY mes)) / 
              NULLIF(LAG(ventas) OVER (ORDER BY mes), 0) * 100, 2) AS crecimiento_mensual
    FROM ventas_mensuales
)
SELECT 
    'Crecimiento Promedio Mensual' AS metrica,
    CONCAT(ROUND(AVG(crecimiento_mensual), 2), '%') AS valor
FROM crecimiento
WHERE crecimiento_mensual IS NOT NULL

UNION ALL

SELECT 
    'Crecimiento Total Anual',
    CONCAT(ROUND(
        (SELECT ventas FROM ventas_mensuales ORDER BY mes DESC LIMIT 1) /
        NULLIF((SELECT ventas FROM ventas_mensuales ORDER BY mes LIMIT 1), 0) * 100 - 100, 2
    ), '%')

UNION ALL

SELECT 
    'Proyección Mensual Siguiente',
    ROUND((SELECT ventas FROM ventas_mensuales ORDER BY mes DESC LIMIT 1) *
          (1 + AVG(crecimiento_mensual) / 100), 2)
FROM crecimiento
WHERE crecimiento_mensual IS NOT NULL;
```

## 3. Estacionalidad

### Patrón por Día de Semana
```sql
SELECT 
    c.nombre_dia,
    COUNT(v.id_factura) AS facturas,
    SUM(v.total) AS ingresos,
    ROUND(AVG(v.total), 2) AS ticket_promedio,
    ROUND(SUM(v.total) / SUM(SUM(v.total)) OVER () * 100, 2) AS porcentaje_semanal,
    ROUND(AVG(SUM(v.total)) OVER (ORDER BY c.dia_semana ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING), 2) AS promedio_semanal,
    ROUND(SUM(v.total) / AVG(SUM(v.total)) OVER (ORDER BY c.dia_semana ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) * 100, 2) AS indice_estacional
FROM calendario c
LEFT JOIN facturas v ON c.fecha = DATE(v.fecha_emision) AND v.estado = 'activa'
WHERE c.fecha BETWEEN '2024-01-01' AND '2024-12-31'
GROUP BY c.nombre_dia, c.dia_semana
ORDER BY c.dia_semana;
```

### Estacionalidad por Hora
```sql
SELECT 
    HOUR(fecha_emision) AS hora,
    COUNT(*) AS transacciones,
    SUM(total) AS ingresos,
    AVG(total) AS ticket_promedio,
    ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER(), 2) AS porcentaje_diario
FROM facturas
WHERE YEAR(fecha_emision) = 2024 AND estado = 'activa'
GROUP BY HOUR(fecha_emision)
ORDER BY HOUR(fecha_emision);
```

### Estacionalidad Mensual
```sql
WITH ventas_mensuales AS (
    SELECT 
        MONTH(fecha_emision) AS mes_num,
        SUM(total) AS ventas
    FROM facturas
    WHERE YEAR(fecha_emision) = 2024 AND estado = 'activa'
    GROUP BY MONTH(fecha_emision)
)
SELECT 
    mes_num,
    MONTHNAME(CONCAT('2024-', mes_num, '-01')) AS mes,
    ventas,
    ROUND(AVG(ventas) OVER (), 2) AS promedio_general,
    ROUND(ventas / AVG(ventas) OVER () * 100, 2) AS indice_estacional,
    CASE 
        WHEN ventas > AVG(ventas) OVER () * 1.2 THEN 'ALTA 📈'
        WHEN ventas < AVG(ventas) OVER () * 0.8 THEN 'BAJA 📉'
        ELSE 'NORMAL'
    END AS temporada
FROM ventas_mensuales
ORDER BY mes_num;
```

## 4. Medias Móviles

```sql
-- Media móvil de 7 y 30 días
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
    ROUND(AVG(ventas) OVER (ORDER BY dia ROWS BETWEEN 6 PRECEDING AND CURRENT ROW), 2) AS media_movil_7d,
    ROUND(AVG(ventas) OVER (ORDER BY dia ROWS BETWEEN 29 PRECEDING AND CURRENT ROW), 2) AS media_movil_30d,
    ROUND(ventas - AVG(ventas) OVER (ORDER BY dia ROWS BETWEEN 6 PRECEDING AND CURRENT ROW), 2) AS desviacion_7d,
    CASE 
        WHEN ABS(ventas - AVG(ventas) OVER (ORDER BY dia ROWS BETWEEN 6 PRECEDING AND CURRENT ROW)) > 
             STDDEV(ventas) OVER (ORDER BY dia ROWS BETWEEN 29 PRECEDING AND CURRENT ROW) * 2
        THEN 'ANOMALÍA'
        ELSE 'Normal'
    END AS alerta
FROM ventas_diarias
ORDER BY dia DESC;
```

## 5. Comparación Año vs Año

```sql
SELECT 
    MONTH(f.fecha_emision) AS mes,
    SUM(CASE WHEN YEAR(f.fecha_emision) = 2024 THEN f.total ELSE 0 END) AS ingresos_2024,
    SUM(CASE WHEN YEAR(f.fecha_emision) = 2023 THEN f.total ELSE 0 END) AS ingresos_2023,
    SUM(CASE WHEN YEAR(f.fecha_emision) = 2024 THEN f.total ELSE 0 END) - 
    SUM(CASE WHEN YEAR(f.fecha_emision) = 2023 THEN f.total ELSE 0 END) AS diferencia,
    ROUND(
        (SUM(CASE WHEN YEAR(f.fecha_emision) = 2024 THEN f.total ELSE 0 END) -
         SUM(CASE WHEN YEAR(f.fecha_emision) = 2023 THEN f.total ELSE 0 END))
        / NULLIF(SUM(CASE WHEN YEAR(f.fecha_emision) = 2023 THEN f.total ELSE 0 END), 0) * 100, 2
    ) AS crecimiento_yoy
FROM facturas f
WHERE YEAR(f.fecha_emision) IN (2023, 2024) AND f.estado = 'activa'
GROUP BY MONTH(f.fecha_emision)
ORDER BY mes;
```

## 6. Forecasting Simple

### Proyección con Media Móvil
```sql
WITH ultimos_30_dias AS (
    SELECT 
        DATE(fecha_emision) AS dia,
        SUM(total) AS ventas
    FROM facturas
    WHERE estado = 'activa'
      AND fecha_emision >= DATE_SUB(CURRENT_DATE, INTERVAL 30 DAY)
    GROUP BY DATE(fecha_emision)
)
SELECT 
    'Pronóstico próxima semana' AS tipo,
    ROUND(AVG(ventas) * 7, 2) AS valor
FROM ultimos_30_dias

UNION ALL

SELECT 
    'Pronóstico próximo mes',
    ROUND(AVG(ventas) * 30, 2)
FROM ultimos_30_dias;
```

### Proyección por Estacionalidad
```sql
WITH promedios_mensuales AS (
    SELECT 
        MONTH(fecha_emision) AS mes,
        AVG(total) AS promedio_venta_diaria
    FROM facturas
    WHERE YEAR(fecha_emision) = 2024 AND estado = 'activa'
    GROUP BY MONTH(fecha_emision)
)
SELECT 
    mes,
    MONTHNAME(CONCAT('2024-', mes, '-01')) AS nombre_mes,
    ROUND(promedio_venta_diaria, 2) AS promedio_diario_historico,
    ROUND(promedio_venta_diaria * DAY(LAST_DAY(CONCAT('2025-', mes, '-01'))), 2) AS proyeccion_mensual_2025
FROM promedios_mensuales
ORDER BY mes;
```

## Dashboard de Series Temporales

```sql
-- KPIs temporales
SELECT 
    CURRENT_DATE AS fecha_reporte,
    (SELECT SUM(total) FROM facturas WHERE DATE(fecha_emision) = CURRENT_DATE AND estado = 'activa') AS ventas_hoy,
    (SELECT AVG(SUM(total)) OVER (ORDER BY DATE(fecha_emision) ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) 
     FROM facturas WHERE DATE(fecha_emision) <= CURRENT_DATE AND estado = 'activa'
     GROUP BY DATE(fecha_emision) LIMIT 1) AS media_movil_7d,
    (SELECT SUM(total) FROM facturas WHERE YEAR(fecha_emision) = YEAR(CURRENT_DATE) AND YEAR(fecha_emision) = YEAR(CURRENT_DATE - INTERVAL 1 YEAR) AND estado = 'activa') AS ingresos_yoy;
```

---

## Anterior: [01 Analisis Exploratorio](../09-analisis-datos/01-analisis-exploratorio.md)
## Siguiente: [03 Segmentacion Clientes](../09-analisis-datos/03-segmentacion-clientes.md)

[Volver al índice](../README.md)
