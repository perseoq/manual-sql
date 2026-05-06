# 2.8 Índices y Optimización

## ¿Qué es un Índice?

Imagina que tienes un libro de 1,000 páginas y necesitas encontrar todas las referencias a "SQL". Sin un índice, tendrías que hojear página por página hasta encontrar lo que buscas. Eso es un FULL TABLE SCAN: leer toda la tabla de principio a fin.

Con un índice, buscas "SQL" en el índice del libro (que está ordenado alfabéticamente), encuentras "SQL → páginas 45, 123, 456", y vas directamente a esas páginas. Eso es un INDEX SEEK: usar el índice para encontrar los datos directamente.

En bases de datos, un índice es una estructura de datos (normalmente un B-Tree, un árbol balanceado) que almacena los valores de una o más columnas en orden, junto con punteros a las filas correspondientes. Esto permite que las búsquedas sean DRÁSTICAMENTE más rápidas.

El ejemplo muestra la diferencia: sin índice, una búsqueda por folio en una tabla de 1 millón de facturas examina TODAS las filas (800ms). Con un índice, encuentra la fila directamente (~2ms). Eso es 400 VECES más rápido.

Pero los índices no son gratis: cada índice ocupa espacio en disco (a veces más que la tabla misma) y RALENTIZA las operaciones de escritura (INSERT, UPDATE, DELETE) porque hay que mantener el índice actualizado. Es un balance entre velocidad de lectura y velocidad de escritura.

```sql
-- Sin índice (Full Table Scan)
-- MySQL tiene que revisar TODAS las filas
SELECT * FROM facturas WHERE folio = 'F-2024-5000';
-- Examina: 1,000,000 filas
-- Tiempo: ~800ms

-- Con índice (Index Seek)
-- Crear índice
CREATE INDEX idx_folio ON facturas(folio);

-- Ahora MySQL encuentra la fila directamente
SELECT * FROM facturas WHERE folio = 'F-2024-5000';
-- Examina: 1 fila (a través del índice)
-- Tiempo: ~2ms
```

## Tipos de Índices

### B-Tree (Balance Tree) - El más común

El B-Tree es el tipo de índice por defecto en casi todos los motores de base de datos (MySQL/PostgreSQL/SQL Server). Estructuralmente, es un árbol donde los nodos hoja contienen los valores de las columnas indexadas y punteros a las filas.

¿Por qué es tan útil? Porque soporta múltiples tipos de búsqueda:
- **Igualdad exacta**: `WHERE nombre = 'Juan'`
- **Rango**: `WHERE precio > 100`
- **Ordenamiento**: `ORDER BY nombre` (el índice ya está ordenado)
- **Búsqueda por prefijo**: `LIKE 'texto%'` (busca palabras que empiecen con "texto")
- **BETWEEN**: `WHERE fecha BETWEEN '2024-01-01' AND '2024-01-31'`

Lo que NO soporta bien: `LIKE '%texto'` (búsqueda por sufijo) porque el índice está ordenado por el principio del texto, no por el final. Para eso necesitas Full-Text.

```sql
-- Índice por defecto en MySQL/PostgreSQL
CREATE INDEX idx_nombre ON clientes(nombre);

-- Ideal para:
-- WHERE columna = valor
-- WHERE columna > valor
-- ORDER BY columna
-- BETWEEN, LIKE 'texto%' (no LIKE '%texto')
```

### Hash

El índice Hash es óptimo para búsquedas de igualdad EXACTA. Usa una función hash para convertir el valor de la columna en una dirección de memoria. Es como un diccionario: buscas una palabra y obtienes su definición al instante.

PERO: no soporta búsquedas por rango, ORDER BY, ni LIKE. Solo igualdad (`=`). Por eso es menos usado que B-Tree. En MySQL solo está disponible para tablas MEMORY. En PostgreSQL se crea explícitamente con `USING HASH`.

