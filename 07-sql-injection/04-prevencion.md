# 7.4 Prevención de SQL Injection

## La Regla de Oro

**NUNCA confíes en la entrada del usuario. Siempre separa el código de los datos.**

## 1. Prepared Statements (Consultas Parametrizadas)

### PHP (MySQLi)
```php
<?php
// ✅ SEGURO: Prepared Statement
$stmt = $conn->prepare("SELECT * FROM productos WHERE id = ? AND activo = ?");
$stmt->bind_param("ii", $id, $activo);
$id = $_GET['id'];
$activo = 1;
$stmt->execute();
$result = $stmt->get_result();
?>
```

### PHP (PDO)
```php
<?php
// ✅ SEGURO: PDO con parámetros
$stmt = $pdo->prepare("INSERT INTO clientes (nombre, email, telefono) VALUES (?, ?, ?)");
$stmt->execute([$_POST['nombre'], $_POST['email'], $_POST['telefono']]);

// ✅ Con parámetros nombrados
$stmt = $pdo->prepare("UPDATE productos SET precio_venta = :precio WHERE id_producto = :id");
$stmt->execute([':precio' => $_POST['precio'], ':id' => $_POST['id']]);
?>
```

### Python
```python
# ✅ SEGURO: MySQL Connector
cursor = conn.cursor(dictionary=True)
cursor.execute("SELECT * FROM clientes WHERE email = %s AND activo = %s", 
               (email, True))

# ✅ SEGURO: SQLAlchemy ORM
resultado = session.query(Cliente).filter(
    Cliente.email == email,
    Cliente.activo == True
).all()
```

### Java (JDBC)
```java
// ✅ SEGURO: PreparedStatement
String sql = "SELECT * FROM productos WHERE id_categoria = ? AND precio BETWEEN ? AND ?";
PreparedStatement stmt = conn.prepareStatement(sql);
stmt.setInt(1, categoria);
stmt.setDouble(2, precioMin);
stmt.setDouble(3, precioMax);
ResultSet rs = stmt.executeQuery();
```

### C# (.NET)
```csharp
// ✅ SEGURO: SqlCommand con parámetros
using var cmd = new SqlCommand(
    "SELECT * FROM clientes WHERE email = @email AND activo = @activo", conn);
cmd.Parameters.AddWithValue("@email", email);
cmd.Parameters.AddWithValue("@activo", true);
var reader = await cmd.ExecuteReaderAsync();
```

### Node.js
```javascript
// ✅ SEGURO: MySQL2 con placeholders
const [rows] = await conn.execute(
    'SELECT * FROM productos WHERE id_categoria = ? AND precio_venta BETWEEN ? AND ?',
    [categoria, precioMin, precioMax]
);

// ✅ SEGURO: pg (PostgreSQL)
const result = await pool.query(
    'SELECT * FROM clientes WHERE email = $1 AND activo = $2',
    [email, true]
);
```

## 2. ORM (Object-Relational Mapping)

Los ORM manejan la seguridad automáticamente.

### SQLAlchemy (Python)
```python
# ✅ SEGURO: El ORM sanitiza automáticamente
nuevo_cliente = Cliente(
    nombre=request.form['nombre'],
    email=request.form['email'],
    credito_limite=request.form['credito']
)
session.add(nuevo_cliente)
session.commit()

# ✅ Consultas seguras con ORM
clientes = session.query(Cliente).filter(
    Cliente.credito_limite > 10000,
    Cliente.activo == True
).all()
```

### Hibernate (Java)
```java
// ✅ SEGURO: Hibernate
Cliente cliente = new Cliente();
cliente.setNombre(request.getParameter("nombre"));
cliente.setEmail(request.getParameter("email"));
session.save(cliente);

// HQL (también seguro)
Query query = session.createQuery(
    "FROM Cliente WHERE creditoLimite > :limite AND activo = :activo");
query.setParameter("limite", 10000);
query.setParameter("activo", true);
```

### Entity Framework (C#)
```csharp
// ✅ SEGURO: EF Core
var clientes = await context.Clientes
    .Where(c => c.CreditoLimite > 10000 && c.Activo)
    .ToListAsync();
```

## 3. Stored Procedures

