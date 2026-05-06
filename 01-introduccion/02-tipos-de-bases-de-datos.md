# 1.2 Tipos de Bases de Datos

Imagina que estás organizando un negocio de ventas. Necesitas guardar información de tus clientes, tus productos, las facturas que emites y el inventario que tienes en tu almacén. Dependiendo de cómo quieras organizar esa información —como si fuera un archivador, un cuaderno, una pizarra o un mapa—, elegirás un tipo de base de datos diferente. En esta sección vamos a recorrer los tipos más importantes para que entiendas cuál te conviene según lo que necesites hacer.

---

## Clasificación General

Antes de meternos de lleno, piensa en las bases de datos como **herramientas de organización**. Al igual que no usarías un martillo para atornillar ni un destornillador para clavar, cada tipo de base de datos tiene un propósito distinto. Algunas son excelentes para llevar el control de ventas y facturación (como un libro de contabilidad), otras son mejores para catálogos de productos que cambian constantemente (como un pizarrón donde puedes borrar y reescribir), y otras están diseñadas para analizar millones de datos en segundos (como una calculadora gigante).

Vamos a ver las dos grandes familias: las **relacionales** (RDBMS) y las **NoSQL**.

---

### 1. Bases de Datos Relacionales (RDBMS)

Las bases de datos relacionales son las más utilizadas en el mundo de los negocios, especialmente para sistemas de ventas, compras, inventarios y facturación. ¿Por qué? Porque organizan la información en **tablas** que se conectan entre sí.

**Piensa en un archivero de la vieja escuela:** tienes un cajón para clientes, otro para productos, otro para facturas. Cada cajón contiene fichas ordenadas. Pero además, las fichas de facturas tienen anotaciones como "este cliente es el número 42", que hace referencia a la ficha del cliente en el cajón de clientes. Eso es exactamente lo que hace una base de datos relacional: guarda datos en tablas separadas pero las relaciona mediante **claves** (como esos números de referencia).

**Ventajas:**
- **Integridad referencial:** La base de datos te garantiza que no puedes tener una factura de un cliente que no existe. Es como si el archivero tuviera un candado que impide poner una ficha de factura que referencia a un cliente borrado.
- **ACID (Atomicidad, Consistencia, Aislamiento, Durabilidad):** Son cuatro propiedades que suenan complicadas pero que en el fondo significan que cuando haces una operación (como cobrar una venta y descontar del inventario), **o se hace todo completo o no se hace nada**. No hay términos medios. Esto es crucial para el dinero: no querrías que se cobre un cliente pero no se descuente del stock, o viceversa.
- **Lenguaje estandarizado (SQL):** Aunque cada motor (MySQL, PostgreSQL, etc.) tiene sus pequeños dialectos, el lenguaje SQL es prácticamente el mismo en todos. Lo que aprendas con uno te servirá para los demás.
- **Madurez y soporte:** Estas tecnologías tienen décadas de existencia. Hay millones de tutoriales, libros, foros y profesionales que las conocen.

**Desventajas:**
- **Escalamiento horizontal complejo:** Si tu negocio crece muchísimo (como pasar de 100 a 100,000 clientes), agregar más servidores no es tan sencillo como en otros tipos de bases de datos.
- **Esquema rígido:** Antes de guardar datos, tienes que definir exactamente la estructura (qué columnas tendrá cada tabla, de qué tipo serán, etc.). Si después necesitas cambiar algo, puede ser complicado. Es como si diseñaras un formulario impreso y luego quisieras agregar un campo nuevo —tendrías que rediseñarlo y reimprimirlo.
- **Performance con datos no estructurados:** Si quieres guardar cosas como fotos de productos, descripciones largas o documentos PDF, las bases relacionales no son las más eficientes.

Ahora vamos a ver los motores de bases de datos relacionales más populares, uno por uno, con ejemplos concretos de código SQL.

---

#### MySQL / MariaDB

MySQL es el motor de base de datos más popular del mundo, especialmente en aplicaciones web. Es como el "auto sedan" de las bases de datos: confiable, económico (gratuito), fácil de manejar y con una gran comunidad de mecánicos (desarrolladores) que pueden ayudarte. MaríaDB es una versión derivada (un "clon mejorado") creada por los mismos desarrolladores originales de MySQL.

**¿Para qué se usa típicamente?** Sistemas de punto de venta (POS), tiendas en línea (e-commerce), sistemas ERP para pequeñas y medianas empresas, y prácticamente cualquier aplicación web.

El siguiente bloque de código crea una tabla llamada `ventas`. Pero antes de verlo, vamos a entender qué significa cada parte, porque este es probablemente el comando SQL más importante que aprenderás.

`CREATE TABLE` es la instrucción que le dice al motor de base de datos: "Quiero crear una nueva estructura para guardar información". Dentro de los paréntesis, defines cada columna de tu tabla. Es como diseñar las columnas de una hoja de Excel:

