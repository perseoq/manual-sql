# 7.1 ¿Qué es SQL Injection?

## Definición

SQL Injection (Inyección SQL) es una vulnerabilidad de seguridad que permite a un atacante interferir con las consultas que una aplicación hace a su base de datos. Permite al atacante ver, modificar o eliminar datos a los que normalmente no debería tener acceso.

### Impacto en el Negocio

| Impacto | Ejemplo en Ventas/Compras/Inventarios |
|---------|---------------------------------------|
| **Confidencialidad** | Robo de datos de clientes, precios, proveedores |
| **Integridad** | Modificación de precios, stocks, facturas |
| **Disponibilidad** | Borrado de tablas, denegación de servicio |
| **Autenticación** | Bypass de login, suplantación de usuarios |
| **Cumplimiento** | Violación de GDPR, PCI-DSS, SOX, LFPDPPP |

## Cómo Funciona

### Ejemplo Vulnerable (Código PHP)
```php
<?php
// ⚠️ CÓDIGO VULNERABLE
$usuario = $_POST['usuario'];
$password = $_POST['password'];

$sql = "SELECT * FROM usuarios WHERE usuario = '$usuario' AND password = '$password'";
$resultado = mysqli_query($conn, $sql);
?>
```

### Ataque Básico - Bypass de Login
El atacante ingresa:
```
Usuario: admin
Password: ' OR '1'='1
```

La consulta se convierte en:
```sql
SELECT * FROM usuarios WHERE usuario = 'admin' AND password = '' OR '1'='1';
-- ¡Siempre retorna TRUE! El atacante accede sin contraseña.
```

### Escalamiento del Ataque
```
Usuario: admin' --
Password: cualquier_cosa
```

Resultado:
```sql
SELECT * FROM usuarios WHERE usuario = 'admin' --' AND password = 'cualquier_cosa';
-- Todo después de -- es comentario. Solo valida usuario.
```

## Historia de Ataques Famosos

### 1. Heartland Payment Systems (2008)
- **Impacto**: Robo de 134 millones de tarjetas de crédito
- **Método**: SQL Injection en sistema de pagos
- **Costo**: $140 millones en multas y remediación

### 2. Sony Pictures (2011)
- **Impacto**: Robo de datos de 77 millones de usuarios
- **Método**: SQL Injection en servidores web
- **Costo**: $170 millones en daños

### 3. TalkTalk (2015)
- **Impacto**: Datos de 157,000 clientes robados
- **Método**: SQL Injection en portal web
- **Costo**: $60 millones, 400,000 clientes perdidos

### 4. British Airways (2018)
- **Impacto**: Robo de datos de 380,000 transacciones
- **Método**: SQL Injection en sistema de reservas
- **Multa**: $230 millones (GDPR)

## ¿Cómo se Produce?

### Flujo del Ataque
```
1. Atacante encuentra un campo de entrada (formulario, URL API, etc.)
2. Prueba caracteres especiales: ' " ; -- /* */ #
3. Si la aplicación no sanitiza, inyecta código SQL malicioso
4. El código se ejecuta en la base de datos
5. Atacante extrae/modifica datos o ejecuta comandos
```

### Puntos de Entrada Comunes
```
Formularios de login      → usuario, password
Búsqueda de productos     → término de búsqueda
Filtros de catálogo       → precio, categoría
Parámetros URL            → id=1, page=2
Campos ocultos            → datos enviados desde el cliente
Cookies                   → session_id, preferencias
Headers HTTP              → User-Agent, Referer
APIs REST/GraphQL         → parámetros JSON
```

## Tipos de Daños

### Robo de Datos
```sql
-- Obtener todas las tablas de la base de datos
' UNION SELECT table_name, NULL FROM information_schema.tables --

-- Obtener usuarios y contraseñas
' UNION SELECT usuario, password FROM usuarios --
```

### Modificación de Datos
```sql
-- Cambiar precio de productos
'; UPDATE productos SET precio_venta = 1 WHERE id_producto > 0; --

-- Transferir fondos
'; UPDATE cuentas SET saldo = saldo + 10000 WHERE id_cuenta = 999; --
```

### Denegación de Servicio
```sql
-- Borrar tablas
'; DROP TABLE facturas; --

-- Llenar disco
'; CREATE TABLE basura (id INT PRIMARY KEY AUTO_INCREMENT, datos TEXT);
   INSERT INTO basura (datos) SELECT REPEAT('A', 10000); --
```

### Ejecución de Comandos (si el DBMS permite)
```sql
-- MySQL: xp_cmdshell no disponible, pero en SQL Server:
'; EXEC xp_cmdshell('format D: /y'); --
```

## Costo de una Inyección SQL

| Concepto | Costo Estimado |
|----------|---------------|
| Multas regulatorias (GDPR) | Hasta 20M EUR o 4% factura anual |
| Notificación a clientes | $50-200 por registro |
| Forense digital | $10,000-$100,000 |
| Remediación técnica | $20,000-$200,000 |
| Pérdida de clientes | 20-40% de la base |
| Daño reputacional | 2-5 años en recuperarse |
| Tiempo de inactividad | $5,000-$100,000 por hora |

## Frecuencia y Estadísticas

- **OWASP Top 10**: SQL Injection consistentemente en el TOP 3
- **Prevalencia**: Presente en ~30% de las aplicaciones web
- **Explotabilidad**: Alta (herramientas automatizadas como SQLMap)
- **Detección**: Moderada (WAFs pueden detectar, no siempre)
- **Impacto**: Severo (pérdida total de datos)

### Herramientas Automatizadas de Ataque
```
SQLMap              → Automatización de detección y explotación
Havij               → GUI para SQL Injection (ya no actualizado)
Burp Suite          → Proxy de interceptación con scanner
Acunetix            → Scanner de vulnerabilidades
sqlsus              → Herramienta Open Source
NoSQLMap            → Para bases de datos NoSQL
BBQSQL              → Blind SQL Injection framework
```

## Principio de Defensa en Profundidad

```
┌─────────────────────────────────────────────┐
│             CAPA DE PRESENTACIÓN             │
│   Validación de entrada (cliente + servidor) │
├─────────────────────────────────────────────┤
│              CAPA DE APLICACIÓN              │
│   Prepared Statements / ORM / Escape APIs    │
├─────────────────────────────────────────────┤
│              CAPA DE BASE DE DATOS            │
│   Principio de mínimo privilegio             │
│   Usuarios limitados por función             │
├─────────────────────────────────────────────┤
│              CAPA DE RED                     │
│   WAF (Web Application Firewall)             │
│   Firewall de base de datos                  │
├─────────────────────────────────────────────┤
│              CAPA DE MONITOREO               │
│   IDS/IPS, logs, auditoría, alertas         │
└─────────────────────────────────────────────┘
```

## El Problema Raíz

SQL Injection ocurre cuando se **mezclan código y datos**:

```sql
-- CÓDIGO + DATOS mezclados (vulnerable)
$sql = "SELECT * FROM productos WHERE id = " . $_GET['id'];

-- CÓDIGO separado de DATOS (seguro)
$stmt = $conn->prepare("SELECT * FROM productos WHERE id = ?");
$stmt->bind_param("i", $_GET['id']);
```

🔴 **La inyección SQL es 100% prevenible con buenas prácticas de programación.**

---

## Anterior: [06 Performance Tuning](../06-sql-avanzado/06-performance-tuning.md)
## Siguiente: [02 Tipos De Inyeccion](../07-sql-injection/02-tipos-de-inyeccion.md)

[Volver al índice](../README.md)