```sql
-- ✅ SEGURO: Store Procedure con parámetros
DELIMITER //
CREATE PROCEDURE sp_buscar_productos(
    IN p_busqueda VARCHAR(100),
    IN p_categoria INT,
    IN p_precio_max DECIMAL(12,2)
)
BEGIN
    SELECT * FROM productos
    WHERE activo = TRUE
      AND (p_busqueda IS NULL OR nombre_comercial LIKE CONCAT('%', p_busqueda, '%'))
      AND (p_categoria IS NULL OR id_categoria = p_categoria)
      AND (p_precio_max IS NULL OR precio_venta <= p_precio_max);
END //
DELIMITER ;
```

```php
<?php
// ✅ Llamada segura al procedure
$stmt = $conn->prepare("CALL sp_buscar_productos(?, ?, ?)");
$stmt->bind_param("sid", $busqueda, $categoria, $precio_max);
$stmt->execute();
?>
```

## 4. Validación de Entrada

### Validar Tipo de Dato
```php
<?php
// ✅ Validar que sea entero
$id = filter_input(INPUT_GET, 'id', FILTER_VALIDATE_INT);
if ($id === false || $id === null) {
    die("ID inválido");
}

// ✅ Validar email
$email = filter_input(INPUT_POST, 'email', FILTER_VALIDATE_EMAIL);

// ✅ Validar rango
$precio = filter_input(INPUT_POST, 'precio', FILTER_VALIDATE_FLOAT, 
    ['options' => ['min_range' => 0, 'max_range' => 999999]]);
?>
```

### Whitelist vs Blacklist
```php
<?php
// ❌ MAL: Blacklist (siempre incompleta)
$input = str_replace(["'", '"', ";", "--"], "", $_GET['param']);

// ✅ BIEN: Whitelist (solo permite lo seguro)
$orden_permitido = ['nombre', 'precio', 'fecha'];
$orden = in_array($_GET['orden'], $orden_permitido) ? $_GET['orden'] : 'nombre';

$direccion_permitido = ['ASC', 'DESC'];
$direccion = in_array($_GET['dir'], $direccion_permitido) ? $_GET['dir'] : 'ASC';
?>
```

## 5. Escapado de Caracteres

Solo usar cuando NO se puedan usar prepared statements:

```php
<?php
// MySQL
$nombre = mysqli_real_escape_string($conn, $_POST['nombre']);

// PostgreSQL
$nombre = pg_escape_string($conn, $_POST['nombre']);

// SQL Server
$nombre = str_replace("'", "''", $_POST['nombre']);
?>
```

⚠️ **El escapado es menos seguro que prepared statements. Siempre preferir parámetros.**

## 6. Principio de Mínimo Privilegio

```sql
-- ❌ MAL: Usuario con todos los permisos
GRANT ALL PRIVILEGES ON ventas.* TO 'app'@'%';

-- ✅ BIEN: Mínimos permisos necesarios
-- Usuario para operaciones normales
GRANT SELECT, INSERT, UPDATE ON ventas.facturas TO 'app_ventas'@'localhost';
GRANT SELECT, INSERT, UPDATE ON ventas.productos TO 'app_ventas'@'localhost';
GRANT SELECT, INSERT ON ventas.clientes TO 'app_ventas'@'localhost';

-- Usuario de solo lectura para reportes
GRANT SELECT ON ventas.* TO 'reportes'@'localhost';

-- Usuario de administración (solo desde IP interna)
GRANT ALL PRIVILEGES ON ventas.* TO 'dba'@'10.0.0.%' IDENTIFIED BY 'pass';
```

## 7. WAF (Web Application Firewall)

### ModSecurity (OWASP CRS)
```apache
# Reglas OWASP Core Rule Set para SQL Injection
SecRule ARGS "@detectSQLi" "id:942100,phase:2,deny,status:403,msg:'SQL Injection Detectada'"
SecRule REQUEST_COOKIES "@detectSQLi" "id:942110,phase:2,deny,status:403"
SecRule REQUEST_HEADERS "@detectSQLi" "id:942120,phase:2,deny,status:403"
```

### Cloudflare WAF
```
# Reglas gestionadas de Cloudflare
# Bloquear patrones de SQL Injection
# Protección OWASP Top 10
# Rate limiting: 100 requests / 10 minutos
```

