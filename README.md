# Manual Completo de SQL para Ventas, Compras e Inventarios

## SQL, SQL Injection, Análisis de Datos y Hacking de Bases de Datos

---

## Índice General

### [1. Introducción](./01-introduccion/)

| Archivo | Descripción |
|---------|-------------|
| [01-que-es-sql.md](./01-introduccion/01-que-es-sql.md) | Qué es SQL, historia, estándares y motores de base de datos |
| [02-tipos-de-bases-de-datos.md](./01-introduccion/02-tipos-de-bases-de-datos.md) | Bases de datos relacionales vs NoSQL, motores populares |
| [03-configuracion-entorno.md](./01-introduccion/03-configuracion-entorno.md) | Instalación y configuración de MySQL, PostgreSQL, SQL Server |

### [2. Fundamentos de SQL](./02-fundamentos/)

| Archivo | Descripción |
|---------|-------------|
| [01-select-basico.md](./02-fundamentos/01-select-basico.md) | SELECT, columnas, alias, DISTINCT, LIMIT |
| [02-where-y-filtros.md](./02-fundamentos/02-where-y-filtros.md) | WHERE, operadores, LIKE, IN, BETWEEN, NULL |
| [03-join-y-relaciones.md](./02-fundamentos/03-join-y-relaciones.md) | INNER, LEFT, RIGHT, FULL JOIN, self-join |
| [04-agrupacion-y-agregacion.md](./02-fundamentos/04-agrupacion-y-agregacion.md) | GROUP BY, HAVING, COUNT, SUM, AVG, MIN, MAX |
| [05-subconsultas.md](./02-fundamentos/05-subconsultas.md) | Subconsultas escalares, de fila, de tabla, correlacionadas |
| [06-insert-update-delete.md](./02-fundamentos/06-insert-update-delete.md) | DML: INSERT, UPDATE, DELETE, MERGE |
| [07-create-table-y-ddl.md](./02-fundamentos/07-create-table-y-ddl.md) | DDL: CREATE, ALTER, DROP, constraints, relaciones |
| [08-indices-y-optimizacion.md](./02-fundamentos/08-indices-y-optimizacion.md) | Índices, planes de ejecución, optimización básica |

### [3. Ventas](./03-ventas/)

| Archivo | Descripción |
|---------|-------------|
| [01-modelo-datos-ventas.md](./03-ventas/01-modelo-datos-ventas.md) | Modelo relacional completo para un sistema de ventas |
| [02-consultas-ventas.md](./03-ventas/02-consultas-ventas.md) | Consultas fundamentales para operaciones de venta |
| [03-analisis-ventas.md](./03-ventas/03-analisis-ventas.md) | Análisis de tendencias, estacionalidad y rendimiento |
| [04-reportes-ventas.md](./03-ventas/04-reportes-ventas.md) | Reportes gerenciales, KPIs y dashboards |

### [4. Compras](./04-compras/)

| Archivo | Descripción |
|---------|-------------|
| [01-modelo-datos-compras.md](./04-compras/01-modelo-datos-compras.md) | Modelo relacional completo para un sistema de compras |
| [02-consultas-compras.md](./04-compras/02-consultas-compras.md) | Consultas para órdenes de compra, recepción y proveedores |
| [03-analisis-compras.md](./04-compras/03-analisis-compras.md) | Análisis de gastos, proveedores y eficiencia |

### [5. Inventarios](./05-inventarios/)

| Archivo | Descripción |
|---------|-------------|
| [01-modelo-datos-inventarios.md](./05-inventarios/01-modelo-datos-inventarios.md) | Modelo relacional completo para gestión de inventarios |
| [02-consultas-inventarios.md](./05-inventarios/02-consultas-inventarios.md) | Consultas de stock, movimientos y ubicaciones |
| [03-analisis-inventarios.md](./05-inventarios/03-analisis-inventarios.md) | Rotación, ABC, obsolescencia y valorización |
| [04-kardex-y-valorizacion.md](./05-inventarios/04-kardex-y-valorizacion.md) | Kardex valorizado, UEPS, PEPS, promedio ponderado |

### [6. SQL Avanzado](./06-sql-avanzado/)

| Archivo | Descripción |
|---------|-------------|
| [01-window-functions.md](./06-sql-avanzado/01-window-functions.md) | ROW_NUMBER, RANK, LAG, LEAD, SUM OVER, particiones |
| [02-cte-y-recursividad.md](./06-sql-avanzado/02-cte-y-recursividad.md) | Common Table Expressions, consultas recursivas |
| [03-store-procedures.md](./06-sql-avanzado/03-store-procedures.md) | Procedimientos almacenados, parámetros, lógica de negocio |
| [04-triggers.md](./06-sql-avanzado/04-triggers.md) | Triggers BEFORE/AFTER, INSTEAD OF, auditoría automática |
| [05-transacciones.md](./06-sql-avanzado/05-transacciones.md) | ACID, COMMIT, ROLLBACK, SAVEPOINT, aislamiento |
| [06-performance-tuning.md](./06-sql-avanzado/06-performance-tuning.md) | EXPLAIN, query tuning, estadísticas, particionamiento |

### [7. SQL Injection](./07-sql-injection/)