- **`id INT AUTO_INCREMENT PRIMARY KEY`**: Esta es la columna que identifica de manera única cada venta. `INT` significa que guardará números enteros (1, 2, 3...). `AUTO_INCREMENT` le dice a MySQL "cada vez que inserts una nueva venta, asigna automáticamente el número siguiente" (como un folio automático). `PRIMARY KEY` indica que esta columna es la llave principal: no pueden existir dos ventas con el mismo ID, y MySQL usará esta columna para buscar registros rápidamente.
- **`total DECIMAL(12,2)`**: Aquí guardaremos el monto total de la venta. `DECIMAL(12,2)` significa un número decimal con 12 dígitos en total, de los cuales 2 son después del punto decimal. Es decir, puedes guardar desde 0.01 hasta 999,999,999.99. Se usa `DECIMAL` en lugar de `FLOAT` porque los números decimales en cálculos financieros deben ser exactos (sin errores de redondeo).
- **`fecha DATETIME`**: Guarda la fecha y hora exacta en que ocurrió la venta. Es como ponerle un sello de tiempo a cada transacción.
- **`INDEX idx_fecha (fecha)`**: Un **índice** es como el índice alfabético al final de un libro. En lugar de hojear página por página para encontrar todas las ventas de un día, el índice le dice a la base de datos exactamente dónde están. Esto acelera enormemente las búsquedas por fecha.
- **`ENGINE=InnoDB`**: Especifica el "motor de almacenamiento". InnoDB es el motor recomendado porque soporta transacciones (ACID), claves foráneas (relaciones entre tablas) y es confiable para sistemas de producción.

```sql
-- Ideal para: ERP, e-commerce, POS
CREATE TABLE ventas (
    id INT AUTO_INCREMENT PRIMARY KEY,
    total DECIMAL(12,2),
    fecha DATETIME,
    INDEX idx_fecha (fecha)
) ENGINE=InnoDB;
```

---

#### PostgreSQL

Si MySQL es el sedán confiable, PostgreSQL es la camioneta todoterreno con herramientas especializadas. Es un motor de base de datos de código abierto que ofrece funcionalidades avanzadas que otros no tienen: soporte nativo para datos geográficos (GIS), almacenamiento de datos en formato JSON (documentos flexibles), vistas materializadas (resultados de consultas guardados como si fueran tablas), y mucho más.

**¿Para qué se usa?** Analítica de datos, sistemas de información geográfica (mapas, ubicaciones), aplicaciones que necesitan combinar datos estructurados con semiestructurados, y como reemplazo directo de Oracle o SQL Server en empresas que quieren ahorrar costos de licencias.

Observa este primer bloque de código. Hay varias diferencias interesantes respecto a MySQL:

- **`id SERIAL PRIMARY KEY`**: `SERIAL` es un atajo en PostgreSQL equivalente a `INT AUTO_INCREMENT` en MySQL. Crea una columna que se auto-incrementa sola. PostgreSQL lo implementa de una manera particular: crea una "secuencia" (un contador) detrás de escenas.
- **`NUMERIC(12,2)`**: Es el equivalente de `DECIMAL` en MySQL. PostgreSQL es muy estricto con los tipos de datos, y `NUMERIC` garantiza precisión exacta para dinero.
- **`TIMESTAMPTZ DEFAULT NOW()`**: `TIMESTAMPTZ` significa "timestamp con zona horaria" (timezone). Guarda no solo la fecha y hora, sino también la zona horaria en la que se registró. Es útil cuando tu negocio opera en diferentes regiones. `DEFAULT NOW()` asigna automáticamente la fecha y hora actual si no se proporciona un valor.
- **`JSONB`**: Este tipo de dato permite guardar documentos JSON en formato binario optimizado. ¿Para qué sirve? Imagina que quieres guardar datos adicionales de cada venta que no siempre tienen la misma estructura: a veces un descuento especial, a veces notas de envío, a veces el método de pago. Con `JSONB` puedes guardar esa información flexible dentro de una tabla relacional.
- **`GEOGRAPHY(POINT)`**: Este tipo de dato guarda coordenadas geográficas (latitud y longitud). PostgreSQL, con su extensión PostGIS, puede hacer cálculos como "dime qué clientes están a menos de 5 kilómetros de esta sucursal".

```sql
-- Ideal para: Analítica, GIS, datos complejos
CREATE TABLE ventas (
    id SERIAL PRIMARY KEY,
    total NUMERIC(12,2),
    fecha TIMESTAMPTZ DEFAULT NOW(),
    datos_adicionales JSONB,
    ubicacion GEOGRAPHY(POINT)
);
```

El siguiente bloque muestra una **vista materializada**. ¿Qué es una vista materializada? Es como si tomaras una foto instantánea de un reporte. En lugar de calcular cada vez el resumen de ventas del día (lo cual puede ser lento si tienes millones de registros), le dices a PostgreSQL que calcule el resultado una vez y lo guarde físicamente en disco como si fuera una tabla. Luego, cuando necesites ese reporte, lo lees de la "foto" en lugar de recalcularlo.