```sql
-- Ideal para igualdad exacta (=)
-- MySQL: Solo en MEMORY
-- PostgreSQL: CREATE INDEX USING HASH
CREATE INDEX idx_hash_email ON clientes USING HASH (email);
```

### Bitmap (Oracle, PostgreSQL)

El índice Bitmap es ideal para columnas con POCOS VALORES DISTINTOS (cardinalidad baja). Por ejemplo: género (M/F), estado (activo/inactivo), sí/no.

En lugar de almacenar cada valor individual, usa un mapa de bits: cada valor distinto tiene un bitmap donde cada bit representa una fila. Esto hace que las operaciones AND/OR entre múltiples condiciones sean extremadamente rápidas.

No es adecuado para columnas con alta cardinalidad (como IDs o nombres) porque los bitmaps serían enormes. Tampoco es bueno para tablas con muchas escrituras concurrentes porque los bitmaps se bloquean.

```sql
-- Ideal para columnas con pocos valores distintos (cardinalidad baja)
-- Ej: estado('activo','inactivo'), genero('M','F')
-- PostgreSQL
CREATE BITMAP INDEX idx_bitmap_estado ON clientes(activo);
```

### Full-Text (Texto Completo)

Los índices B-Tree no sirven para buscar palabras dentro de un texto largo (no puedes hacer `LIKE '%laptop%'` de manera eficiente). Para eso existe Full-Text.

El índice Full-Text analiza el texto, extrae palabras clave (eliminando artículos, preposiciones, etc.), y construye un diccionario invertido. Luego puedes buscar con `MATCH...AGAINST` de manera muy eficiente.

Soporta:
- Búsqueda booleana: `+laptop -hp` (debe incluir "laptop", no debe incluir "hp")
- Búsqueda por frase: `"laptop hp probook"`
- Ranking por relevancia: los resultados se ordenan automáticamente por qué tan relevantes son

```sql
-- Búsqueda textual avanzada
CREATE FULLTEXT INDEX idx_busqueda ON productos(nombre_comercial, descripcion);

-- Uso
SELECT * FROM productos 
WHERE MATCH(nombre_comercial, descripcion) AGAINST('laptop hp probook' IN BOOLEAN MODE);
```

### GiST / SP-GiST (PostgreSQL)

GiST (Generalized Search Tree) es un índice para tipos de datos no estándar: geometría (puntos, polígonos), búsquedas de texto completo (alternativa a Full-Text), y datos geoespaciales.

El ejemplo muestra un índice sobre coordenadas geográficas. Esto permitiría consultas como "encuentra todas las sucursales en un radio de 10 km de esta ubicación" usando operadores como `<->` (distancia).

```sql
-- Búsqueda geoespacial y texto completo
CREATE INDEX idx_ubicacion ON sucursales USING GIST (coordenadas);
```

## Índices Compuestos

Un índice compuesto incluye VARIAS columnas. Es como tener un índice de libro que ordena primero por apellido, luego por nombre, luego por inicial del segundo apellido. El ORDEN de las columnas es CRUCIAL.

El ejemplo crea un índice en `(fecha_emision, estado)`. Esto es óptimo para consultas que filtran por fecha Y estado. Pero NO sirve para consultas que solo filtran por estado (sin fecha), porque la primera columna del índice es fecha_emision.

Piénsalo así: el índice ordena primero por fecha_emision, y dentro de cada fecha, ordena por estado. Si buscas solo por estado, el índice no puede ayudarte porque los estados están desordenados a través de las fechas.

```sql
-- Índice compuesto
CREATE INDEX idx_fecha_estado ON facturas(fecha_emision, estado);

-- Útil para:
SELECT * FROM facturas 
WHERE fecha_emision >= '2024-01-01' AND estado = 'activa';

-- NO es útil para (no usa la primera columna del índice):
SELECT * FROM facturas WHERE estado = 'activa';
```

### Regla del Prefijo Izquierdo

