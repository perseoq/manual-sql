# 8.4 Backup y Recovery

## Introducción

Una estrategia de backup y recovery es crítica para cualquier sistema de ventas, compras e inventarios. La pérdida de datos puede significar la quiebra del negocio.

### RPO y RTO
```
RPO (Recovery Point Objective): Máxima cantidad de datos que se puede perder
    → Financiero: 0-15 minutos (backups cada 15 min)
    → PyME: 1-24 horas (backups diarios)
    
RTO (Recovery Time Objective): Tiempo máximo para recuperar el servicio
    → Financiero: 1-4 horas
    → PyME: 4-24 horas
```

## 1. Tipos de Backup

### Backup Físico (Raw)
```bash
# Copia directa de archivos de datos
# MySQL: /var/lib/mysql/
# PostgreSQL: /var/lib/postgresql/data/

# ❌ Requiere detener el servicio o usar LVM snapshot
# ✅ Rápido para restauración completa
```

### Backup Lógico (SQL)
```bash
# Exporta comandos SQL
# ✅ Portable entre versiones
# ✅ Se puede restaurar selectivamente
# ❌ Más lento que físico
```

### mysqldump
```bash
# Backup completo de una base de datos
mysqldump -u dba -p --databases sistema_ventas > ventas_backup.sql

# Backup comprimido
mysqldump -u dba -p --databases sistema_ventas | gzip > ventas_backup.sql.gz

# Backup de tablas específicas
mysqldump -u dba -p sistema_ventas facturas clientes > ventas_parcial.sql

# Backup sin datos (solo estructura)
mysqldump -u dba -p --no-data sistema_ventas > estructura.sql

# Backup con condiciones
mysqldump -u dba -p sistema_ventas facturas --where="fecha_emision >= '2024-01-01'" > facturas_2024.sql

# Backup de todas las bases de datos
mysqldump -u dba -p --all-databases --routines --triggers --events > full_backup.sql
```

### mysqldump - Opciones Avanzadas
```bash
# Backup rápido (menos consistencia, más velocidad)
mysqldump -u dba -p --opt sistema_ventas > ventas.sql

# Backup transaccional (consistente)
mysqldump -u dba -p --single-transaction --routines --triggers sistema_ventas > ventas.sql

# Backup con master position (para replicación)
mysqldump -u dba -p --master-data=2 --single-transaction sistema_ventas > ventas.sql

# Backup cifrado
mysqldump -u dba -p sistema_ventas | gpg --encrypt --recipient admin@empresa.com > ventas.sql.gpg

# Backup a servidor remoto
mysqldump -u dba -p sistema_ventas | ssh backup@10.0.1.100 "cat > /backups/ventas.sql"
```

### pg_dump (PostgreSQL)
```bash
# Backup de una base de datos
pg_dump -U dba -h localhost sistema_ventas > ventas_backup.sql

# Backup en formato custom (compresión, restauración paralela)
pg_dump -U dba -Fc sistema_ventas > ventas_backup.dump

# Backup comprimido
pg_dump -U dba sistema_ventas | gzip > ventas_backup.sql.gz

# Backup de todas las bases de datos
pg_dumpall -U dba -h localhost > full_backup.sql

# Backup de tablas específicas
pg_dump -U dba -t facturas -t clientes sistema_ventas > ventas_parcial.sql
```

### SQL Server (sqlcmd)
```bash
# Backup completo
sqlcmd -S localhost -U SA -Q "BACKUP DATABASE SistemaVentas TO DISK='C:\Backups\ventas.bak'"

# Backup con compresión
sqlcmd -S localhost -U SA -Q "BACKUP DATABASE SistemaVentas TO DISK='C:\Backups\ventas.bak' WITH COMPRESSION"

# Backup diferencial
sqlcmd -S localhost -U SA -Q "BACKUP DATABASE SistemaVentas TO DISK='C:\Backups\ventas_diff.bak' WITH DIFFERENTIAL"

# Backup de log de transacciones
sqlcmd -S localhost -U SA -Q "BACKUP LOG SistemaVentas TO DISK='C:\Backups\ventas_log.trn'"
```

## 2. Estrategia de Backup

### Regla 3-2-1
```
3 → 3 copias de los datos
2 → 2 medios de almacenamiento diferentes
1 → 1 copia fuera del sitio
```

### Plan de Backup Recomendado
```bash
# Backup completo: Domingo 02:00 AM
# Backup incremental: Lunes-Sábado 02:00 AM
# Binlog/Transaction log: Cada 15-60 minutos

# Automatizar con cron
# /etc/cron.d/backup_ventas
0 2 * * 0 root /scripts/full_backup.sh       # Domingo
0 2 * * 1-6 root /scripts/incremental.sh     # Lunes-Sábado
*/15 * * * * root /scripts/binlog_backup.sh  # Cada 15 min
```