`CREATE MATERIALIZED VIEW` crea esa estructura. La consulta interna `SELECT DATE(fecha) AS dia, COUNT(*) AS total_transacciones, SUM(total) AS ingresos FROM ventas GROUP BY DATE(fecha)` hace lo siguiente:
- `DATE(fecha)`: Extrae solo la parte de la fecha (sin la hora) de la columna `fecha`.
- `COUNT(*)`: Cuenta cuántas ventas hay. El asterisco significa "todas las filas".
- `SUM(total)`: Suma todos los totales de venta.
- `GROUP BY DATE(fecha)`: Agrupa los resultados por día. Es como si dijeras "para cada día, dame el total de transacciones y la suma de ingresos".

El resultado es una tabla que podrías consultar tan rápido como `SELECT * FROM resumen_ventas_diarias` sin tener que recorrer millones de registros cada vez.

```sql
-- Vistas materializadas para reporting
CREATE MATERIALIZED VIEW resumen_ventas_diarias AS
SELECT 
    DATE(fecha) AS dia,
    COUNT(*) AS total_transacciones,
    SUM(total) AS ingresos
FROM ventas
GROUP BY DATE(fecha);
```

---

#### Microsoft SQL Server

SQL Server es el motor de bases de datos de Microsoft. Piensa en él como un **automóvil de lujo con asistentes de conducción avanzados**: viene cargado de funcionalidades empresariales, integración profunda con herramientas de Microsoft (Excel, Power BI, SharePoint) y soporte técnico de primera clase. No es gratuito (excepto la edición Developer para desarrollo), pero muchas empresas lo eligen por su confiabilidad y su ecosistema.

**¿Para qué se usa?** Empresas que ya usan tecnología Microsoft (.NET, Azure, Office 365), business intelligence (BI) corporativo, reporting con SQL Server Reporting Services (SSRS), y aplicaciones que requieren alta disponibilidad.

Fíjate en las particularidades de este código:

- **`id INT IDENTITY(1,1) PRIMARY KEY`**: `IDENTITY(1,1)` es el equivalente de `AUTO_INCREMENT` en SQL Server. El primer número (1) es el valor inicial, el segundo (1) es el incremento. Así, la primera venta tendrá id=1, la segunda id=2, etc.
- **`DATETIME2`**: Es una versión más precisa de `DATETIME`. Mientras que `DATETIME` tradicional tiene precisión de 3.33 milisegundos, `DATETIME2` llega a 100 nanosegundos. Además, tiene mayor rango de fechas.
- **`GENERATED ALWAYS AS ROW START` y `GENERATED ALWAYS AS ROW END`**: Aquí empieza lo interesante. Estas columnas son parte de una característica llamada **Temporal Tables** (tablas temporales). En lugar de solo guardar el estado actual de los datos, SQL Server guarda automáticamente todo el historial de cambios. Cuando modificas una venta, SQL Server no sobrescribe el registro anterior: lo mueve a una tabla de historial y deja el nuevo como actual.
- **`PERIOD FOR SYSTEM_TIME (...)`**: Define el período de vigencia de cada versión del registro.
- **`WITH (SYSTEM_VERSIONING = ON)`**: Activa el versionado automático. Como resultado, puedes hacer consultas como "muéstrame cómo estaba esta venta el mes pasado" sin haber programado nada adicional.

```sql
-- Ideal para: Empresas .NET, BI corporativo
CREATE TABLE ventas (
    id INT IDENTITY(1,1) PRIMARY KEY,
    total DECIMAL(12,2),
    fecha DATETIME2,
    -- Columnas para temporal tables
    periodo_inicio DATETIME2 GENERATED ALWAYS AS ROW START,
    periodo_fin DATETIME2 GENERATED ALWAYS AS ROW END,
    PERIOD FOR SYSTEM_TIME (periodo_inicio, periodo_fin)
) WITH (SYSTEM_VERSIONING = ON);
```

---

#### Oracle Database

Oracle es el **tanque blindado** de las bases de datos. Es el motor más robusto, maduro y caro del mercado. Lo utilizan grandes corporaciones, bancos, gobiernos y aerolíneas —organizaciones donde una falla en la base de datos puede costar millones de dólares por minuto. Ofrece características de alta disponibilidad, recuperación ante desastres, particionamiento avanzado y seguridad de nivel empresarial.

**¿Para qué se usa?** Grandes corporaciones, sistemas de misión crítica (transacciones bancarias, reservas aéreas), aplicaciones Java empresariales (Oracle es dueño de Java), y data warehouses masivos.

Analicemos el primer bloque:

- **`id NUMBER GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY`**: `NUMBER` es el tipo numérico genérico de Oracle (equivalenta a INT o DECIMAL). `GENERATED BY DEFAULT AS IDENTITY` es el equivalente de `AUTO_INCREMENT`: le dice a Oracle que genere automáticamente un valor para esta columna si no se proporciona uno. "By default" significa que aún puedes insertar tus propios valores si quieres.
- **`NUMBER(12,2)`**: Similar a `DECIMAL(12,2)` en otros motores: 12 dígitos totales, 2 decimales.
- **`TIMESTAMP WITH LOCAL TIME ZONE`**: Guarda la fecha y hora convirtiéndola a la zona horaria de la base de datos. Cuando recuperas el dato, Oracle lo convierte automáticamente a la zona horaria de tu sesión.

```sql
-- Ideal para: Grandes corporaciones, misión crítica
CREATE TABLE ventas (
    id NUMBER GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
    total NUMBER(12,2),
    fecha TIMESTAMP WITH LOCAL TIME ZONE
);
```

El segundo bloque muestra **particionamiento** (partitioning). Imagina que tienes un archivero con miles de facturas. Sería mucho más rápido buscar si las facturas están organizadas por mes en carpetas separadas, en lugar de tener todo revuelto en un solo cajón gigante. Eso es exactamente lo que hace el particionamiento: divide una tabla grande en segmentos más pequeños (particiones) basados en una regla, en este caso, el rango de fechas.

`PARTITION BY RANGE (fecha)` le dice a Oracle "divide esta tabla usando el valor de la columna fecha". Luego defines cada partición con `VALUES LESS THAN (DATE '2024-04-01')`, que significa "todo lo que sea anterior al 1 de abril de 2024 va a esta partición".

Cuando hagas una consulta como `SELECT * FROM ventas_part WHERE fecha BETWEEN '2024-05-01' AND '2024-06-01'`, Oracle sabrá automáticamente que solo necesita revisar la partición `ventas_2024_q2`, ignorando las demás. Esto acelera las consultas de manera drástica.

```sql
-- Particionamiento por rango de fechas
CREATE TABLE ventas_part (
    id NUMBER,
    total NUMBER(12,2),
    fecha DATE
) PARTITION BY RANGE (fecha) (
    PARTITION ventas_2024_q1 VALUES LESS THAN (DATE '2024-04-01'),
    PARTITION ventas_2024_q2 VALUES LESS THAN (DATE '2024-07-01'),
    PARTITION ventas_2024_q3 VALUES LESS THAN (DATE '2024-10-01'),
    PARTITION ventas_2024_q4 VALUES LESS THAN (DATE '2025-01-01')
);
```

---

### 2. Bases de Datos NoSQL

Hasta ahora hemos visto bases de datos relacionales, que son como **archiveros organizados con formularios estrictos**. Pero ¿qué pasa si tu información no encaja en un formulario fijo? ¿Qué pasa si cada producto de tu catálogo tiene características diferentes? ¿O si necesitas guardar millones de lecturas de sensores por segundo?

Aquí entran las bases de datos NoSQL ("Not Only SQL" —no solo SQL). Son una familia de bases de datos que **no requieren tablas fijas** y están diseñadas para escenarios específicos donde las bases relacionales se quedan cortas.

Existen varios tipos de bases NoSQL, cada una con una "personalidad" diferente. Vamos a verlas una por una.

---

#### Documentales (MongoDB)

Las bases de datos documentales son como un **pizarrón donde puedes pegar notas adhesivas**: cada nota puede tener la información que necesites, sin seguir un formato preestablecido. Una nota puede tener nombre, precio y foto; otra puede tener nombre, precio, talla, color, material, reseñas y videos; y ambas conviven perfectamente.

MongoDB es la base de datos documental más famosa. En lugar de tablas con filas y columnas, usa **colecciones** de **documentos** en formato JSON (JavaScript Object Notation, que es una forma de organizar datos usando llaves `{}` y pares de "clave: valor").

**¿Para qué se usa?** Catálogos de productos donde cada producto puede tener atributos diferentes, sistemas de gestión de contenido (CMS), aplicaciones que evolucionan rápidamente y necesitan cambiar su estructura de datos con frecuencia.

Observa este documento JSON que representa un producto en un catálogo:

- `"_id": ObjectId("...")` es el identificador único del documento, similar al `PRIMARY KEY` en SQL, pero MongoDB lo genera automáticamente.
- `"nombre": "Laptop HP ProBook"` es un campo de texto (clave: valor).
- `"precios": { ... }` contiene un objeto anidado con diferentes tipos de precio (venta, mayoreo, distribuidor). En SQL esto requeriría una tabla separada de precios, pero aquí está todo dentro del mismo documento.
- `"especificaciones": { ... }` otro objeto anidado para las características técnicas.
- `"inventario": [ ... ]` es un **arreglo** (lista) de objetos, cada uno con el almacén y el stock. En SQL necesitarías una tabla separada para inventario.
- `"categorias": ["electrónicos", "laptops", "negocios"]` es una lista simple de textos. En SQL necesitarías una tabla de categorías y otra tabla intermedia para relacionar productos con categorías.

La gran ventaja es que obtienes toda la información de un producto en **una sola lectura**, sin tener que hacer JOINs entre múltiples tablas.