Esta regla es fundamental para entender los índices compuestos. Para un índice en columnas `(A, B, C)`:

- `WHERE A = ?` → USA el índice (usa A)
- `WHERE A = ? AND B = ?` → USA el índice (usa A y B)
- `WHERE A = ? AND B = ? AND C = ?` → USA el índice (usa A, B, C)
- `WHERE B = ?` → NO usa el índice (falta A, la primera columna)
- `WHERE A = ? AND C = ?` → USA el índice PARCIALMENTE (usa A pero no C, porque B está entre A y C)

La regla es: el índice se usa desde la IZQUIERDA. Necesitas la primera columna para acceder a la segunda, la segunda para acceder a la tercera, etc. Es como una escalera: no puedes saltarte escalones.

En el ejemplo del índice compuesto para reporting, la primera columna es `fecha_emision` porque las consultas de reporting siempre filtran por fecha (un periodo específico). Luego `id_cliente` (para ventas por cliente), luego `estado`, y finalmente `total` (para cubrir ORDER BY o agregaciones).

```sql
-- Buen diseño de índice compuesto para reporting de ventas
CREATE INDEX idx_reportes_ventas ON facturas(
    fecha_emision,
    id_cliente,
    estado,
    total
);

-- Consultas que se benefician:
-- 1. Ventas por fecha
SELECT * FROM facturas WHERE fecha_emision BETWEEN '2024-01-01' AND '2024-01-31';

-- 2. Ventas por cliente en un periodo
SELECT * FROM facturas 
WHERE fecha_emision >= '2024-01-01' AND id_cliente = 100;

-- 3. Ventas activas por periodo
SELECT * FROM facturas 
WHERE fecha_emision >= '2024-01-01' AND estado = 'activa';
```

## ANALYZE y Estadísticas

El optimizador de consultas es el "cerebro" de la base de datos. Decide si usar un índice o hacer un full scan basándose en ESTADÍSTICAS: cuántas filas tiene la tabla, cuántos valores distintos hay en cada columna, cómo se distribuyen los datos.

ANALYZE TABLE (MySQL) o ANALYZE (PostgreSQL) actualiza estas estadísticas. Si no las actualizas, el optimizador puede tomar malas decisiones. Por ejemplo, puede decidir hacer un full scan porque las estadísticas dicen que la tabla tiene 100 filas, cuando en realidad ya tiene 1,000,000.

¿Cuándo ejecutar ANALYZE?
- Después de cargar muchos datos (importaciones masivas)
- Periódicamente (en mantenimiento nocturno)
- Cuando notas que consultas que antes eran rápidas ahora son lentas (puede indicar estadísticas desactualizadas)

SHOW INDEX FROM (MySQL) y pg_stats (PostgreSQL) muestran las estadísticas actuales incluyendo cardinalidad, selectividad, etc.

```sql
-- MySQL
ANALYZE TABLE facturas;

-- PostgreSQL
ANALYZE facturas;

-- Ver estadísticas de una tabla
-- MySQL
SHOW INDEX FROM facturas;

-- PostgreSQL
SELECT * FROM pg_stats WHERE tablename = 'facturas';
```

## EXPLAIN - Plan de Ejecución

EXPLAIN es LA HERRAMIENTA MÁS IMPORTANTE para optimizar consultas. Te muestra CÓMO ejecutará MySQL tu consulta: qué índices usará, en qué orden unirá las tablas, cuántas filas estima examinar.

Para usarlo, solo pon EXPLAIN antes de tu SELECT (o EXPLAIN ANALYZE en PostgreSQL que además ejecuta la consulta y muestra tiempos reales).

El ejemplo muestra un JOIN entre facturas y clientes. EXPLAIN te dirá:
- Si usa índices o hace full scan en cada tabla
- El tipo de acceso (const, eq_ref, ref, range, index, ALL)
- Cuántas filas estima examinar
- Si usa "Using where", "Using index", "Using temporary", etc.

