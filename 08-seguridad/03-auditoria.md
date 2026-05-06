# 8.3 Auditoría

## Introducción

La auditoría de bases de datos registra quién hizo qué, cuándo y desde dónde. Es crucial para seguridad, cumplimiento normativo y forense.

## 1. Logs de Auditoría en MySQL

### General Query Log
```ini
[mysqld]
general_log = 1
general_log_file = /var/log/mysql/query.log
log_output = FILE
```

```sql
-- Ver logs de forma segura
SELECT * FROM mysql.general_log 
WHERE command_type != 'Sleep'
ORDER BY event_time DESC
LIMIT 100;
```

### Auditoría con Trigger (Cambios en Tablas Sensibles)
```sql
-- Tabla de auditoría
CREATE TABLE audit_log (
    id_audit INT PRIMARY KEY AUTO_INCREMENT,
    tabla VARCHAR(50) NOT NULL,
    id_registro INT NOT NULL,
    operacion ENUM('INSERT', 'UPDATE', 'DELETE') NOT NULL,
    datos_anteriores JSON,
    datos_nuevos JSON,
    usuario VARCHAR(100),
    ip_address VARCHAR(45),
    user_agent VARCHAR(500),
    fecha_cambio DATETIME DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_audit_tabla (tabla),
    INDEX idx_audit_fecha (fecha_cambio),
    INDEX idx_audit_usuario (usuario)
);

-- Trigger de auditoría para facturas
DELIMITER //
CREATE TRIGGER trg_audit_facturas
AFTER UPDATE ON facturas
FOR EACH ROW
BEGIN
    INSERT INTO audit_log (tabla, id_registro, operacion, 
        datos_anteriores, datos_nuevos, usuario, ip_address)
    VALUES (
        'facturas', NEW.id_factura, 'UPDATE',
        JSON_OBJECT('estado', OLD.estado, 'total', OLD.total, 'metodo_pago', OLD.metodo_pago),
        JSON_OBJECT('estado', NEW.estado, 'total', NEW.total, 'metodo_pago', NEW.metodo_pago),
        CURRENT_USER(), SUBSTRING_INDEX(USER(), '@', -1)
    );
END //

DELIMITER ;

-- Consultar auditoría
SELECT 
    fecha_cambio,
    tabla,
    id_registro,
    operacion,
    datos_anteriores,
    datos_nuevos,
    usuario,
    ip_address
FROM audit_log
WHERE tabla = 'facturas' 
  AND fecha_cambio >= DATE_SUB(NOW(), INTERVAL 7 DAY)
ORDER BY fecha_cambio DESC;
```

### MySQL Enterprise Audit
```sql
-- Instalar plugin de auditoría (MySQL Enterprise)
INSTALL PLUGIN audit_log SONAME 'audit_log.so';

-- Configurar
SET GLOBAL audit_log_policy = 'ALL';
SET GLOBAL audit_log_strategy = 'ASYNCHRONOUS';
SET GLOBAL audit_log_format = 'JSON';
SET GLOBAL audit_log_file = '/var/log/mysql/audit.log';

-- Filtrar por usuario
SET GLOBAL audit_log_include_users = 'app_ventas,reportes';

-- Ver eventos de auditoría
SELECT * FROM mysql.audit_log 
WHERE utc_timestamp >= NOW() - INTERVAL 1 HOUR;
```

## 2. PostgreSQL Auditoría

### pgAudit (Extensión)
```bash
# Instalar pgAudit
sudo apt install postgresql-16-pgaudit
```

```ini
# postgresql.conf
shared_preload_libraries = 'pgaudit'
pgaudit.log = 'read,write,ddl,misc'
pgaudit.log_level = 'notice'
pgaudit.log_catalog = off
pgaudit.log_relation = on
pgaudit.log_statement_once = off
```

```sql
-- Crear extensión
CREATE EXTENSION IF NOT EXISTS pgaudit;

-- Auditoría específica por objeto
SELECT pgaudit.audit_table('facturas', 'read,write');
SELECT pgaudit.audit_table('clientes', 'read,write,ddl');

-- Ver logs de auditoría
SELECT * FROM pg_stat_activity WHERE query LIKE '%DELETE FROM facturas%';
```