```javascript
// Ideal para: Catálogos de productos flexibles
{
  "_id": ObjectId("..."),
  "nombre": "Laptop HP ProBook",
  "precios": {
    "venta": 25000,
    "mayoreo": 22000,
    "distribuidor": 19500
  },
  "especificaciones": {
    "ram": "16GB",
    "disco": "512GB SSD",
    "procesador": "Intel i7"
  },
  "inventario": [
    { "almacen": "principal", "stock": 15 },
    { "almacen": "secundario", "stock": 8 }
  ],
  "categorias": ["electrónicos", "laptops", "negocios"]
}
```

---

#### Clave-Valor (Redis)

Las bases de datos clave-valor son las más simples de todas. Piensa en un **casillero con miles de pequeños compartimentos, cada uno con una etiqueta y un contenido**. Para obtener algo, solo necesitas saber la etiqueta correcta. No hay tablas, ni esquemas, ni relaciones —solo pares de "clave" y "valor".

Redis es la base de datos clave-valor más popular. Es extremadamente rápida porque guarda los datos en la memoria RAM (en lugar de en el disco duro), lo que permite leer y escribir en milisegundos.

**¿Para qué se usa?** Caché (guardar resultados de consultas costosas para no repetirlas), sesiones de usuario (carritos de compra), colas de mensajes, y cualquier cosa que necesite acceso ultrarrápido.

Vamos a entender los comandos:
- `SET producto:100:nombre "Laptop HP ProBook"`: Guarda el valor "Laptop HP ProBook" bajo la clave `producto:100:nombre`. La convención de nombres con dos puntos (`:`) es solo una forma de organizar las claves, como si fueran carpetas: "producto/100/nombre".
- `EXPIRE producto:100:stock 3600`: Le asigna un tiempo de vida (TTL —Time To Live) a la clave. Después de 3600 segundos (1 hora), Redis eliminará automáticamente esa clave. Esto es útil para datos temporales como precios promocionales o sesiones de usuario.
- `HSET carrito:usuario:123 producto:100 2`: `HSET` significa "hash set". Redis permite estructuras de datos más complejas que simples valores, como los **hashes** (similares a objetos). Aquí estamos creando un carrito de compras donde dentro del hash `carrito:usuario:123` guardamos pares de producto:cantidad.
- `HGETALL carrito:usuario:123`: Obtiene todos los campos y valores del hash. Es como decir "dame todo lo que hay en este carrito".

```bash
# Ideal para: Caché de catálogos, sesiones de carrito
SET producto:100:nombre "Laptop HP ProBook"
SET producto:100:precio 25000
SET producto:100:stock 15
EXPIRE producto:100:stock 3600  # TTL de 1 hora

# Carrito de compras temporal
HSET carrito:usuario:123 producto:100 2
HSET carrito:usuario:123 producto:200 1
HGETALL carrito:usuario:123
```

---

#### Columnares (Apache Cassandra)

Las bases de datos columnares están diseñadas para manejar **enormes volúmenes de datos** distribuidos en muchos servidores. Mientras que las bases relacionales guardan los datos fila por fila en el disco (primero toda la fila 1, luego toda la fila 2), las bases columnares los guardan columna por columna (primero todos los valores de la columna A, luego todos los de la columna B).

**Piénsalo así:** las bases relacionales son como una lista de compras escrita en un papel (artículo 1: nombre, precio, cantidad; artículo 2: nombre, precio, cantidad...). Las bases columnares son como tener todos los nombres en una lista, todos los precios en otra, todas las cantidades en otra. Esto hace que operaciones como "suma todos los precios" sean mucho más rápidas, porque solo tienes que leer una lista en lugar de leer el papel completo.

Apache Cassandra fue creada por Facebook para la búsqueda de mensajes en el inbox y es conocida por su capacidad de escalar horizontalmente (agregar más servidores) sin límite práctico.

**¿Para qué se usa?** Series temporales (ventas por día, logs de servidores), datos que necesitan alta disponibilidad y tolerancia a particiones (caídas de red), aplicaciones que deben funcionar en múltiples regiones geográficas.

La sintaxis se parece a SQL, pero hay diferencias importantes:
- `PRIMARY KEY ((id_producto), fecha)`: La clave primaria tiene dos partes. La primera, `(id_producto)` entre paréntesis adicionales, es la **clave de partición**: define cómo se distribuyen los datos entre los servidores. Todos los datos de un mismo producto van al mismo servidor. La segunda parte, `fecha`, es la **clave de agrupación** (clustering): dentro de la partición de un producto, los datos se ordenan por fecha.
- `WITH CLUSTERING ORDER BY (fecha DESC)`: Indica que dentro de cada partición, las filas se ordenan de la fecha más reciente a la más antigua. Esto es útil porque lo más común es consultar las ventas más recientes de un producto.