```sql
EXPLAIN SELECT f.folio, c.nombre, f.total
FROM facturas f
JOIN clientes c ON f.id_cliente = c.id_cliente
WHERE f.fecha_emision >= '2024-01-01'
  AND c.activo = TRUE
ORDER BY f.total DESC;
```

### Columnas de EXPLAIN

Las columnas más importantes de EXPLAIN son:

- **type**: el tipo de acceso. Es la columna MÁS IMPORTANTE para identificar problemas. De peor a mejor:
  - `ALL` = Full table scan (peor) — la base de datos lee toda la tabla
  - `index` = Full index scan — lee todo el índice (mejor que ALL pero todavía malo)
  - `range` = Búsqueda por rango — usa el índice para filtrar un rango
  - `ref` = Búsqueda por valor no único — encuentra múltiples filas por un valor
  - `eq_ref` = Búsqueda por clave única en JOIN — para cada fila de la tabla anterior, encuentra UNA fila
  - `const` = Búsqueda por PRIMARY KEY — la mejor, solo una fila

- **key**: qué índice usó (o NULL si no usó ninguno).
- **rows**: estimación de cuántas filas examinará. Entre más alto, peor.
- **Extra**: información adicional como "Using index" (cubriente), "Using where", "Using temporary" (usa tabla temporal, malo), "Using filesort" (ordena sin índice, malo).

| Columna | Significado |
|---------|-------------|
| type | ALL (full scan), range, ref, eq_ref, const |
| key | Índice usado |
| rows | Filas estimadas a examinar |
| Extra | Usando índice, usando temporal, etc. |

### Tipos de Acceso (de peor a mejor)

- `ALL` = Full table scan: la base de datos LEE TODA LA TABLA. Es lo peor que puede pasar. Generalmente significa que falta un índice.
- `index` = Full index scan: lee todo el índice. Mejor que ALL (el índice suele ser más pequeño que la tabla), pero sigue siendo costoso.
- `range` = Búsqueda por rango: usa el índice para encontrar un rango de valores (BETWEEN, >, <, LIKE prefijo). Bueno.
- `ref` = Búsqueda por valor no único: usa el índice para buscar un valor que puede aparecer varias veces (como estado = 'activa' con índice). Bueno.
- `eq_ref` = Búsqueda por única en JOIN: para cada fila de la primera tabla, encuentra EXACTAMENTE UNA fila en la segunda tabla usando su clave primaria. Muy bueno.
- `const` = Búsqueda por PRIMARY KEY con valor constante: la mejor. Solo se lee una fila.

```
ALL         → Full table scan (peor)
index       → Full index scan
range       → Búsqueda por rango
ref         → Búsqueda por no único
eq_ref      → Búsqueda por único (JOIN)
const       → Búsqueda por PK (mejor)
```

## Optimización de Consultas

### 1. Evitar SELECT *

`SELECT *` trae TODAS las columnas de la tabla, incluso las que no necesitas. Esto tiene varios problemas:
- Más datos transferidos entre la base de datos y la aplicación (más ancho de banda, más lento).
- La base de datos no puede usar "índices cubrientes" (covering index): si todas las columnas que necesitas están en el índice, la consulta puede resolverse solo con el índice, sin tocar la tabla.
- Más memoria en el servidor de base de datos.
- Más difícil de leer y mantener (no sabes qué columnas espera la aplicación).

Siempre especifica SOLO las columnas que necesitas: `SELECT nombre, email FROM clientes`. Es más eficiente, más claro, y menos propenso a errores si la estructura de la tabla cambia.

```sql
-- MAL
SELECT * FROM clientes WHERE id_cliente = 100;

-- BIEN
SELECT nombre, email FROM clientes WHERE id_cliente = 100;
```

### 2. Usar EXISTS en lugar de IN

Como viste en el capítulo de subconsultas, EXISTS usa cortocircuito: en cuanto encuentra una coincidencia, se detiene. IN, en cambio, recolecta TODOS los resultados de la subconsulta primero y luego los compara.

