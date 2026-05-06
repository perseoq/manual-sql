# 8.1 Hardening de Bases de Datos

## Principios de Hardening

Endurecimiento (hardening) es el proceso de asegurar un sistema reduciendo su superficie de ataque.

### Checklist General
```
□ Versión actualizada y parcheada
□ Usuarios con mínimo privilegio
□ Autenticación fuerte
□ Red segura (firewall, VLAN)
□ Cifrado en tránsito y reposo
□ Logs y auditoría
□ Backups seguros
□ Monitoreo continuo
```

## MySQL / MariaDB Hardening

### 1. Instalación Segura
```bash
# Ejecutar script de seguridad
mysql_secure_installation

# Este script:
# - Establece contraseña de root
# - Elimina usuarios anónimos
# - Deshabilita login remoto de root
# - Elimina base de datos de prueba
# - Recarga privilegios
```

### 2. Configuración de Red
```ini
[mysqld]
# Solo escuchar en localhost si no se necesita acceso remoto
bind-address = 127.0.0.1

# Puerto no estándar (cambiar de 3306 a otro)
port = 3307

# Deshabilitar skip-networking si se necesita red
# skip-networking
```

### 3. Gestión de Usuarios
```sql
-- Eliminar usuarios anónimos
DELETE FROM mysql.user WHERE User = '';

-- Eliminar usuarios sin password
DELETE FROM mysql.user WHERE authentication_string = '';

-- Renombrar root (dificulta ataques)
RENAME USER 'root'@'localhost' TO 'dba'@'localhost';

-- Eliminar cuentas por defecto
DROP USER IF EXISTS ''@'localhost';
DROP USER IF EXISTS ''@'hostname';
DROP USER IF EXISTS 'root'@'%';

-- Crear usuarios con nombre descriptivo
CREATE USER 'app_ventas'@'10.0.1.%' IDENTIFIED BY 'Str0ng!P@ssw0rd';

-- Aplicar cambios
FLUSH PRIVILEGES;
```

### 4. Permisos Mínimos
```sql
-- ❌ MAL: Demasiados permisos
GRANT ALL PRIVILEGES ON *.* TO 'app'@'%';

-- ✅ BIEN: Permisos específicos
GRANT SELECT, INSERT, UPDATE ON ventas.facturas TO 'app_ventas'@'10.0.1.%';
GRANT SELECT ON ventas.clientes TO 'app_ventas'@'10.0.1.%';
GRANT EXECUTE ON PROCEDURE ventas.sp_registrar_venta TO 'app_ventas'@'10.0.1.%';

-- Usuario de solo lectura para reportes
CREATE USER 'reportes'@'10.0.2.%' IDENTIFIED BY 'R3port3s!2024';
GRANT SELECT ON ventas.* TO 'reportes'@'10.0.2.%';

-- Usuario de backup
CREATE USER 'backup'@'localhost' IDENTIFIED BY 'B4ckup!2024';
GRANT SELECT, RELOAD, LOCK TABLES, REPLICATION CLIENT ON *.* TO 'backup'@'localhost';
```

### 5. Configuración de Seguridad
```ini
[mysqld]
# Autenticación
default_authentication_plugin = sha256_password
plugin-load = sha256_password.so

# Contraseñas
validate_password.length = 12
validate_password.mixed_case_count = 1
validate_password.number_count = 1
validate_password.special_char_count = 1

# Límites de conexión
max_connect_errors = 5
max_connections = 500
connect_timeout = 10

# Deshabilitar funciones peligrosas
local_infile = 0
skip_show_database
safe-user-create

# Logs
log_error = /var/log/mysql/error.log
log_warnings = 2
general_log = 0
slow_query_log = 1

# SSL
ssl-ca = /etc/mysql/ssl/ca.pem
ssl-cert = /etc/mysql/ssl/server-cert.pem
ssl-key = /etc/mysql/ssl/server-key.pem
require_secure_transport = ON
```

### 6. Firewall de Base de Datos
```bash
# iptables: Solo permitir IPs específicas
iptables -A INPUT -p tcp --dport 3306 -s 10.0.1.0/24 -j ACCEPT
iptables -A INPUT -p tcp --dport 3306 -j DROP

# Usar ProxySQL como gateway de base de datos
# ProxySQL permite: filtrado de consultas, caching, routing, firewall SQL
```

### 7. Archivos y Directorios
```bash
# Permisos correctos
chmod 750 /etc/mysql/
chmod 640 /etc/mysql/my.cnf
chmod 700 /var/lib/mysql/

# Dueño correcto
chown -R mysql:mysql /etc/mysql/
chown -R mysql:mysql /var/lib/mysql/

# Audit files
chmod 640 /var/log/mysql/*.log
```

## PostgreSQL Hardening

### 1. Configuración de Red (pg_hba.conf)
```conf
# Solo conexiones SSL
hostssl ventas app_ventas 10.0.1.0/24 scram-sha-256
hostssl ventas reportes 10.0.2.0/24 scram-sha-256

# Rechazar todo lo demás
host all all 0.0.0.0/0 reject
host all all ::0/0 reject
```

### 2. Configuración del Servidor (postgresql.conf)
```ini
listen_addresses = '10.0.1.10'
port = 5432
ssl = on
ssl_cert_file = '/etc/ssl/certs/server.crt'
ssl_key_file = '/etc/ssl/private/server.key'

# Autenticación
password_encryption = scram-sha-256
krb_server_keyfile = ''

# Límites
max_connections = 200
superuser_reserved_connections = 5

# Logs
log_destination = 'syslog'
log_min_error_statement = error
log_line_prefix = '%t %u %d %r '
log_statement = 'ddl'
log_timezone = 'UTC'
```

