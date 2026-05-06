# 8.2 Encriptación

## Introducción

La encriptación protege los datos transformándolos en un formato ilegible sin la clave adecuada.

### Tipos de Encriptación
```
En Tránsito:    Datos moviéndose entre cliente y servidor (SSL/TLS)
En Reposo:      Datos almacenados en disco (TDE, cifrado de tablas)
En Uso:         Datos en memoria (poco común en BD)
A nivel de Aplicación:  Datos cifrados antes de enviarlos a la BD
```

## 1. Encriptación en Tránsito (SSL/TLS)

### MySQL
```sql
-- Verificar si SSL está habilitado
SHOW VARIABLES LIKE '%ssl%';
SHOW STATUS LIKE 'ssl%';

-- Crear certificados (OpenSSL)
-- server-cert.pem, server-key.pem, ca.pem

-- Configurar MySQL
[mysqld]
ssl-ca = /etc/mysql/ssl/ca.pem
ssl-cert = /etc/mysql/ssl/server-cert.pem
ssl-key = /etc/mysql/ssl/server-key.pem
require_secure_transport = ON

-- Usuario que requiere SSL
ALTER USER 'app_ventas'@'%' REQUIRE SSL;
ALTER USER 'reportes'@'%' REQUIRE X509;  -- Requiere certificado

-- Verificar conexiones SSL
SELECT * FROM information_schema.PROCESSLIST WHERE ssl_cipher IS NOT NULL;
```

### PostgreSQL
```ini
# postgresql.conf
ssl = on
ssl_cert_file = 'server.crt'
ssl_key_file = 'server.key'
ssl_ca_file = 'root.crt'
ssl_crl_file = 'root.crl'
```

### Cliente MySQL con SSL
```bash
mysql --ssl-ca=ca.pem --ssl-cert=client-cert.pem --ssl-key=client-key.pem \
      -h server.com -u app_ventas -p
```

## 2. Transparent Data Encryption (TDE)

Cifra automáticamente los datos en disco sin cambios en la aplicación.

### MySQL (tablaspaces cifrados)
```sql
-- Verificar soporte
SHOW VARIABLES LIKE 'have_encryption';

-- Configurar clave de cifrado
[mysqld]
early-plugin-load = keyring_file.so
keyring_file_data = /var/lib/mysql-keyring/keyring

-- Crear tablespace cifrado
CREATE TABLESPACE ventas_ts
ADD DATAFILE 'ventas_ts.ibd'
ENCRYPTION = 'Y';

-- Crear tabla en tablespace cifrado
CREATE TABLE facturas_cifrada (
    id_factura INT,
    folio VARCHAR(30),
    total DECIMAL(12,2)
) TABLESPACE = ventas_ts ENCRYPTION = 'Y';
```

### SQL Server (TDE)
```sql
-- Crear clave maestra
CREATE MASTER KEY ENCRYPTION BY PASSWORD = 'Str0ng!P@ssw0rd';

-- Crear certificado
CREATE CERTIFICATE TDE_Cert WITH SUBJECT = 'TDE Certificate';

-- Crear clave de encriptación de base de datos
USE Ventas;
CREATE DATABASE ENCRYPTION KEY
WITH ALGORITHM = AES_256
ENCRYPTION BY SERVER CERTIFICATE TDE_Cert;

-- Habilitar TDE
ALTER DATABASE Ventas SET ENCRYPTION ON;

-- Verificar estado
SELECT DB_NAME(database_id), encryption_state,
    CASE encryption_state
        WHEN 0 THEN 'No encrypted'
        WHEN 1 THEN 'Unencrypted'
        WHEN 2 THEN 'Encryption in progress'
        WHEN 3 THEN 'Encrypted'
        WHEN 4 THEN 'Key change in progress'
        WHEN 5 THEN 'Decryption in progress'
    END AS estado
FROM sys.dm_database_encryption_keys;
```

## 3. Cifrado a Nivel de Columna

Cifra campos específicos como tarjetas de crédito, RFC, etc.

### MySQL - AES_ENCRYPT / AES_DECRYPT
```sql
-- Crear tabla con campo cifrado
CREATE TABLE tarjetas_credito (
    id_tarjeta INT PRIMARY KEY AUTO_INCREMENT,
    id_cliente INT NOT NULL,
    numero_tarjeta VARBINARY(256) NOT NULL,  -- Cifrado
    cvv VARBINARY(256) NOT NULL,              -- Cifrado
    titular VARCHAR(100),
    fecha_expiracion DATE,
    ultimos_4_digitos VARCHAR(4),             -- Solo últimos 4 visibles
    FOREIGN KEY (id_cliente) REFERENCES clientes(id_cliente)
);

-- Insertar datos cifrados
INSERT INTO tarjetas_credito (id_cliente, numero_tarjeta, cvv, titular, ultimos_4_digitos)
VALUES (
    1,
    AES_ENCRYPT('4111111111111111', 'clave_secreta_256_bits'),
    AES_ENCRYPT('123', 'clave_secreta_256_bits'),
    'María García',
    RIGHT('4111111111111111', 4)
);

-- Leer datos cifrados
SELECT 
    id_tarjeta,
    AES_DECRYPT(numero_tarjeta, 'clave_secreta_256_bits') AS numero_completo,
    ultimos_4_digitos
FROM tarjetas_credito
WHERE id_cliente = 1;
```