La diferencia es dramática cuando la subconsulta devuelve muchos valores. EXISTS puede ser órdenes de magnitud más rápido. Además, EXISTS maneja NULLs correctamente.

```sql
-- MAL (lento si subconsulta es grande)
SELECT * FROM clientes 
WHERE id_cliente IN (SELECT id_cliente FROM facturas);

-- BIEN (usa cortocircuito)
SELECT * FROM clientes c
WHERE EXISTS (SELECT 1 FROM facturas f WHERE f.id_cliente = c.id_cliente);
```

### 3. Filtrar lo antes posible

En un JOIN, entre más pronto filtres datos, menos filas tendrá que procesar el JOIN. Hay dos formas de filtrar: en el WHERE o en el ON del JOIN.

Ambas consultas producen el mismo resultado, pero la segunda es más eficiente porque el filtro `f.fecha_emision >= '2024-01-01'` se aplica ANTES del JOIN, reduciendo las filas de facturas que entran al JOIN. En la primera, primero se hace el JOIN completo y luego se filtra.

Esto es especialmente importante cuando una tabla es mucho más grande que la otra. Filtra la tabla grande primero.

```sql
-- MAL (filtra después de JOIN)
SELECT c.nombre, SUM(f.total)
FROM clientes c
JOIN facturas f ON c.id_cliente = f.id_cliente
WHERE f.fecha_emision >= '2024-01-01'
GROUP BY c.id_cliente;

-- BIEN (filtra antes)
SELECT c.nombre, SUM(f.total)
FROM clientes c
JOIN facturas f ON c.id_cliente = f.id_cliente 
    AND f.fecha_emision >= '2024-01-01'
GROUP BY c.id_cliente;
```

### 4. Evitar funciones en columnas indexadas

Cuando envuelves una columna indexada dentro de una función, el índice se vuelve INÚTIL. La base de datos no puede buscar en el índice porque el valor indexado está transformado.

`YEAR(fecha_emision) = 2024` NO puede usar el índice de `fecha_emision`. La base de datos tiene que calcular YEAR() para cada fila. En cambio, `fecha_emision >= '2024-01-01' AND fecha_emision < '2025-01-01'` SÍ puede usar el índice porque está comparando directamente el valor de la columna.

Esta optimización es extremadamente común y fácil de aplicar. Siempre que veas una función alrededor de una columna en el WHERE, pregúntate si puedes reescribirla como una comparación de rango.

```sql
-- MAL (no usa índice aunque exista en fecha_emision)
SELECT * FROM facturas WHERE YEAR(fecha_emision) = 2024;

-- BIEN (usa índice)
SELECT * FROM facturas WHERE fecha_emision >= '2024-01-01' AND fecha_emision < '2025-01-01';
```

### 5. Usar UNION ALL en vez de UNION

UNION elimina duplicados entre los dos conjuntos de resultados para dar un resultado único. Para eliminar duplicados, la base de datos tiene que ordenar o hashear los resultados, lo que cuesta tiempo y memoria.

UNION ALL simplemente concatena los resultados. Si sabes que no hay duplicados (o no te importan), UNION ALL es mucho más rápido.

La regla: pregúntate "¿necesito eliminar duplicados?" Si la respuesta es no, usa UNION ALL.

```sql
-- UNION elimina duplicados (costo extra)
SELECT nombre, email FROM clientes_activos
UNION
SELECT nombre, email FROM clientes_inactivos;

-- UNION ALL es más rápido (no elimina duplicados)
SELECT nombre, email FROM clientes_activos
UNION ALL
SELECT nombre, email FROM clientes_inactivos;
```

### 6. Limitar resultados temprano

Si solo necesitas los 100 mejores clientes, no calcules datos para todos los clientes y luego tomes los 100 primeros. Aplica LIMIT lo antes posible.