### 3. Roles y Permisos
```sql
-- Revocar permisos públicos por defecto
REVOKE ALL ON SCHEMA public FROM PUBLIC;
GRANT USAGE ON SCHEMA public TO app_ventas;

-- Crear roles por función
CREATE ROLE ventas_lectura;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO ventas_lectura;

CREATE ROLE ventas_escritura;
GRANT SELECT, INSERT, UPDATE ON facturas, facturas_detalle TO ventas_escritura;

-- Asignar roles a usuarios
GRANT ventas_lectura TO reportes;
GRANT ventas_escritura TO app_ventas;

-- Políticas de seguridad a nivel de fila (RLS)
ALTER TABLE clientes ENABLE ROW LEVEL SECURITY;
CREATE POLICY clientes_policy ON clientes
    USING (id_sucursal = current_setting('app.id_sucursal')::INT);
```

## SQL Server Hardening

### 1. Configuración
```sql
-- Habilitar solo autenticación Windows (si aplica)
EXEC xp_instance_regwrite N'HKEY_LOCAL_MACHINE',
    N'Software\Microsoft\MSSQLServer\MSSQLServer',
    N'LoginMode', REG_DWORD, 1;

-- Deshabilitar xp_cmdshell
EXEC sp_configure 'show advanced options', 0;
EXEC sp_configure 'xp_cmdshell', 0;
RECONFIGURE;

-- Deshabilitar procedimientos peligrosos
EXEC sp_configure 'Ole Automation Procedures', 0;
EXEC sp_configure 'Ad Hoc Distributed Queries', 0;
EXEC sp_configure 'Database Mail XPs', 0;
RECONFIGURE;
```

### 2. Auditoría
```sql
-- Crear auditoría
CREATE SERVER AUDIT SQL_Server_Audit
TO FILE (FILEPATH = 'C:\AuditLogs\')
WITH (ON_FAILURE = CONTINUE);

-- Iniciar auditoría
ALTER SERVER AUDIT SQL_Server_Audit WITH (STATE = ON);

-- Auditar cambios de esquema
CREATE DATABASE AUDIT SPECIFICATION SchemaChanges
FOR SERVER AUDIT SQL_Server_Audit
ADD (SCHEMA_OBJECT_CHANGE_GROUP);

-- Auditar DML en tablas sensibles
CREATE DATABASE AUDIT SPECIFICATION DataAccess
FOR SERVER AUDIT SQL_Server_Audit
ADD (SELECT, INSERT, UPDATE, DELETE ON OBJECT::ventas.clientes BY dbo);
```

## Oracle Hardening

```sql
-- Perfil de contraseñas
CREATE PROFILE ventas_profile AS
    FAILED_LOGIN_ATTEMPTS 3
    PASSWORD_LOCK_TIME 1
    PASSWORD_LIFE_TIME 90
    PASSWORD_REUSE_TIME 365
    PASSWORD_GRACE_TIME 3;

-- Asignar perfil
ALTER USER app_ventas PROFILE ventas_profile;

-- Mínimos privilegios
GRANT CREATE SESSION TO app_ventas;
GRANT SELECT, INSERT, UPDATE ON ventas.facturas TO app_ventas;
GRANT SELECT ON ventas.clientes TO app_ventas;

-- Virtual Private Database (VPD)
BEGIN
    DBMS_RLS.ADD_POLICY(
        object_schema => 'ventas',
        object_name => 'facturas',
        policy_name => 'sucursal_policy',
        function_schema => 'ventas',
        policy_function => 'fn_filtro_sucursal',
        statement_types => 'SELECT, INSERT, UPDATE, DELETE'
    );
END;
```

## Herramientas de Hardening

### CIS Benchmarks
```bash
# MySQL CIS Benchmark
# https://www.cisecurity.org/benchmark/mysql

# PostgreSQL CIS Benchmark
# https://www.cisecurity.org/benchmark/postgresql

# Herramienta automatizada
git clone https://github.com/scarpent/mysql-cis-benchmark
cd mysql-cis-benchmark
python mysql-cis.py --host localhost --user dba --password
```

### Lynis (Auditoría de Seguridad)
```bash
# Auditoría del sistema completo
lynis audit system

# Auditoría específica de base de datos
lynis --tests "DATABASES"
```

## Monitoreo de Seguridad

```sql
-- Intentos de conexión fallidos (MySQL)
SELECT * FROM mysql.general_log 
WHERE command_type = 'Connect' 
  AND status = 'Access denied'
ORDER BY event_time DESC;

-- Usuarios y conexiones activas
SELECT user, host, db, command, time, state 
FROM information_schema.PROCESSLIST;

-- Consultas sospechosas
SELECT * FROM mysql.slow_log 
WHERE sql_text REGEXP 'UNION|SLEEP|BENCHMARK|INFORMATION_SCHEMA|0x|--|#'
ORDER BY start_time DESC;

-- Cambios en permisos
SELECT * FROM mysql.general_log 
WHERE sql_text REGEXP 'GRANT|REVOKE|CREATE USER|DROP USER|SET PASSWORD';
```

---

## Anterior: [05 Ejemplos Practicos Ventas](../07-sql-injection/05-ejemplos-practicos-ventas.md)
## Siguiente: [02 Encriptacion](../08-seguridad/02-encriptacion.md)

[Volver al índice](../README.md)