| Archivo | Descripción |
|---------|-------------|
| [01-que-es-sql-injection.md](./07-sql-injection/01-que-es-sql-injection.md) | Definición, historia, impacto en el negocio |
| [02-tipos-de-inyeccion.md](./07-sql-injection/02-tipos-de-inyeccion.md) | In-band, Blind, Out-of-band, Second-order |
| [03-deteccion-y-exploit.md](./07-sql-injection/03-deteccion-y-exploit.md) | Técnicas de detección, herramientas de explotación |
| [04-prevencion.md](./07-sql-injection/04-prevencion.md) | Prepared statements, ORM, WAF, parametrización |
| [05-ejemplos-practicos-ventas.md](./07-sql-injection/05-ejemplos-practicos-ventas.md) | Casos reales en sistemas de ventas, compras e inventarios |

### [8. Seguridad de Bases de Datos](./08-seguridad/)

| Archivo | Descripción |
|---------|-------------|
| [01-hardening-bases-datos.md](./08-seguridad/01-hardening-bases-datos.md) | Endurecimiento, configuración segura, parches |
| [02-encriptacion.md](./08-seguridad/02-encriptacion.md) | Encriptación en reposo, en tránsito, TDE, hashing |
| [03-auditoria.md](./08-seguridad/03-auditoria.md) | Logs de auditoría, seguimiento de cambios, forense |
| [04-backup-y-recovery.md](./08-seguridad/04-backup-y-recovery.md) | Estrategias de backup, point-in-time recovery, DRP |

### [9. Análisis de Datos con SQL](./09-analisis-datos/)

| Archivo | Descripción |
|---------|-------------|
| [01-analisis-exploratorio.md](./09-analisis-datos/01-analisis-exploratorio.md) | EDA con SQL, perfiles de datos, detección de anomalías |
| [02-series-temporales.md](./09-analisis-datos/02-series-temporales.md) | Análisis temporal, tendencias, estacionalidad, forecasting |
| [03-segmentacion-clientes.md](./09-analisis-datos/03-segmentacion-clientes.md) | RFM, clustering, cohortes, customer analytics |
| [04-dashboards-sql.md](./09-analisis-datos/04-dashboards-sql.md) | SQL para dashboards, métricas, KPI, visualización |

### [10. Apéndices](./10-apendices/)

| Archivo | Descripción |
|---------|-------------|
| [01-glosario.md](./10-apendices/01-glosario.md) | Glosario completo de términos SQL y seguridad |
| [02-hojas-de-trucos.md](./10-apendices/02-hojas-de-trucos.md) | Cheat sheets: funciones, comandos, sintaxis rápida |
| [03-recursos-adicionales.md](./10-apendices/03-recursos-adicionales.md) | Libros, cursos, herramientas, comunidades |

---

## Navegación Rápida

| Sección | Enlace |
|---------|--------|
| 📘 Introducción | [Inicio](./01-introduccion/01-que-es-sql.md) |
| 🧱 Fundamentos | [SELECT](./02-fundamentos/01-select-basico.md) |
| 💰 Ventas | [Modelo](./03-ventas/01-modelo-datos-ventas.md) |
| 📦 Compras | [Modelo](./04-compras/01-modelo-datos-compras.md) |
| 🏭 Inventarios | [Modelo](./05-inventarios/01-modelo-datos-inventarios.md) |
| ⚡ SQL Avanzado | [Window Functions](./06-sql-avanzado/01-window-functions.md) |
| 🔓 SQL Injection | [Introducción](./07-sql-injection/01-que-es-sql-injection.md) |
| 🔒 Seguridad | [Hardening](./08-seguridad/01-hardening-bases-datos.md) |
| 📊 Análisis | [EDA](./09-analisis-datos/01-analisis-exploratorio.md) |
| 📚 Apéndices | [Glosario](./10-apendices/01-glosario.md) |

---

## Convenciones Usadas en Este Manual

```sql
-- Ejemplo de código SQL
SELECT * FROM clientes;
```

> **Nota:** Notas importantes y advertencias aparecen en bloques como este.

🔴 **Advertencia:** Bloques rojos indican peligros de seguridad.

✅ **Mejor práctica:** Recomendaciones y buenas prácticas.

📊 **Ejemplo del mundo real:** Ejemplos aplicados a ventas, compras e inventarios.

---

## Mapas de Aprendizaje Sugeridos

### Ruta para Principiantes
1. Introducción (01 → 03)
2. Fundamentos (01 → 08)
3. Modelos de datos (Ventas, Compras, Inventarios)
4. Consultas básicas por módulo

### Ruta para Desarrolladores
1. Fundamentos (rápido)
2. SQL Avanzado completo
3. Seguridad y SQL Injection
4. Performance Tuning

### Ruta para Analistas de Datos
1. Fundamentos (01 → 06)
2. Ventas, Compras, Inventarios (análisis)
3. SQL Avanzado (Window Functions, CTE)
4. Análisis de Datos completo

### Ruta para Auditores/Seguridad
1. Fundamentos (01 → 05)
2. SQL Injection completo
3. Seguridad completo
4. Performance Tuning

---

*Manual creado con fines educativos. El contenido sobre SQL injection y hacking debe usarse exclusivamente en entornos autorizados.*