La subconsulta interna encuentra los 100 clientes con más compras. Luego, la consulta externa filtra solo aquellos con compras > $10,000. El LIMIT dentro de la subconsulta hace que la ordenación sea mucho más barata (solo ordena hasta encontrar los 100 mejores, no toda la tabla).

```sql
-- Usar subconsulta con LIMIT
SELECT * FROM (
    SELECT id_cliente, nombre, total_compras
    FROM clientes
    ORDER BY total_compras DESC
    LIMIT 100
) AS top_clientes
WHERE top_clientes.total_compras > 10000;
```

## Estrategias de Indexación

### Para sistemas OLTP (muchas escrituras)

OLTP (Online Transaction Processing) son sistemas donde hay muchas operaciones de escritura: INSERT, UPDATE, DELETE. Ejemplos: sistemas de punto de venta, registro de pedidos, sistemas bancarios.

En estos sistemas, CADA ÍNDICE ADICIONAL RALENTIZA LAS ESCRITURAS. Por cada INSERT, la base de datos debe actualizar TODOS los índices de la tabla. Si tienes 5 índices, un INSERT es 5 veces más lento que sin índices.

La estrategia es: índices pequeños en las columnas críticas. Solo indexa lo que realmente necesitas para JOINs y WHEREs frecuentes. No sobresatures.

```sql
-- Menos índices, más pequeños
-- Índices en columnas de JOIN y WHERE frecuente
CREATE INDEX idx_facturas_cliente ON facturas(id_cliente);
CREATE INDEX idx_facturas_fecha ON facturas(fecha_emision);
CREATE INDEX idx_facturas_estado ON facturas(estado);
```

### Para sistemas OLAP/Reporting (muchas lecturas)

OLAP (Online Analytical Processing) son sistemas de análisis y reportes donde hay muchas lecturas y pocas escrituras. Ejemplos: data warehouses, tableros de control, reportes mensuales.

Aquí los índices son más agresivos. Usa índices compuestos, índices cubrientes (covering index: el índice contiene TODAS las columnas que necesita la consulta, por lo que la base de datos nunca toca la tabla), e índices parciales (PostgreSQL).

Un índice cubriente con INCLUDE (PostgreSQL) guarda columnas adicionales en el índice sin ordenarlas. Son como "adjuntos" al índice: ayudan a cubrir la consulta sin añadir costo de ordenamiento.

```sql
-- Índices más agresivos, compuestos
CREATE INDEX idx_reportes ON facturas(fecha_emision, id_cliente, estado, total);
-- Índices cubrientes (covering index)
CREATE INDEX idx_cubriente ON facturas(fecha_emision, total) 
    INCLUDE (folio, id_cliente);  -- PostgreSQL
```

### Para búsquedas textuales

Para buscar palabras dentro de texto largo, necesitas FULLTEXT. No hay alternativa eficiente con B-Tree. Si tu aplicación tiene un campo de búsqueda que busca en nombres y descripciones de productos, este índice es indispensable.

```sql
CREATE FULLTEXT INDEX idx_fulltext ON productos(nombre_comercial, descripcion);
```

## Particionamiento

El particionamiento divide una tabla GRANDE en partes más pequeñas (particiones) basándose en una regla. Cada partición se almacena y se consulta de forma independiente. Es como tener varios archivos más pequeños en lugar de uno gigante.

### Particionamiento por rango

Divide los datos por rangos de valores. En el ejemplo, las facturas se dividen por año: todas las de 2022 van a la partición p2022, las de 2023 a p2023, etc. Cuando consultas facturas de 2024, la base de datos SOLO busca en la partición p2024, ignorando las demás. Esto se llama "pruning de particiones".

Ideal para datos históricos donde siempre consultas por fecha.

### Particionamiento por hash

Distribuye los datos EQUITATIVAMENTE entre N particiones usando una función hash. No importa el valor de la columna — el hash determina la partición. Útil cuando no hay un rango natural para particionar pero quieres distribuir la carga.