```sql
-- Ideal para: Series temporales de ventas, logs
CREATE TABLE ventas_por_dia (
    id_producto UUID,
    fecha DATE,
    total_vendido DECIMAL,
    unidades_vendidas INT,
    PRIMARY KEY ((id_producto), fecha)
) WITH CLUSTERING ORDER BY (fecha DESC);
```

---

#### Grafos (Neo4j)

Las bases de datos de grafos están diseñadas para manejar **relaciones complejas** entre datos. Piensa en un mapa de metro: las estaciones son los datos y las líneas son las relaciones. O piensa en una red social: las personas son los datos y las "amistades" o "seguimientos" son las relaciones.

**¿Qué las hace especiales?** En una base relacional, para saber "qué productos compraron los clientes que compraron este producto" tendrías que hacer múltiples JOINs (unir tablas). Conforme más niveles de profundidad agregues, más compleja se vuelve la consulta. En una base de grafos, esa misma pregunta se responde con una consulta simple y elegante.

Neo4j es la base de datos de grafos más conocida. Usa un lenguaje llamado Cypher (que no es SQL, pero se ve similar). En Cypher:
- Los paréntesis `()` representan **nodos** (entidades: clientes, productos).
- Los guiones y flechas `-[:RELACION]->` representan las **relaciones** entre nodos.
- Las llaves `{}` contienen las **propiedades** de nodos y relaciones.

Analicemos el código:
1. `CREATE (c:Cliente {nombre: 'María', email: 'maria@email.com'})`: Crea un nodo de tipo `Cliente` con las propiedades nombre y email.
2. `CREATE (p:Producto {nombre: 'Laptop HP', precio: 25000})`: Crea un nodo de tipo `Producto`.
3. `CREATE (c)-[:COMPRO {fecha: '2024-01-15', cantidad: 1}]->(p)`: Crea una relación de tipo `COMPRO` desde el cliente hacia el producto. La relación también tiene propiedades (fecha, cantidad).
4. La consulta final hace la famosa recomendación "los clientes que compraron esto también compraron...":
   - `MATCH (c:Cliente)-[:COMPRO]->(p:Producto {nombre: 'Laptop HP'})`: Encuentra clientes que compraron la Laptop HP.
   - `MATCH (c)-[:COMPRO]->(otros:Producto)`: De esos clientes, encuentra otros productos que también compraron.
   - `WHERE otros.nombre <> 'Laptop HP'`: Excluye la misma laptop.
   - `RETURN otros.nombre, COUNT(*) AS frecuencia ORDER BY frecuencia DESC LIMIT 5`: Devuelve los 5 productos más comprados por esos clientes.

```cypher
// Ideal para: Recomendaciones, detección de fraude
CREATE (c:Cliente {nombre: 'María', email: 'maria@email.com'})
CREATE (p:Producto {nombre: 'Laptop HP', precio: 25000})
CREATE (c)-[:COMPRO {fecha: '2024-01-15', cantidad: 1}]->(p)

// Recomendación: clientes que compraron esto también compraron...
MATCH (c:Cliente)-[:COMPRO]->(p:Producto {nombre: 'Laptop HP'})
MATCH (c)-[:COMPRO]->(otros:Producto)
WHERE otros.nombre <> 'Laptop HP'
RETURN otros.nombre, COUNT(*) AS frecuencia
ORDER BY frecuencia DESC
LIMIT 5;
```

---

## Comparativa para el Sector Comercial

Ahora que conoces los principales tipos de bases de datos, aquí tienes una tabla comparativa para que puedas contrastar rápidamente sus características. Fíjate en las columnas que más te importen según tu proyecto:

- **Costo:** Desde gratuito (MySQL, PostgreSQL) hasta costos muy altos (Oracle).
- **Facilidad:** Qué tan fácil es aprenderlo y usarlo.
- **Transacciones:** Si soporta ACID (crucial para ventas y finanzas).
- **JSON:** Capacidad para manejar datos en formato JSON.
- **GIS:** Capacidad para manejar datos geográficos/espaciales.
- **Replicación:** Facilidad para tener copias en múltiples servidores.
- **Particionado:** Capacidad para dividir tablas grandes en segmentos.

| Característica | MySQL | PostgreSQL | SQL Server | Oracle | MongoDB |
|---|---|---|---|---|---|
| Costo | Gratuito | Gratuito | $$$ | $$$$ | Gratuito |
| Facilidad | Alta | Media | Alta | Media | Alta |
| Transacciones | Sí | Sí | Sí | Sí | Limitado |
| JSON | Básico | Excelente | Bueno | Bueno | Nativo |
| GIS | Básico | Excelente | Bueno | Bueno | Limitado |
| Replicación | Buena | Excelente | Excelente | Excelente | Excelente |
| Particionado | Limitado | Bueno | Excelente | Excelente | Nativo |
| Ideal para | PyME, web | Analítica | Empresa | Corporación | Catálogos |

---

## Bases de Datos en la Nube