### hstore para Auditoría de Cambios
```sql
CREATE EXTENSION IF NOT EXISTS hstore;

CREATE OR REPLACE FUNCTION fn_audit_cambios()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'UPDATE' THEN
        INSERT INTO audit_log (tabla, id_registro, operacion, 
            datos_anteriores, datos_nuevos, usuario)
        VALUES (
            TG_TABLE_NAME, NEW.id, 'UPDATE',
            hstore_to_json(hstore(OLD)),
            hstore_to_json(hstore(NEW)),
            current_user
        );
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_audit_productos
AFTER UPDATE ON productos
FOR EACH ROW EXECUTE FUNCTION fn_audit_cambios();
```

## 3. SQL Server Auditoría

### Server Audit
```sql
-- Crear auditoría a nivel de servidor
CREATE SERVER AUDIT VentasAudit
TO FILE (FILEPATH = 'C:\AuditLogs\', MAXSIZE = 256 MB, MAX_FILES = 10)
WITH (ON_FAILURE = CONTINUE, QUEUE_DELAY = 1000);

ALTER SERVER AUDIT VentasAudit WITH (STATE = ON);

-- Auditar SELECT en tabla sensibles
CREATE DATABASE AUDIT SPECIFICATION SelectAudit
FOR SERVER AUDIT VentasAudit
ADD (SELECT ON OBJECT::ventas.clientes BY public)
WITH (STATE = ON);

-- Auditar cambios DDL
CREATE DATABASE AUDIT SPECIFICATION DDL_Audit
FOR SERVER AUDIT VentasAudit
ADD (SCHEMA_OBJECT_CHANGE_GROUP)
WITH (STATE = ON);

-- Consultar logs de auditoría
SELECT * FROM sys.fn_get_audit_file('C:\AuditLogs\*', DEFAULT, DEFAULT);
```

### Change Data Capture (CDC)
```sql
-- Habilitar CDC a nivel de base de datos
EXEC sys.sp_cdc_enable_db;

-- Habilitar CDC en tabla
EXEC sys.sp_cdc_enable_table
    @source_schema = 'dbo',
    @source_name = 'facturas',
    @role_name = 'cdc_reader';

-- Consultar cambios
DECLARE @from_lsn binary(10), @to_lsn binary(10);
SET @from_lsn = sys.fn_cdc_get_min_lsn('dbo_facturas');
SET @to_lsn = sys.fn_cdc_get_max_lsn();

SELECT * FROM cdc.fn_cdc_get_all_changes_dbo_facturas(
    @from_lsn, @to_lsn, 'all');
```

## 4. Forense Digital

### Investigación de Incidentes
```sql
-- 1. Identificar cambios sospechosos en la última hora
SELECT * FROM audit_log
WHERE fecha_cambio >= DATE_SUB(NOW(), INTERVAL 1 HOUR)
  AND (datos_nuevos->>'$.total' < 1 
       OR datos_nuevos->>'$.estado' = 'cancelada')
ORDER BY fecha_cambio DESC;

-- 2. Buscar actividad de un usuario específico
SELECT * FROM audit_log
WHERE usuario LIKE '%hacker%'
   OR usuario LIKE '%unknown%'
ORDER BY fecha_cambio DESC;

-- 3. Identificar conexiones desde IPs sospechosas
SELECT * FROM mysql.general_log
WHERE command_type = 'Connect'
  AND ip NOT IN ('127.0.0.1', '10.0.1.%')
ORDER BY event_time DESC;

-- 4. Buscar datos eliminados
SELECT * FROM audit_log
WHERE operacion = 'DELETE'
  AND fecha_cambio >= DATE_SUB(NOW(), INTERVAL 7 DAY);
```