Las particiones son transparentes para las consultas: escribes SELECT normalmente y la base de datos decide qué particiones usar.

```sql
-- Particionamiento por rango (MySQL)
CREATE TABLE facturas (
    id_factura INT,
    folio VARCHAR(30),
    fecha_emision DATE,
    total DECIMAL(12,2)
) PARTITION BY RANGE (YEAR(fecha_emision)) (
    PARTITION p2022 VALUES LESS THAN (2023),
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION p2024 VALUES LESS THAN (2025),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);

-- Particionamiento por hash (distribución equitativa)
CREATE TABLE facturas_detalle (
    id_detalle INT,
    id_factura INT,
    id_producto INT,
    cantidad INT
) PARTITION BY HASH(id_factura) PARTITIONS 8;
```

## Mantenimiento de Índices

Los índices no son "configurar y olvidar". Necesitan mantenimiento:

**Fragmentación**: con el tiempo, los índices se fragmentan (especialmente con muchas operaciones de INSERT/UPDATE/DELETE). Un índice fragmentado ocupa más espacio y es más lento.

**Reconstrucción**: en MySQL, `ALTER TABLE ... ENGINE=InnoDB` reconstruye la tabla y sus índices. En PostgreSQL, `REINDEX` hace lo mismo.

**Monitoreo de tamaño**: los índices pueden ocupar MÁS espacio que los datos. La consulta de información_schema te muestra cuánto espacio ocupan los datos vs los índices de cada tabla. Si los índices son desproporcionadamente grandes, puede indicar fragmentación o índices redundantes.

**Estadísticas actualizadas**: ANALYZE TABLE mantiene las estadísticas frescas para que el optimizador tome buenas decisiones.

```sql
-- Ver tamaño de índices (MySQL)
SELECT 
    TABLE_NAME,
    ROUND(DATA_LENGTH/1024/1024, 2) AS data_mb,
    ROUND(INDEX_LENGTH/1024/1024, 2) AS index_mb
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'sistema_ventas';

-- Reconstruir índices (MySQL)
ALTER TABLE facturas ENGINE=InnoDB;

-- Reindexar (PostgreSQL)
REINDEX TABLE facturas;

-- Actualizar estadísticas
ANALYZE TABLE facturas;
```

## 📊 Monitoreo de Performance

Para optimizar, primero necesitas SABER qué está lento. Aquí tienes herramientas para identificar problemas:

### Slow Query Log (MySQL)

El slow query log registra automáticamente las consultas que tardan más de N segundos (configurable con `long_query_time`). Es la primera herramienta para identificar consultas problemáticas.

### pg_stat_statements (PostgreSQL)

Esta vista del sistema muestra estadísticas de TODAS las consultas ejecutadas: cuántas veces se llamó cada una, tiempo promedio, filas devueltas. Ordenando por `total_time DESC` encuentras las consultas que más recursos consumen.

### SHOW PROCESSLIST (MySQL)

Muestra TODAS las conexiones activas y qué consulta está ejecutando cada una. Si ves una consulta que lleva horas ejecutándose, puedes matarla con `KILL CONNECTION`.

### Locks

Los bloqueos (locks) ocurren cuando dos transacciones intentan modificar los mismos datos al mismo tiempo. `INNODB_LOCKS` y `INNODB_LOCK_WAITS` muestran qué transacciones están bloqueadas y cuáles las están bloqueando.

```sql
-- Consultas lentas (MySQL)
-- Habilitar slow query log en my.cnf:
-- slow_query_log = 1
-- long_query_time = 2

-- Identificar consultas lentas (PostgreSQL)
SELECT 
    query,
    calls,
    total_time / calls AS avg_time_ms,
    rows / calls AS avg_rows
FROM pg_stat_statements
ORDER BY total_time DESC
LIMIT 10;

-- Ver qué está bloqueando (MySQL)
SHOW PROCESSLIST;
SHOW FULL PROCESSLIST;

-- Terminar consulta problemática
KILL CONNECTION 12345;

-- Ver locks (MySQL)
SELECT * FROM information_schema.INNODB_LOCKS;
SELECT * FROM information_schema.INNODB_LOCK_WAITS;
```