Hasta ahora hemos hablado de instalar bases de datos en tus propios servidores (on-premise). Pero cada vez es más común usar **bases de datos en la nube**, donde alguien más (Amazon, Google, Microsoft) se encarga de mantener los servidores, las copias de seguridad, las actualizaciones y la seguridad. Tú solo te preocupas de tus datos y tus consultas.

¿La analogía? Es como contratar un servicio de cochera en lugar de construir tu propio garaje. Pagas por uso, no te preocupas por el mantenimiento, y si necesitas más espacio, lo obtienes al instante.

### Amazon RDS

Amazon RDS (Relational Database Service) es un servicio de Amazon Web Services que te permite tener bases de datos MySQL, PostgreSQL, SQL Server, Oracle o MariaDB sin instalar ni administrar nada. Solo creas la base de datos con unos clics y comienzas a usarla.

Entre sus ventajas: hace backups automáticos, puede escalar el tamaño del servidor con solo cambiar una configuración, y puede replicar tu base de datos en múltiples zonas geográficas para alta disponibilidad.

```sql
-- Base de datos administrada compatible con MySQL/PostgreSQL/Oracle/SQL Server
-- Ventajas: Auto-scaling, backups automáticos, multi-AZ
```

### Google Cloud SQL

Google Cloud SQL es el equivalente de Google, optimizado para integrarse con el resto de servicios de Google Cloud Platform, especialmente BigQuery (el data warehouse de Google para analítica masiva).

```sql
-- Base de datos administrada con alta disponibilidad
-- Integración nativa con BigQuery para analítica
```

### Azure SQL Database

Azure SQL Database es la versión como servicio de SQL Server en la nube de Microsoft. Ofrece características como "elastic pools" (grupos de bases de datos que comparten recursos) y opción serverless (sin servidor fijo, pagas solo cuando usas).

```sql
-- SQL Server como servicio
-- Elastic pools, geo-replicación, serverless
```

### Snowflake (Data Warehouse)

Snowflake no es una base de datos tradicional: es un **data warehouse** (almacén de datos) diseñado específicamente para analítica masiva en la nube. ¿La diferencia clave? Separa el almacenamiento (dónde guardas los datos) del cómputo (la capacidad de procesamiento). Esto significa que puedes tener terabytes de datos guardados sin pagar por capacidad de cómputo, y solo "encender" más servidores cuando necesites hacer consultas pesadas.

```sql
-- Data warehouse cloud para analítica masiva
-- Separación de almacenamiento y cómputo
CREATE WAREHOUSE analítica_ventas WITH WAREHOUSE_SIZE = 'X-LARGE';
CREATE DATABASE data_warehouse_ventas;
```

---

## Modelos Híbridos (Políglota Persistente)

Una idea importante que debes tener desde el principio: **no estás obligado a usar una sola base de datos**. Las empresas modernas usan múltiples tecnologías, cada una para lo que hace mejor. Esto se llama "persistencia políglota" (usar diferentes bases de datos según la necesidad).

Imagina tu negocio como una cocina profesional:
- Para las transacciones del día a día (ventas, compras, inventario) usas **MySQL**, como si fuera tu estufa principal.
- Para el catálogo de productos (que cambia constantemente y tiene muchos atributos flexibles) usas **MongoDB**, como tu refrigerador donde guardas ingredientes variados.
- Para los reportes y análisis (ventas por región, productos más vendidos) usas **PostgreSQL**, como tu horno de precisión.
- Para la caché (datos temporales que necesitas muy rápido) usas **Redis**, como tu microondas.
- Para búsquedas de texto completo usas **Elasticsearch**, como tu libro de recetas indexado.
- Para colas de mensajes (comunicación entre sistemas) usas **RabbitMQ o Kafka**, como el sistema de campanas y timbres entre cocineros.

```
                            ┌─────────────────────────────────────────────┐
                            │                 CAPA DE APLICACIÓN           │
                            ├─────────────────────────────────────────────┤
                            │  Transaccional  │   Catálogo    │   Analítica│
                            │  (MySQL)        │   (MongoDB)   │ (PostgreSQL│
                            │  Ventas         │   Productos   │   Reporting│
                            │  Compras        │   Precios     │   Dashboards│
                            │  Inventario     │   Imágenes    │   BI       │
                            │  Clientes       │   Variantes   │   ML       │
                            ├─────────────────┴───────────────┴──────────────┤
                            │              CACHÉ (Redis)                    │
                            │            BÚSQUEDA (Elasticsearch)           │
                            │            COLAS (RabbitMQ/Kafka)             │
                            └──────────────────────────────────────────────┘
```

---

## Cómo Elegir la Base de Datos Correcta

Elegir una base de datos no es como elegir un sabor de helado (donde la respuesta correcta es "el que más te guste"). Es más como elegir una herramienta para un trabajo específico. No usarías un martillo para atornillar, ni un desarmador para clavar. De la misma manera, cada base de datos tiene su propósito ideal.

Aquí tienes algunas preguntas clave que debes hacerte antes de elegir:

### Preguntas Clave