### Script de Backup Automatizado
```bash
#!/bin/bash
# /scripts/full_backup.sh

set -e

DB_USER="dba"
DB_PASS="DBA_Admin2024!"
DB_NAME="sistema_ventas"
BACKUP_DIR="/backups/ventas"
RETENTION_DAYS=30
DATE=$(date +%Y%m%d_%H%M%S)

# Crear directorio
mkdir -p $BACKUP_DIR/{daily,weekly,monthly}

# Backup completo
echo "[$(date)] Iniciando backup de $DB_NAME..."

mysqldump -u $DB_USER -p$DB_PASS \
    --single-transaction \
    --routines \
    --triggers \
    --events \
    --databases $DB_NAME \
    | gzip > $BACKUP_DIR/daily/${DB_NAME}_${DATE}.sql.gz

# Backup de estructura separada
mysqldump -u $DB_USER -p$DB_PASS \
    --no-data \
    --routines \
    --triggers \
    $DB_NAME \
    | gzip > $BACKUP_DIR/daily/${DB_NAME}_estructura_${DATE}.sql.gz

# Respaldar archivos de configuración
tar -czf $BACKUP_DIR/daily/config_${DATE}.tar.gz /etc/mysql/

# Verificar integridad del backup
gunzip -c $BACKUP_DIR/daily/${DB_NAME}_${DATE}.sql.gz | head -n 50 > /dev/null 2>&1
if [ $? -eq 0 ]; then
    echo "[$(date)] Backup verificado correctamente"
    echo "${DB_NAME}_${DATE}.sql.gz|$(date)|SUCCESS|$(ls -lh $BACKUP_DIR/daily/${DB_NAME}_${DATE}.sql.gz | awk '{print $5}')" >> $BACKUP_DIR/backup_log.txt
else
    echo "[$(date)] ERROR: Backup corrupto" >> $BACKUP_DIR/backup_log.txt
    exit 1
fi

# Rotación: eliminar backups antiguos
find $BACKUP_DIR/daily/ -name "*.sql.gz" -mtime +$RETENTION_DAYS -delete
find $BACKUP_DIR/daily/ -name "*.tar.gz" -mtime +$RETENTION_DAYS -delete

# Copia semanal (domingos)
if [ $(date +%u) -eq 7 ]; then
    cp $BACKUP_DIR/daily/${DB_NAME}_${DATE}.sql.gz $BACKUP_DIR/weekly/
fi

# Copia mensual (día 1)
if [ $(date +%d) -eq 01 ]; then
    cp $BACKUP_DIR/daily/${DB_NAME}_${DATE}.sql.gz $BACKUP_DIR/monthly/
fi

echo "[$(date)] Backup completado exitosamente"
```

## 3. Restauración (Recovery)

### mysql (MySQL)
```bash
# Restaurar backup completo
mysql -u dba -p sistema_ventas < ventas_backup.sql

# Restaurar backup comprimido
gunzip -c ventas_backup.sql.gz | mysql -u dba -p sistema_ventas

# Restaurar tabla específica
mysql -u dba -p sistema_ventas -e "DROP TABLE facturas;"
sed -n '/CREATE TABLE.*facturas/,/CREATE TABLE/p' backup.sql | mysql -u dba -p sistema_ventas

# Restaurar usando source
mysql -u dba -p
mysql> source /backups/ventas_backup.sql
```

### Point-in-Time Recovery (MySQL)
```bash
# 1. Restaurar último backup completo
mysql -u dba -p sistema_ventas < full_backup.sql

# 2. Aplicar binary logs hasta el momento deseado
mysqlbinlog /var/log/mysql/mysql-bin.000023 \
    --stop-datetime="2024-01-15 14:30:00" \
    | mysql -u dba -p sistema_ventas

# 3. O recuperar hasta un punto específico
mysqlbinlog /var/log/mysql/mysql-bin.000023 \
    --stop-position=573849 \
    | mysql -u dba -p sistema_ventas

# 4. Omitir una transacción problemática
mysqlbinlog /var/log/mysql/mysql-bin.000023 \
    --stop-position=573849 \
    --start-position=573000 \
    --skip-gtid \
    | mysql -u dba -p sistema_ventas
```

### pg_restore (PostgreSQL)
```bash
# Restaurar backup SQL
psql -U dba -d sistema_ventas < ventas_backup.sql

# Restaurar backup custom (con opciones)
pg_restore -U dba -d sistema_ventas ventas_backup.dump

# Restaurar tabla específica
pg_restore -U dba -d sistema_ventas -t facturas ventas_backup.dump

# Restaurar con reemplazo de tablas existentes
pg_restore -U dba -d sistema_ventas --clean --if-exists ventas_backup.dump

# Restaurar en paralelo (más rápido)
pg_restore -U dba -d sistema_ventas -j 4 ventas_backup.dump
```