## Errores Comunes

### 1. Demasiados índices

El error más común de los principiantes es: "si un índice es bueno, más índices son mejores". FALSO. Cada índice adicional hace que las escrituras sean más lentas.

En el ejemplo, una tabla con 1 millón de facturas: con 1 índice un INSERT tarda 5ms; con 5 índices tarda 25ms. En un sistema con muchas escrituras, esto se acumula.

**Regla**: indexa solo lo necesario. No crees índices "por si acaso". Cada índice debe estar justificado por una consulta específica.

```sql
-- Cada índice ralentiza INSERT/UPDATE/DELETE
-- Para una tabla de facturas con 1M registros:
-- 1 índice: INSERT toma 5ms
-- 5 índices: INSERT toma 25ms
```

### 2. No indexar columnas de JOIN

Las columnas usadas en JOINs son las PRIMERAS candidatas a indexar. Si haces JOIN de clientes con facturas por `id_cliente`, DEBE haber un índice en `facturas.id_cliente`. Sin él, la base de datos hará un full scan de facturas para cada cliente.

Revisa siempre los JOINs en tus consultas. Si una columna de JOIN no tiene índice, es casi seguro que es un problema de rendimiento.

```sql
-- Lento: no hay índice en facturas.id_cliente
SELECT c.nombre, f.total
FROM clientes c
JOIN facturas f ON c.id_cliente = f.id_cliente;
```

### 3. Orden incorrecto en índices compuestos

En un índice compuesto, la PRIMERA columna debe ser la de MAYOR SELECTIVIDAD (la que más reduce el resultado). Poner una columna de baja selectividad (como estado con solo 3 valores) primero hace que el índice sea menos útil.

En el ejemplo MAL: `(estado, fecha_emision)` — primero filtras por estado (reduce a ~33% de las filas), luego por fecha (reduce aún más). Pero la base de datos no puede usar el índice eficientemente para rangos de fecha porque la fecha está en segundo lugar.

En el ejemplo BIEN: `(fecha_emision, estado)` — primero filtras por fecha (reduce drásticamente), luego por estado. La base de datos puede hacer una búsqueda por rango en fecha y luego filtrar por estado.

```sql
-- Poner columnas de alta selectividad primero
-- MAL: (estado, fecha) - estado tiene pocos valores
CREATE INDEX idx_mal ON facturas(estado, fecha_emision);

-- BIEN: (fecha, estado) - fecha tiene más valores distintos
CREATE INDEX idx_bien ON facturas(fecha_emision, estado);
```

## Checklist de Optimización

Esta lista resume TODO lo que debes verificar antes de decir que una consulta está optimizada:

- [ ] ¿Las columnas en WHERE tienen índices?
- [ ] ¿Los JOIN usan columnas indexadas?
- [ ] ¿Los ORDER BY están cubiertos por índices?
- [ ] ¿Evitas SELECT * en producción?
- [ ] ¿Las funciones en WHERE no bloquean índices?
- [ ] ¿Los tipos de datos son correctos?
- [ ] ¿Hay índices duplicados o redundantes?
- [ ] ¿Las estadísticas están actualizadas?
- [ ] ¿Los índices compuestos tienen el orden correcto?
- [ ] ¿Hay tablas que necesiten particionamiento?
- [ ] ¿Las consultas más frecuentes están optimizadas?

---

## Anterior: [07 Create Table Y Ddl](../02-fundamentos/07-create-table-y-ddl.md)
## Siguiente: [01 Modelo Datos Ventas](../03-ventas/01-modelo-datos-ventas.md)

[Volver al índice](../README.md)