1. **¿Volumen de datos?**
   - Si tu negocio genera menos de 100 GB de datos (lo típico de una PyME), cualquiera de las bases relacionales funcionará sin problemas.
   - Entre 100 GB y 10 TB (una empresa mediana-grande), necesitas algo más robusto como PostgreSQL, SQL Server u Oracle.
   - Más de 10 TB (grandes corporaciones), entras al terreno de Oracle, Snowflake o BigQuery.

2. **¿Tipo de carga de trabajo?**
   - **OLTP** (Online Transaction Processing): Muchas transacciones pequeñas y rápidas, como las ventas en una tienda. Ideal: MySQL, PostgreSQL.
   - **OLAP** (Online Analytical Processing): Consultas pesadas que analizan grandes volúmenes de datos, como reportes anuales. Ideal: bases columnares, Snowflake.
   - **HTAP** (Hybrid Transactional/Analytical Processing): Una mezcla de ambos. Ideal: PostgreSQL, SQL Server.

3. **¿Presupuesto?**
   - Cero pesos: MySQL, PostgreSQL, MariaDB.
   - Medio: SQL Server Standard.
   - Alto: Oracle, SQL Server Enterprise.

4. **¿Equipo?**
   - Si tu equipo es full-stack web: MySQL es la opción más natural.
   - Si trabajan con analítica de datos: PostgreSQL es el rey.
   - Si están en el ecosistema Microsoft (.NET, Azure): SQL Server es la decisión obvia.
   - Si son una empresa Java corporativa: Oracle es el estándar.

### Casos de Uso en el Sector

| Negocio | Base de Datos Recomendada | Razón |
|---|---|---|
| Tienda en línea pequeña | MySQL/MariaDB | Bajo costo, fácil |
| ERP PyME | PostgreSQL | Robustez, costo cero |
| Retail grande | SQL Server | BI, integración Office |
| Distribuidora | Oracle | Confiabilidad, soporte |
| E-commerce grande | MySQL + MongoDB | Flexibilidad catálogo |
| Data Warehouse | Snowflake | Analítica masiva |

---

## 📊 Ejemplo del Mundo Real: Estrategia Multi-Base de Datos

Para cerrar este capítulo, veamos un ejemplo concreto de una empresa de retail con 500 tiendas. Este caso te ayudará a entender cómo se combinan diferentes bases de datos en la vida real.

**El escenario:** Una cadena de tiendas departamentales necesita:
- Procesar ventas en tiempo real en cada sucursal
- Consolidar la información de todas las sucursales en una base central
- Mantener un catálogo de productos flexible (con imágenes, variantes por talla y color, etc.)
- Ofrecer una tienda en línea con carritos de compra rápidos
- Hacer análisis anuales de rentabilidad y segmentación de clientes

La solución usa 5 bases de datos diferentes, cada una para su propósito:

1. **Cada tienda tiene MySQL local (POS):** En cada sucursal hay un servidor MySQL que registra las ventas en tiempo real. Esto es importante porque si falla la conexión a internet, la tienda puede seguir vendiendo.

2. **Base central PostgreSQL (consolidación):** Cada noche, un proceso automático recoge los datos de todas las sucursales y los consolida en una base PostgreSQL central. Allí se unifican los catálogos de productos y clientes.

3. **MongoDB para catálogo de productos:** Los productos tienen atributos muy variados (un televisor tiene pulgadas y resolución; una camisa tiene talla y color). MongoDB permite manejar esta flexibilidad sin esfuerzo.

4. **Redis para caché de precios y carritos:** Los precios actualizados y los carritos de compra en línea se guardan en Redis para acceso ultrarrápido.

5. **Snowflake para analítica anual:** Los datos históricos se envían a Snowflake para hacer forecasting (pronósticos de ventas), segmentación RFM (Recencia, Frecuencia, Monto) y otros análisis pesados.

```sql
-- Empresa de retail con 500 tiendas
-- Necesita: Transacciones en tiempo real + Analítica centralizada

-- 1. Cada tienda tiene MySQL local (POS)
CREATE DATABASE tienda_001;
-- Tablas: ventas, productos, clientes (solo locales)

-- 2. Base central PostgreSQL (consolidación)
CREATE DATABASE central_ventas;
-- Recibe datos de todas las tiendas cada noche
-- Tablas: ventas_consolidadas, clientes_unicos, catalogos_centrales

-- 3. MongoDB para catálogo de productos con imágenes
db.createCollection("catalogo_productos");
-- Documentos flexibles con especificaciones, imágenes, variantes

-- 4. Redis para caché de precios y carritos
-- Tiempo real: precios actualizados, carritos de compra en línea

-- 5. Snowflake para analítica anual
-- Datos históricos, forecasting, segmentación RFM
```

---

## Anterior: [01 Que Es Sql](../01-introduccion/01-que-es-sql.md)
## Siguiente: [03 Configuracion Entorno](../01-introduccion/03-configuracion-entorno.md)

[Volver al índice](../README.md)