### SQL Server Recovery
```bash
# Restaurar backup completo
sqlcmd -S localhost -U SA -Q "RESTORE DATABASE SistemaVentas FROM DISK='C:\Backups\ventas.bak'"

# Restaurar con punto en el tiempo
sqlcmd -S localhost -U SA -Q "
RESTORE DATABASE SistemaVentas FROM DISK='C:\Backups\ventas.bak'
WITH NORECOVERY;
RESTORE LOG SistemaVentas FROM DISK='C:\Backups\ventas_log.trn'
WITH STOPAT='2024-01-15 14:30:00';
"

# Verificar integridad del backup
sqlcmd -S localhost -U SA -Q "RESTORE VERIFYONLY FROM DISK='C:\Backups\ventas.bak'"
```

## 4. Pruebas de Restauración

```bash
#!/bin/bash
# /scripts/test_restore.sh

# Probar restauración en servidor de pruebas
echo "=== Probando restauración ==="

# Restaurar en BD de prueba
mysql -u dba -p -e "CREATE DATABASE test_restore;"
gunzip -c /backups/ventas_20240101.sql.gz | mysql -u dba -p test_restore

# Verificar integridad
mysql -u dba -p -e "
USE test_restore;
SELECT 'Facturas:', COUNT(*) FROM facturas;
SELECT 'Clientes:', COUNT(*) FROM clientes;
SELECT 'Productos:', COUNT(*) FROM productos;
SELECT 'Total ventas:', SUM(total) FROM facturas WHERE estado = 'activa';
"

# Verificar consistencia
mysql -u dba -p -e "
USE test_restore;
SELECT COUNT(*) AS inconsistencias FROM facturas f
LEFT JOIN clientes c ON f.id_cliente = c.id_cliente
WHERE c.id_cliente IS NULL;
"

# Limpiar
mysql -u dba -p -e "DROP DATABASE test_restore;"
echo "=== Prueba completada ==="
```

## 5. Copias de Seguridad en la Nube

### AWS S3
```bash
# Subir backup a S3
aws s3 cp /backups/ventas_daily.sql.gz s3://ventas-backups/daily/

# Subir con lifecycle policy
aws s3 cp /backups/ventas_daily.sql.gz s3://ventas-backups/daily/ --storage-class STANDARD_IA

# Script automático con S3
#!/bin/bash
aws s3 sync /backups/ventas/ s3://ventas-backups/$(date +%Y/%m/%d)/
```

### Google Cloud Storage
```bash
# Subir backup
gsutil cp /backups/ventas_daily.sql.gz gs://ventas-backups/daily/

# Sincronizar directorio
gsutil rsync -r /backups/ventas/ gs://ventas-backups/daily/
```

### Azure Blob Storage
```bash
# Subir backup
az storage blob upload \
    --container-name ventas-backups \
    --file /backups/ventas_daily.sql.gz \
    --name daily/ventas_daily.sql.gz
```

## 6. Disaster Recovery Plan

### Plan de Acción
```yaml
Escenario: Pérdida total de base de datos
1. Detectar: Monitoreo alerta de caída
2. Evaluar: Determinar alcance del daño
3. Comunicar: Notificar a stakeholders
4. Activar DR: Iniciar procedimiento de recuperación
5. Restaurar: 
   a. Provisionar nuevo servidor (si es necesario)
   b. Restaurar último backup completo
   c. Aplicar binary logs (point-in-time recovery)
   d. Verificar integridad de datos
   e. Probar conectividad de aplicaciones
6. Validar: Confirmar que datos son correctos
7. Reanudar: Poner sistema en producción
8. Post-mortem: Investigar causa raíz
```

### Checklist de DRP
- [ ] Backups automáticos configurados y funcionando
- [ ] Backups fuera del sitio (cloud, otro datacenter)
- [ ] Procedimiento de restauración documentado y probado
- [ ] Tiempos RPO/RTO definidos y alcanzables
- [ ] Personal capacitado en procedimientos de recovery
- [ ] Pruebas de restauración trimestrales
- [ ] Monitoreo de integridad de backups
- [ ] Alertas de fallo de backup
- [ ] Cifrado de backups
- [ ] Acceso seguro a backups (solo personal autorizado)

---

## Anterior: [03 Auditoria](../08-seguridad/03-auditoria.md)
## Siguiente: [01 Analisis Exploratorio](../09-analisis-datos/01-analisis-exploratorio.md)

[Volver al índice](../README.md)