### Recuperación de Datos
```sql
-- Si se eliminaron registros de facturas:
SELECT datos_anteriores->>'$.total' AS total_anterior,
       datos_anteriores->>'$.estado' AS estado_anterior,
       fecha_cambio
FROM audit_log
WHERE tabla = 'facturas' 
  AND id_registro = 100
  AND operacion = 'UPDATE'
ORDER BY fecha_cambio DESC
LIMIT 1;
```

## 5. Monitoreo en Tiempo Real

### Alertas SQL
```sql
-- Crear evento para alertar de actividad sospechosa
DELIMITER //
CREATE EVENT ev_alertar_actividad_sospechosa
ON SCHEDULE EVERY 5 MINUTE
DO
BEGIN
    -- Detectar múltiples fallos de login
    IF EXISTS (
        SELECT 1 FROM mysql.general_log
        WHERE command_type = 'Connect' 
          AND status = 'Access denied'
          AND event_time >= DATE_SUB(NOW(), INTERVAL 5 MINUTE)
        GROUP BY ip
        HAVING COUNT(*) > 10
    ) THEN
        INSERT INTO alertas_seguridad (tipo, descripcion, severidad)
        SELECT 'FUERZA_BRUTA', 
               CONCAT('Múltiples fallos de login desde: ', ip),
               'ALTA'
        FROM mysql.general_log
        WHERE command_type = 'Connect' 
          AND status = 'Access denied'
          AND event_time >= DATE_SUB(NOW(), INTERVAL 5 MINUTE)
        GROUP BY ip
        HAVING COUNT(*) > 10;
    END IF;
    
    -- Detectar consultas masivas
    IF EXISTS (
        SELECT 1 FROM mysql.general_log
        WHERE sql_text REGEXP 'SELECT.*FROM.*(clientes|tarjetas)'
          AND event_time >= DATE_SUB(NOW(), INTERVAL 5 MINUTE)
        GROUP BY thread_id
        HAVING COUNT(*) > 100
    ) THEN
        INSERT INTO alertas_seguridad (tipo, descripcion, severidad)
        VALUES ('EXTRACCION_DATOS', 
                'Posible extracción masiva de datos', 'CRITICA');
    END IF;
END //
DELIMITER ;
```

## 6. Reportes de Auditoría

### Reporte de Cumplimiento
```sql
-- Actividad por usuario (últimos 30 días)
SELECT 
    usuario,
    COUNT(*) AS total_operaciones,
    SUM(CASE WHEN operacion = 'INSERT' THEN 1 ELSE 0 END) AS inserts,
    SUM(CASE WHEN operacion = 'UPDATE' THEN 1 ELSE 0 END) AS updates,
    SUM(CASE WHEN operacion = 'DELETE' THEN 1 ELSE 0 END) AS deletes,
    MIN(fecha_cambio) AS primera_actividad,
    MAX(fecha_cambio) AS ultima_actividad
FROM audit_log
WHERE fecha_cambio >= DATE_SUB(NOW(), INTERVAL 30 DAY)
GROUP BY usuario
ORDER BY total_operaciones DESC;

-- Cambios en tablas críticas
SELECT 
    tabla,
    COUNT(*) AS cambios,
    COUNT(DISTINCT usuario) AS usuarios_diferentes
FROM audit_log
WHERE tabla IN ('facturas', 'clientes', 'productos_stock', 'precios')
  AND fecha_cambio >= DATE_SUB(NOW(), INTERVAL 7 DAY)
GROUP BY tabla
ORDER BY cambios DESC;
```

## Herramientas de Auditoría

```bash
# Open Source
# - pgAudit (PostgreSQL)
# - MySQL Enterprise Audit
# - Auditd (Linux)
# - OSSEC (HIDS)
# - Wazuh (SIEM)

# Comerciales
# - Imperva SecureSphere
# - Guardium (IBM)
# - SolarWinds Database Performance Analyzer
# - Splunk (SIEM)
```

---

## Anterior: [02 Encriptacion](../08-seguridad/02-encriptacion.md)
## Siguiente: [04 Backup Y Recovery](../08-seguridad/04-backup-y-recovery.md)

[Volver al índice](../README.md)