### PostgreSQL - pgcrypto
```sql
-- Habilitar extensión
CREATE EXTENSION IF NOT EXISTS pgcrypto;

-- Crear tabla
CREATE TABLE datos_sensibles (
    id SERIAL PRIMARY KEY,
    id_cliente INT,
    rfc_cifrado BYTEA,
    email_cifrado BYTEA
);

-- Insertar con cifrado
INSERT INTO datos_sensibles (id_cliente, rfc_cifrado, email_cifrado)
VALUES (
    1,
    pgp_sym_encrypt('GALM890101ABC', 'clave_secreta'),
    pgp_sym_encrypt('maria@email.com', 'clave_secreta')
);

-- Leer descifrando
SELECT 
    id_cliente,
    pgp_sym_decrypt(rfc_cifrado, 'clave_secreta') AS rfc,
    pgp_sym_decrypt(email_cifrado, 'clave_secreta') AS email
FROM datos_sensibles;
```

## 4. Hashing de Contraseñas

### MySQL
```sql
-- Usar funciones hash para contraseñas
CREATE TABLE usuarios (
    id_usuario INT PRIMARY KEY AUTO_INCREMENT,
    usuario VARCHAR(50) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    salt VARCHAR(255)
);

-- Insertar con hash
INSERT INTO usuarios (usuario, password_hash, salt)
VALUES (
    'admin',
    SHA2(CONCAT('MiPassword123', 'salt_aleatorio'), 256),
    'salt_aleatorio'
);

-- Verificar login
SELECT * FROM usuarios 
WHERE usuario = 'admin' 
  AND password_hash = SHA2(CONCAT('MiPassword123', salt), 256);
```

### PostgreSQL
```sql
-- Usar pgcrypto para hash seguro
CREATE EXTENSION pgcrypto;

INSERT INTO usuarios (usuario, password_hash)
VALUES (
    'admin',
    crypt('MiPassword123', gen_salt('bf', 10))  -- bcrypt con factor 10
);

-- Verificar
SELECT * FROM usuarios 
WHERE usuario = 'admin' 
  AND password_hash = crypt('MiPassword123', password_hash);
```

## 5. Encriptación a Nivel de Aplicación

```python
from cryptography.fernet import Fernet
import base64

# Generar clave (una vez, guardar de forma segura)
clave = Fernet.generate_key()
cipher = Fernet(clave)

# Cifrar antes de guardar
rfc_cifrado = cipher.encrypt(b"GALM890101ABC")
# Guardar rfc_cifrado en la BD

# Descifrar al leer
rfc = cipher.decrypt(rfc_cifrado)
print(rfc.decode())
```

```javascript
// Node.js con crypto
const crypto = require('crypto');

const algorithm = 'aes-256-gcm';
const key = crypto.randomBytes(32);
const iv = crypto.randomBytes(16);

function encrypt(text) {
    const cipher = crypto.createCipheriv(algorithm, key, iv);
    let encrypted = cipher.update(text, 'utf8', 'hex');
    encrypted += cipher.final('hex');
    return { encrypted, iv: iv.toString('hex'), tag: cipher.getAuthTag().toString('hex') };
}

function decrypt(encrypted, iv, tag) {
    const decipher = crypto.createDecipheriv(algorithm, key, Buffer.from(iv, 'hex'));
    decipher.setAuthTag(Buffer.from(tag, 'hex'));
    let decrypted = decipher.update(encrypted, 'hex', 'utf8');
    decrypted += decipher.final('utf8');
    return decrypted;
}
```

## 6. Gestión de Claves

### Almacenamiento Seguro de Claves
```bash
# ❌ MAL: Claves en el código
password = "MiPassword123"

# ❌ MAL: Claves en archivos de configuración sin proteger
config.ini: password = MiPassword123

# ✅ BIEN: Variables de entorno
export DB_PASSWORD="MiPassword123"
# En el código:
password = os.environ.get('DB_PASSWORD')

# ✅ BIEN: HashiCorp Vault
vault write database/creds/ventas-role

# ✅ BIEN: AWS Secrets Manager
aws secretsmanager get-secret-value --secret-id ventas/db/password
```

### Rotación de Claves
```sql
-- Re-cifrar datos con nueva clave
UPDATE tarjetas_credito 
SET numero_tarjeta = AES_ENCRYPT(
    AES_DECRYPT(numero_tarjeta, 'clave_vieja'),
    'clave_nueva'
)
WHERE id_tarjeta > 0;
```

## Mejores Prácticas

### Cumplimiento Normativo
```
PCI-DSS:    Tarjetas de crédito cifradas, no almacenar CVV
GDPR:       Datos personales cifrados o seudonimizados
LFPDPPP:    Datos sensibles cifrados en México
SOX:        Datos financieros protegidos
HIPAA:      Datos médicos cifrados
```

### Checklist de Encriptación
- [ ] SSL/TLS en todas las conexiones
- [ ] Certificados válidos y renovados
- [ ] TDE en tablas con datos sensibles
- [ ] Columnas sensibles cifradas individualmente
- [ ] Contraseñas hasheadas (bcrypt, SHA-256+)
- [ ] Claves almacenadas de forma segura (Vault, variables de entorno)
- [ ] Rotación periódica de claves
- [ ] Backups también cifrados
- [ ] Acceso a claves auditado y restringido
- [ ] Pruebas de recuperación con cifrado

---

## Anterior: [01 Hardening Bases Datos](../08-seguridad/01-hardening-bases-datos.md)
## Siguiente: [03 Auditoria](../08-seguridad/03-auditoria.md)

[Volver al índice](../README.md)