## 8. Configuración Segura del Servidor

### MySQL
```ini
[mysqld]
# No mostrar información en errores
sql_mode = STRICT_ALL_TABLES,NO_ENGINE_SUBSTITUTION

# Deshabilitar carga de archivos locales
local_infile = 0

# No permitir acceso remoto sin necesidad
bind-address = 127.0.0.1

# Deshabilitar funciones peligrosas
--secure-file-priv = /tmp

# Logging de consultas sospechosas
log_warnings = 2
```

### PostgreSQL
```ini
# pg_hba.conf
# Solo conexiones locales para aplicaciones
host    ventas    app_ventas    127.0.0.1/32    scram-sha-256

# postgresql.conf
ssl = on
ssl_cert_file = 'server.crt'
listen_addresses = 'localhost'
```

## 9. Monitoreo y Detección

```sql
-- Consultas sospechosas
SELECT * FROM mysql.general_log 
WHERE argument REGEXP 'UNION|SLEEP|0x|--|#' 
AND argument NOT LIKE '%mysql%'
ORDER BY event_time DESC
LIMIT 100;

-- Alertas de WAF
SELECT 
    ip_address,
    COUNT(*) AS ataques,
    GROUP_CONCAT(DISTINCT payload) AS payloads
FROM waf_logs
WHERE accion = 'BLOCKED'
  AND tipo_ataque = 'SQL_INJECTION'
  AND fecha >= DATE_SUB(NOW(), INTERVAL 1 HOUR)
GROUP BY ip_address
HAVING ataques > 10;
```

### PHP - Sistema de Detección Simple
```php
<?php
function detectar_sqli($input) {
    $patrones = [
        '/\bUNION\b.*\bSELECT\b/i',
        '/\bSLEEP\s*\(/i',
        '/\bBENCHMARK\s*\(/i',
        '/\bINFORMATION_SCHEMA\b/i',
        '/\bLOAD_FILE\b/i',
        '/\bINTO\s+OUTFILE\b/i',
        '/\bINTO\s+DUMPFILE\b/i',
        '/\bxp_cmdshell\b/i',
        '/\bWAITFOR\s+DELAY\b/i',
        '/\bpg_sleep\b/i',
        '/\bEXEC\b.*\bxp_/i',
    ];
    
    foreach ($patrones as $patron) {
        if (preg_match($patron, $input)) {
            return true;  // Posible ataque detectado
        }
    }
    return false;
}

// Uso
if (detectar_sqli($_GET['id'])) {
    error_log("SQL Injection detectado: " . $_SERVER['REMOTE_ADDR']);
    header('HTTP/1.1 403 Forbidden');
    die("Acceso denegado");
}
?>
```

## 10. Checklist de Seguridad

- [ ] **Prepared Statements** en TODAS las consultas con datos variables
- [ ] **ORM** configurado correctamente (sin raw SQL)
- [ ] **Validación de entrada** (tipo, longitud, rango, whitelist)
- [ ] **Mínimo privilegio** en usuarios de BD
- [ ] **WAF** configurado y actualizado
- [ ] **Errores de BD** no mostrados al usuario
- [ ] **Backups** automáticos y probados
- [ ] **Logs** de actividad sospechosa
- [ ] **Parches** de seguridad al día
- [ ] **Pruebas de penetración** regulares
- [ ] **Firewall de BD** (solo IPs autorizadas)
- [ ] **Cifrado** de datos sensibles en BD

## Resumen

```
✅ HACER:
  - Prepared Statements / ORM
  - Validación de entrada (whitelist)
  - Mínimo privilegio
  - WAF
  - Logs y monitoreo

❌ NO HACER:
  - Concatenar strings SQL
  - Escapar como única defensa
  - Mostrar errores de BD
  - Usar root/admin en aplicaciones
  - Confiar en entrada del usuario
```

---

## Anterior: [03 Deteccion Y Exploit](../07-sql-injection/03-deteccion-y-exploit.md)
## Siguiente: [05 Ejemplos Practicos Ventas](../07-sql-injection/05-ejemplos-practicos-ventas.md)

[Volver al índice](../README.md)
