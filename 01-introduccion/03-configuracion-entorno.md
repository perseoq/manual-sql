# 1.3 Configuración del Entorno

Antes de poder escribir una sola línea de SQL, necesitas tener un **motor de base de datos** instalado en tu computadora. Piensa en esto como preparar tu cocina antes de cocinar: necesitas la estufa (el motor de base de datos), los utensilios (las herramientas de administración) y los ingredientes (tus datos) listos para usar.

En este capítulo te guiaré paso a paso para instalar y configurar los motores de base de datos más populares: MySQL, PostgreSQL, SQL Server, y también veremos cómo usar Docker para tenerlos todos sin ensuciar tu computadora. No te preocupes si no entiendes todos los comandos al principio; te explicaré cada línea en detalle.

---

## Instalación de Motores de Base de Datos

Cada motor de base de datos se instala de manera diferente, pero todos siguen el mismo patrón general: descargar el software, instalarlo en tu sistema, iniciar el servicio, y configurarlo para que sea seguro y funcional.

---

### 1. MySQL / MariaDB

MySQL es el motor de base de datos más popular del mundo, especialmente en aplicaciones web. MariaDB es un derivado ("fork") creado por los desarrolladores originales de MySQL, compatible al 100% pero con mejoras adicionales. Para efectos prácticos, todo lo que aprendas con MySQL funciona igual en MariaDB.

#### Instalación en Ubuntu/Debian

Primero veamos cómo instalar MySQL en Ubuntu o Debian (los sistemas Linux más comunes). Si usas Windows o macOS, no te preocupes —más abajo cubrimos esos casos.

Vamos comando por comando para que entiendas qué hace cada uno:

- **`sudo apt update`**: Este comando actualiza la lista de paquetes disponibles en los repositorios de Ubuntu. Piensa en ello como actualizar el catálogo de una tienda antes de ir a comprar: necesitas saber qué productos (versiones de software) están disponibles. `sudo` significa "superuser do" y te da permisos de administrador para ejecutar el comando (te pedirá tu contraseña). `apt` es el gestor de paquetes de Ubuntu.

- **`sudo apt install mysql-server -y`**: Le dice a Ubuntu "instala el paquete mysql-server". La bandera `-y` responde automáticamente "sí" a cualquier pregunta de confirmación. Sin esa bandera, el instalador te preguntaría "¿Estás seguro de que quieres instalar?" y tendrías que escribir "y" manualmente.

- **`mysql --version`**: Verifica que MySQL se haya instalado correctamente mostrando la versión instalada. Si ves algo como `mysql  Ver 8.0.x`, todo está bien.

- **`sudo systemctl start mysql`**: Inicia el servicio de MySQL. `systemctl` es el sistema de control de servicios de Ubuntu. `start` le dice "enciende MySQL". Sin este paso, MySQL estaría instalado pero apagado.

- **`sudo systemctl enable mysql`**: Configura MySQL para que se inicie automáticamente cada vez que enciendas tu computadora. Sin esto, tendrías que iniciar MySQL manualmente después de cada reinicio.

- **`sudo mysql_secure_installation`**: Este es un script de seguridad que te guía para hacer tu instalación más segura: te permite establecer una contraseña para el usuario root (el administrador), eliminar usuarios anónimos, deshabilitar el acceso root remoto, y eliminar bases de datos de prueba.

```bash
# Actualizar repositorios
sudo apt update

# Instalar MySQL Server
sudo apt install mysql-server -y

# Verificar instalación
mysql --version

# Iniciar servicio
sudo systemctl start mysql
sudo systemctl enable mysql

# Ejecutar script de seguridad
sudo mysql_secure_installation
```

#### Instalación en Windows

Si usas Windows, el proceso es diferente porque Windows no tiene un gestor de paquetes como `apt`. En su lugar, usas un instalador gráfico tradicional:

1. **Descargar MySQL Installer**: Ve a la página oficial de MySQL y descarga el instalador. Es un archivo `.msi` (instalador de Windows) que contiene todo lo necesario.
2. **Seleccionar "Developer Default"**: Durante la instalación, te preguntará qué componentes instalar. "Developer Default" instala MySQL Server, las herramientas de línea de comandos, los conectores para distintos lenguajes de programación, y MySQL Workbench (una herramienta gráfica).
3. **Configurar contraseña para root**: El usuario `root` es el administrador de la base de datos. Como es el "dueño" del sistema, debes protegerlo con una contraseña segura. No uses "admin123" ni "password" —usa algo como "V3ntas_2024!".
4. **Iniciar MySQL desde Services**: En Windows, los servicios se administran desde el "Administrador de servicios". MySQL se registra como un servicio de Windows y puedes iniciarlo, detenerlo o configurarlo para que arranque automáticamente desde allí.

```
1. Descargar MySQL Installer desde https://dev.mysql.com/downloads/installer/
2. Ejecutar el instalador y seleccionar "Developer Default"
3. Configurar contraseña para root
4. Iniciar MySQL desde Services
```

#### Configuración Inicial de MySQL

Una vez que MySQL está instalado, necesitas crear tu base de datos y tus usuarios. Es como comprar un edificio de oficinas: ahora tienes que ponerle nombre al edificio (la base de datos), crear los empleados (usuarios) y darles las llaves de las puertas que les corresponden (permisos).

Vamos a analizar cada comando SQL de esta sección:

- **`mysql -u root -p`**: Este comando se ejecuta en la terminal (no dentro de MySQL) y te conecta al servidor MySQL como usuario root. `-u` especifica el usuario, `-p` le dice a MySQL que te pida la contraseña. Después de escribir esto, presiona Enter y te pedirá la contraseña que configuraste durante la instalación.

- **`CREATE DATABASE sistema_ventas`**: Crea una nueva base de datos llamada `sistema_ventas`. Una base de datos es como una carpeta que contendrá todas las tablas de tu proyecto.
  - **`CHARACTER SET utf8mb4`**: Define la codificación de caracteres. `utf8mb4` es el estándar moderno que soporta cualquier carácter: letras con acento (é, á, í, ó, ú), la ñ, emojis 😊, símbolos de otros idiomas (chino, árabe, etc.). Si no especificas esto, podrías tener problemas guardando nombres como "María José" o "México".
  - **`COLLATE utf8mb4_unicode_ci`**: Define cómo se ordenan y comparan los textos. `utf8mb4_unicode_ci` significa que 'a' y 'A' se consideran iguales (ci = case insensitive), y que 'ñ' se ordena después de 'n' pero antes de 'o' (como en español).

- **`CREATE USER 'app_ventas'@'localhost' IDENTIFIED BY 'ContraseñaSegura123!'`**: Crea un usuario de aplicación. Fíjate en las partes:
  - `'app_ventas'`: Es el nombre del usuario.
  - `@'localhost'`: Significa que este usuario solo puede conectarse desde la misma computadora donde está instalado MySQL.
  - `@'%'`: El símbolo `%` es un comodín que significa "cualquier dirección". Este segundo usuario permite conexiones desde cualquier computadora en la red.
  - `IDENTIFIED BY 'ContraseñaSegura123!'`: Establece la contraseña.

- **`GRANT SELECT, INSERT, UPDATE, DELETE ON sistema_ventas.* TO 'app_ventas'@'localhost'`**: Otorga permisos específicos. `SELECT` para leer, `INSERT` para agregar, `UPDATE` para modificar, `DELETE` para eliminar. `sistema_ventas.*` significa "todas las tablas dentro de la base de datos sistema_ventas". El punto y asterisco es como decir "todo lo que está dentro de esta carpeta".

- **`GRANT ALL PRIVILEGES ON sistema_ventas.* TO 'dba'@'localhost'`**: El usuario `dba` (database administrator) obtiene TODOS los privilegios sobre la base de datos, incluyendo crear tablas, eliminar tablas, modificar estructuras, etc.

- **`GRANT SUPER, PROCESS, SHOW DATABASES ON *.* TO 'dba'@'localhost'`**: Además, el DBA obtiene permisos globales (sobre todas las bases de datos `*.*`): `SUPER` para tareas administrativas, `PROCESS` para ver qué consultas están ejecutándose, `SHOW DATABASES` para ver todas las bases de datos del servidor.

- **`FLUSH PRIVILEGES`**: Recarga la configuración de permisos. Es como decirle a MySQL "terminé de dar permisos, aplícalos ahora". Sin este comando, los cambios podrían no estar disponibles inmediatamente.

```sql
-- Acceder a MySQL como root
mysql -u root -p

-- Crear base de datos para el sistema de ventas
CREATE DATABASE sistema_ventas 
    CHARACTER SET utf8mb4 
    COLLATE utf8mb4_unicode_ci;

-- Crear usuario para la aplicación (NUNCA usar root)
CREATE USER 'app_ventas'@'localhost' IDENTIFIED BY 'ContraseñaSegura123!';
CREATE USER 'app_ventas'@'%' IDENTIFIED BY 'ContraseñaSegura123!';

-- Otorgar permisos específicos
GRANT SELECT, INSERT, UPDATE, DELETE ON sistema_ventas.* TO 'app_ventas'@'localhost';
GRANT SELECT, INSERT, UPDATE, DELETE ON sistema_ventas.* TO 'app_ventas'@'%';

-- Crear usuario de solo lectura para reportes
CREATE USER 'reportes'@'%' IDENTIFIED BY 'Reportes2024!';
GRANT SELECT ON sistema_ventas.* TO 'reportes'@'%';

-- Crear usuario DBA
CREATE USER 'dba'@'localhost' IDENTIFIED BY 'DBA_Admin2024!';
GRANT ALL PRIVILEGES ON sistema_ventas.* TO 'dba'@'localhost';
GRANT SUPER, PROCESS, SHOW DATABASES ON *.* TO 'dba'@'localhost';

FLUSH PRIVILEGES;
```

#### Configuración de MySQL para Ventas (my.cnf)

El archivo `my.cnf` es el **archivo de configuración principal de MySQL**. Es como el panel de control de un auto: aquí ajustas cómo se comporta el motor, cuánta memoria puede usar, cómo se conecta, etc.

Vamos a entender cada parámetro:

**Características generales:**
- **`port = 3306`**: El puerto en el que MySQL escucha conexiones. El puerto 3306 es el estándar para MySQL. Piensa en el puerto como la "puerta" por la que las aplicaciones se conectan a la base de datos.
- **`bind-address = 0.0.0.0`**: Le dice a MySQL que acepte conexiones desde cualquier dirección IP. `0.0.0.0` significa "todas las interfaces de red". Si usas `127.0.0.1`, solo aceptaría conexiones locales.
- **`max_connections = 500`**: El número máximo de conexiones simultáneas que MySQL aceptará. Si tienes 500 usuarios intentando conectarse al mismo tiempo, el número 501 recibirá un error.

**Almacenamiento:**
- **`default-storage-engine = InnoDB`**: Define el motor de almacenamiento predeterminado. InnoDB es el motor que soporta transacciones (ACID), claves foráneas y es confiable para sistemas de producción. La alternativa, MyISAM, es más antigua y no soporta transacciones.
- **`innodb_buffer_pool_size = 4G`**: Este es probablemente el parámetro más importante para el rendimiento. Define cuánta memoria RAM usará InnoDB para almacenar en caché los datos y los índices. La recomendación es 70-80% de la RAM disponible en un servidor dedicado a MySQL. Si tu servidor tiene 8 GB de RAM, asigna unos 6 GB aquí.
- **`innodb_log_file_size = 512M`**: Tamaño del archivo de registro (log) de InnoDB, donde se guardan las transacciones antes de escribirse definitivamente en la base de datos. Un valor más grande mejora el rendimiento en sistemas con muchas escrituras.
- **`innodb_flush_log_at_trx_commit = 2`**: Controla cómo y cuándo se guardan los registros de transacciones en el disco. El valor `2` es un equilibrio entre rendimiento y seguridad: es más rápido que el valor `1` (que es el más seguro), pero podría perder hasta 1 segundo de datos en caso de un corte de energía.

**Seguridad:**
- **`skip_show_database`**: Cuando está activado, los usuarios solo pueden ver las bases de datos sobre las que tienen permisos. Es una medida de seguridad para que un usuario no pueda husmear en bases de datos que no le corresponden.
- **`local-infile = 0`**: Deshabilita la capacidad de cargar archivos locales con `LOAD DATA LOCAL INFILE`. Esto previene un ataque de seguridad conocido donde un atacante podría leer archivos del servidor.
- **`sql_mode = STRICT_TRANS_TABLES,NO_ENGINE_SUBSTITUTION`**: Define el modo SQL estricto. `STRICT_TRANS_TABLES` rechaza operaciones que intenten guardar valores inválidos (por ejemplo, insertar un texto en una columna numérica). `NO_ENGINE_SUBSTITUTION` evita que MySQL cambie silenciosamente el motor de almacenamiento si el especificado no está disponible.

**Logging (registros):**
- **`log_error = /var/log/mysql/error.log`**: Dónde se guardan los errores de MySQL. Cuando algo salga mal, aquí es donde debes mirar primero.
- **`slow_query_log = 1`**: Activa el registro de consultas lentas. Una "consulta lenta" es aquella que tarda más de cierto tiempo en ejecutarse.
- **`slow_query_log_file = /var/log/mysql/slow.log`**: Dónde se guardan esas consultas lentas.
- **`long_query_time = 2`**: Define qué se considera "lento": cualquier consulta que tarde más de 2 segundos.

```ini
[mysqld]
# Características generales
port = 3306
bind-address = 0.0.0.0
max_connections = 500

# Almacenamiento
default-storage-engine = InnoDB
innodb_buffer_pool_size = 4G  # 70-80% de RAM disponible
innodb_log_file_size = 512M
innodb_flush_log_at_trx_commit = 2  # Mejor performance para OLTP

# Seguridad
skip_show_database
local-infile = 0
sql_mode = STRICT_TRANS_TABLES,NO_ENGINE_SUBSTITUTION

# Logging
log_error = /var/log/mysql/error.log
slow_query_log = 1
slow_query_log_file = /var/log/mysql/slow.log
long_query_time = 2
```

---

### 2. PostgreSQL

PostgreSQL (a menudo llamado "Postgres" a secas) es el motor de base de datos de código abierto más avanzado del mundo. Es como la navaja suiza de las bases de datos: tiene herramientas para casi cualquier escenario.

#### Instalación en Ubuntu/Debian

La instalación de PostgreSQL requiere algunos pasos adicionales porque el repositorio oficial de Ubuntu a veces no tiene la versión más reciente.

Primero: **`sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'`**. Este comando agrega el repositorio oficial de PostgreSQL a la lista de fuentes de Ubuntu. Vamos por partes:
- `sh -c '...'`: Ejecuta el comando entre comillas como un script.
- `echo "deb ..."`: Crea una línea de texto que describe la ubicación del repositorio.
- `> /etc/apt/sources.list.d/pgdg.list`: Redirige la salida a un archivo (el `>` "escribe" el resultado en ese archivo).
- `$(lsb_release -cs)`: Sustituye esto por el nombre de tu versión de Ubuntu (por ejemplo, "jammy" para Ubuntu 22.04).

Luego: **`wget -qO- https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -`**. Descarga la clave GPG del repositorio (un archivo que verifica que los paquetes son auténticos y no han sido modificados) y la agrega al sistema.

Después: **`sudo apt update && sudo apt install postgresql-16 postgresql-contrib-16 -y`**. Actualiza la lista de paquetes (ahora incluyendo el repositorio de PostgreSQL) e instala PostgreSQL versión 16 junto con los paquetes contrib (que añaden funcionalidades extras como el módulo `pg_stat_statements` para monitoreo).

Finalmente: **`sudo systemctl start postgresql`** y **`sudo systemctl enable postgresql`**: Inician el servicio y lo configuran para que arranque automáticamente al encender la computadora.

```bash
# Agregar repositorio oficial
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
wget -qO- https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -

# Instalar
sudo apt update
sudo apt install postgresql-16 postgresql-contrib-16 -y

# Verificar
psql --version

# Iniciar servicio
sudo systemctl start postgresql
sudo systemctl enable postgresql
```

#### Configuración Inicial de PostgreSQL

A diferencia de MySQL, PostgreSQL crea un usuario del sistema operativo llamado `postgres` durante la instalación. Este usuario es el "superadministrador" de la base de datos. Para trabajar con PostgreSQL, primero debes cambiar a ese usuario con **`sudo -i -u postgres`**. El flag `-i` inicia una sesión de login como ese usuario, y `-u postgres` especifica el usuario.

Una vez que eres el usuario `postgres`, ejecutas **`psql`** para abrir la consola interactiva de PostgreSQL. Aparecerá un prompt como `postgres=#` indicando que estás dentro de PostgreSQL.

```bash
# Cambiar al usuario postgres
sudo -i -u postgres

# Acceder a psql
psql
```

Ahora dentro de `psql`, ejecutamos los comandos SQL:

- **`CREATE DATABASE sistema_compras ...`**: Crea la base de datos con configuraciones específicas.
  - `WITH ENCODING 'UTF8'`: Define la codificación de caracteres (equivalente a `utf8mb4` en MySQL).
  - `LC_COLLATE 'es_MX.UTF-8'`: Establece la configuración regional para ordenamiento (collation). `es_MX` significa "español de México". Esto afecta cómo se ordenan los textos: la 'ñ' irá después de la 'n', y las vocales acentuadas se ordenarán correctamente.
  - `LC_CTYPE 'es_MX.UTF-8'`: Define la clasificación de caracteres (qué se considera letra, número, etc.).
  - `TEMPLATE template0`: Usa una plantilla limpia para crear la base de datos. `template0` es una base de datos vacía, mientras que `template1` podría incluir objetos personalizados.

- **`CREATE ROLE app_compras WITH LOGIN PASSWORD 'AppCompras2024!'`**: En PostgreSQL, los usuarios se llaman "roles" (roles). `WITH LOGIN` significa que este rol puede iniciar sesión. Si creas un rol sin `LOGIN`, solo sirve como grupo para agrupar permisos.

- **`GRANT CONNECT ON DATABASE sistema_compras TO app_compras`**: Permite que el rol `app_compras` se conecte a la base de datos.

- **`GRANT USAGE ON SCHEMA public TO app_compras`**: Permite usar el esquema `public` (el esquema predeterminado donde se crean las tablas). Sin esto, aunque el usuario pueda conectarse a la base de datos, no podría ver ni usar las tablas.

- **`GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO app_compras`**: Otorga permisos de lectura, inserción, actualización y eliminación sobre todas las tablas existentes en el esquema `public`.

- **`GRANT USAGE ON ALL SEQUENCES IN SCHEMA public TO app_compras`**: Permite usar las secuencias (los contadores automáticos, equivalentes a `AUTO_INCREMENT`). Sin este permiso, el usuario no podría insertar registros en tablas que tienen columnas con valores auto-generados.

```sql
-- Crear base de datos
CREATE DATABASE sistema_compras
    WITH ENCODING 'UTF8'
    LC_COLLATE 'es_MX.UTF-8'
    LC_CTYPE 'es_MX.UTF-8'
    TEMPLATE template0;

-- Crear roles (usuarios)
CREATE ROLE app_compras WITH LOGIN PASSWORD 'AppCompras2024!';
CREATE ROLE analista WITH LOGIN PASSWORD 'Analisis2024!';

-- Otorgar permisos
GRANT CONNECT ON DATABASE sistema_compras TO app_compras;
GRANT USAGE ON SCHEMA public TO app_compras;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO app_compras;
GRANT USAGE ON ALL SEQUENCES IN SCHEMA public TO app_compras;

-- Permisos de solo lectura para analista
GRANT CONNECT ON DATABASE sistema_compras TO analista;
GRANT USAGE ON SCHEMA public TO analista;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO analista;
```

#### Configuración pg_hba.conf

El archivo `pg_hba.conf` (PostgreSQL Host-Based Authentication) es el **archivo de control de acceso** de PostgreSQL. Piensa en él como la **lista de invitados** de una fiesta: aquí defines quién puede entrar, desde dónde, y con qué método de identificación se le permite el acceso.

Cada línea tiene el formato: **tipo | base_de_datos | usuario | dirección | método**

- **`local`**: Conexiones a través de un socket local (cuando te conectas desde la misma máquina sin usar la red). Es más rápida y segura.
- **`host`**: Conexiones a través de la red (TCP/IP), incluso si son desde la misma computadora.
- **`sistema_compras`**: La base de datos a la que se aplica esta regla. Puedes usar `all` para todas las bases de datos.
- **`app_compras`**: El usuario al que se aplica. Puedes usar `all` para todos los usuarios.
- **`192.168.1.0/24`**: La dirección IP o rango de red. `192.168.1.0/24` significa "cualquier IP desde 192.168.1.0 hasta 192.168.1.255". `10.0.0.0/8` significa "cualquier IP desde 10.0.0.0 hasta 10.255.255.255".
- **`scram-sha-256`**: El método de autenticación. `scram-sha-256` es el método más seguro actualmente. `peer` significa que confía en el usuario del sistema operativo (solo para conexiones locales).

**Importante:** Las reglas se evalúan en orden. La primera regla que coincida es la que se aplica. Por eso, las reglas específicas deben ir antes que las generales.

```
# Formato: tipo  base_datos  usuario  dirección  método
# Conexiones locales
local   sistema_compras   app_compras   scram-sha-256
local   sistema_compras   analista      scram-sha-256

# Conexiones remotas (red interna)
host    sistema_compras   app_compras   192.168.1.0/24   scram-sha-256
host    sistema_compras   analista      10.0.0.0/8       scram-sha-256

# Solo local para postgres
local   all              postgres       peer
```

#### Configuración postgresql.conf

El archivo `postgresql.conf` es equivalente al `my.cnf` de MySQL: aquí se configuran todos los parámetros del motor. Veamos los más importantes:

- **`listen_addresses = '*'`**: Le dice a PostgreSQL que escuche conexiones en todas las interfaces de red. Si pones `'localhost'`, solo aceptará conexiones locales.
- **`port = 5432`**: El puerto estándar de PostgreSQL (el 5432). MySQL usa el 3306 para que no se confundan.
- **`max_connections = 200`**: Máximo de conexiones simultáneas (menos que MySQL porque PostgreSQL maneja las conexiones de manera diferente y cada una consume más recursos).
- **`shared_buffers = 2GB`**: Memoria compartida para caché de datos. Similar al `innodb_buffer_pool_size` de MySQL. Se recomienda entre 15% y 25% de la RAM total.
- **`effective_cache_size = 6GB`**: Le dice a PostgreSQL cuánta memoria RAM estima que está disponible para caché del sistema operativo (no es memoria que PostgreSQL reserve, sino una estimación para que el planificador de consultas tome mejores decisiones). Si este valor es muy bajo, PostgreSQL puede subestimar la velocidad de las consultas y elegir planes ineficientes.
- **`work_mem = 64MB`**: Memoria disponible para operaciones como ordenamientos (`ORDER BY`) y uniones (`JOIN`). Por cada consulta que haga un ordenamiento, puede usar hasta 64 MB. Si tienes muchas consultas concurrentes, este valor no debe ser demasiado alto porque la memoria se multiplica por cada consulta.
- **`maintenance_work_mem = 512MB`**: Memoria para tareas de mantenimiento como `VACUUM` (limpieza de registros obsoletos), creación de índices y agregación de claves foráneas. Como estas tareas no ocurren con frecuencia, se puede asignar un valor más alto.
- **`wal_level = replica`**: Configura el nivel de registro de Write-Ahead Log (WAL). `replica` permite hacer replicación (tener copias de la base de datos en otros servidores). El WAL es como un "diario de operaciones" donde se anotan todos los cambios antes de aplicarlos a la base de datos.
- **`max_wal_size = 2GB`**: Tamaño máximo que puede alcanzar el WAL antes de forzar un checkpoint (un punto de sincronización).
- **`min_wal_size = 1GB`**: Tamaño mínimo del WAL. PostgreSQL tratará de mantener al menos este espacio reservado.

```ini
listen_addresses = '*'
port = 5432
max_connections = 200
shared_buffers = 2GB
effective_cache_size = 6GB
work_mem = 64MB
maintenance_work_mem = 512MB
wal_level = replica
max_wal_size = 2GB
min_wal_size = 1GB
```

---

### 3. SQL Server (Linux)

Tradicionalmente, SQL Server solo funcionaba en Windows. Pero desde la versión 2017, Microsoft lanzó SQL Server para Linux. Sí, leíste bien: Microsoft haciendo software para Linux. Esto permite ejecutar SQL Server en servidores Ubuntu sin necesidad de una máquina Windows.

**¿Por qué querrías instalar SQL Server en Linux?** Quizás tu empresa usa SQL Server pero quiere aprovechar servidores Linux (que son más económicos en términos de licencias), o tal vez estás aprendiendo y no tienes acceso a una máquina Windows.

Los comandos:
- **`wget -qO- https://packages.microsoft.com/keys/microsoft.asc | sudo apt-key add -`**: Descarga la clave GPG de Microsoft y la agrega al sistema, para verificar la autenticidad de los paquetes.
- **`sudo add-apt-repository "$(wget -qO- ...)"`**: Agrega el repositorio de Microsoft a la lista de fuentes de Ubuntu. `add-apt-repository` es un comando que simplifica la adición de repositorios.
- **`sudo apt install mssql-server -y`**: Instala el paquete `mssql-server` (la abreviatura de "Microsoft SQL Server").
- **`sudo /opt/mssql/bin/mssql-conf setup`**: Ejecuta el asistente de configuración. Te pedirá que aceptes la licencia (EULA) y que establezcas la contraseña para el usuario `sa` (system administrator, el administrador del sistema). El usuario `sa` es el root de SQL Server.
- **`systemctl status mssql-server`**: Verifica que el servicio esté corriendo.
- **`sudo apt install mssql-tools unixodbc-dev -y`**: Instala las herramientas de línea de comandos de SQL Server (`sqlcmd` para ejecutar consultas y `bcp` para importar/exportar datos) junto con el controlador ODBC necesario para conectarse desde aplicaciones.
- **`echo 'export PATH="$PATH:/opt/mssql-tools/bin"' >> ~/.bashrc && source ~/.bashrc`**: Agrega la carpeta donde están los comandos `sqlcmd` y `bcp` al PATH del sistema (para que puedas ejecutarlos escribiendo solo el nombre, sin la ruta completa). `~/.bashrc` es el archivo de configuración de tu terminal.

```bash
# Importar clave GPG
wget -qO- https://packages.microsoft.com/keys/microsoft.asc | sudo apt-key add -

# Agregar repositorio
sudo add-apt-repository "$(wget -qO- https://packages.microsoft.com/config/ubuntu/22.04/mssql-server-2022.list)"

# Instalar
sudo apt update
sudo apt install mssql-server -y

# Configurar (aceptar licencia, establecer contraseña SA)
sudo /opt/mssql/bin/mssql-conf setup

# Verificar
systemctl status mssql-server

# Instalar herramientas de línea de comandos
sudo apt install mssql-tools unixodbc-dev -y

# Agregar al PATH
echo 'export PATH="$PATH:/opt/mssql-tools/bin"' >> ~/.bashrc
source ~/.bashrc
```

Una vez instalado, usamos `sqlcmd` para conectarnos y configurar:

- **`sqlcmd -S localhost -U SA -P 'TuContraseña!'`**: Conecta a SQL Server usando `sqlcmd`. `-S` especifica el servidor (localhost = esta misma máquina), `-U` el usuario (SA), y `-P` la contraseña.

- **`CREATE DATABASE SistemaInventarios`**: Crea la base de datos. Nota que SQL Server usa `GO` para marcar el final de un bloque de comandos. Es como un "ejecuta esto ahora". En MySQL y PostgreSQL no necesitas `GO`; los comandos se ejecutan inmediatamente cuando pones punto y coma (`;`).

- **`CREATE LOGIN app_inventario WITH PASSWORD = 'AppInv2024!'`**: En SQL Server, `CREATE LOGIN` crea un inicio de sesión a nivel del servidor (puede conectarse al servidor). Luego, `USE SistemaInventarios` cambia a la base de datos, y `CREATE USER app_inventario FOR LOGIN app_inventario` crea un usuario dentro de esa base de datos asociado al login.

- **`EXEC sp_addrolemember 'db_datareader', 'app_inventario'`**: Agrega el usuario al rol `db_datareader` (permiso de lectura en todas las tablas). `EXEC` ejecuta un procedimiento almacenado.

```sql
-- Conectar con sqlcmd
sqlcmd -S localhost -U SA -P 'TuContraseña!'

-- Crear base de datos
CREATE DATABASE SistemaInventarios;
GO

-- Crear usuario de aplicación
CREATE LOGIN app_inventario WITH PASSWORD = 'AppInv2024!';
USE SistemaInventarios;
CREATE USER app_inventario FOR LOGIN app_inventario;
EXEC sp_addrolemember 'db_datareader', 'app_inventario';
EXEC sp_addrolemember 'db_datawriter', 'app_inventario';
GO
```

---

### 4. Docker (Todos los Motores)

Si instalar cada motor de base de datos por separado te parece tedioso, o si no quieres "ensuciar" tu computadora con varios servicios, **Docker es la solución**. Docker te permite ejecutar aplicaciones (incluyendo bases de datos) en "contenedores" aislados. Piensa en los contenedores como **cajas de zapatos**: dentro de cada caja hay una instalación completa de MySQL, PostgreSQL, etc., con todo lo que necesita para funcionar, y puedes tener muchas cajas en tu escritorio sin que se mezclen entre sí.

La gran ventaja: cuando ya no necesites una base de datos, solo eliminas el contenedor y no queda ningún rastro. No tienes que desinstalar ni limpiar archivos residuales.

El archivo `docker-compose.yml` define múltiples servicios en un solo archivo. Vamos a entenderlo:

- **`version: '3.8'`**: La versión del formato de docker-compose. Cada versión soporta diferentes características.
- **`services:`**: Aquí comienza la definición de los servicios (contenedores). Cada servicio es una base de datos.

**Servicio MySQL:**
- **`image: mysql:8.0`**: La imagen de Docker que se va a usar. `mysql:8.0` significa "MySQL versión 8.0".
- **`container_name: mysql_ventas`**: El nombre que le damos al contenedor para identificarlo fácilmente.
- **`environment:`**: Variables de entorno que se pasan al contenedor. `MYSQL_ROOT_PASSWORD` es la contraseña del root, `MYSQL_DATABASE` crea automáticamente una base de datos al iniciar.
- **`ports:`**: Mapea puertos. `"3306:3306"` significa "el puerto 3306 del contenedor se expone como el puerto 3306 de mi computadora". Así te conectas a `localhost:3306`.
- **`volumes:`**: Persiste los datos. Sin volúmenes, cuando elimines el contenedor, perderías todos los datos. `mysql_data:/var/lib/mysql` guarda los datos de MySQL en un volumen llamado `mysql_data`. `./scripts:/docker-entrypoint-initdb.d` monta la carpeta local `./scripts` para que los scripts SQL que pongas allí se ejecuten automáticamente al iniciar el contenedor.

**Servicio PostgreSQL:**
- Sigue el mismo patrón que MySQL, pero con la imagen `postgres:16` y las variables `POSTGRES_DB`, `POSTGRES_USER`, `POSTGRES_PASSWORD`.

**Servicio SQL Server:**
- Usa la imagen oficial de Microsoft `mcr.microsoft.com/mssql/server:2022-latest`.
- `ACCEPT_EULA: "Y"`: Acepta el contrato de licencia de Microsoft (obligatorio).
- `MSSQL_PID: Developer`: Usa la edición Developer (gratuita para desarrollo, pero no para producción).

**Volúmenes:**
Al final, `volumes:` define los volúmenes nombrados (`mysql_data`, `postgres_data`, `sqlserver_data`) que almacenarán los datos de forma persistente. Docker los administra automáticamente.

```yaml
# docker-compose.yml
version: '3.8'

services:
  mysql:
    image: mysql:8.0
    container_name: mysql_ventas
    environment:
      MYSQL_ROOT_PASSWORD: root123
      MYSQL_DATABASE: ventas_dev
    ports:
      - "3306:3306"
    volumes:
      - mysql_data:/var/lib/mysql
      - ./scripts:/docker-entrypoint-initdb.d

  postgres:
    image: postgres:16
    container_name: postgres_compras
    environment:
      POSTGRES_DB: compras_dev
      POSTGRES_USER: devuser
      POSTGRES_PASSWORD: devpass
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

  sqlserver:
    image: mcr.microsoft.com/mssql/server:2022-latest
    container_name: sqlserver_inventario
    environment:
      SA_PASSWORD: "DevPassword123!"
      ACCEPT_EULA: "Y"
      MSSQL_PID: Developer
    ports:
      - "1433:1433"
    volumes:
      - sqlserver_data:/var/opt/mssql

volumes:
  mysql_data:
  postgres_data:
  sqlserver_data:
```

Para usar este archivo:

- **`docker-compose up -d`**: Inicia todos los servicios definidos en `docker-compose.yml`. El flag `-d` significa "detached" (en segundo plano), para que la terminal no se quede bloqueada mostrando los logs.
- **`docker-compose ps`**: Muestra el estado de los servicios (si están corriendo, su nombre, puertos expuestos).
- **`docker exec -i mysql_ventas mysql -uroot -proot123 ventas_dev < init.sql`**: Ejecuta un comando dentro del contenedor `mysql_ventas`. `docker exec` significa "ejecuta en este contenedor". El `< init.sql` redirige el contenido del archivo `init.sql` como entrada para el comando `mysql`, ejecutando así todas las instrucciones SQL del archivo dentro de la base de datos `ventas_dev`.
- **`docker exec -i postgres_compras psql -U devuser -d compras_dev < init.sql`**: Similar, pero para PostgreSQL.

```bash
# Iniciar todos los servicios
docker-compose up -d

# Verificar estado
docker-compose ps

# Ejecutar SQL en MySQL
docker exec -i mysql_ventas mysql -uroot -proot123 ventas_dev < init.sql

# Ejecutar SQL en PostgreSQL
docker exec -i postgres_compras psql -U devuser -d compras_dev < init.sql
```

---

## Herramientas de Administración

Trabajar con bases de datos solo desde la línea de comandos puede ser incómodo, especialmente cuando estás aprendiendo. Por suerte, existen herramientas gráficas que te facilitan la vida. Piensa en ellas como el **escritorio de Windows vs. la línea de comandos**: ambas hacen lo mismo, pero una tiene botones y menús visuales.

### DBeaver (Multiplataforma)

DBeaver es el **cuchillo suizo de las herramientas de bases de datos**. Es gratuita, funciona en Windows, macOS y Linux, y se conecta a prácticamente cualquier motor de base de datos (MySQL, PostgreSQL, SQL Server, Oracle, SQLite, y más de 50 tipos). 

**Lo que puedes hacer con DBeaver:**
- Conectarte a múltiples bases de datos al mismo tiempo, incluso de diferentes tipos.
- Escribir consultas SQL con autocompletado (te sugiere nombres de tablas, columnas, funciones mientras escribes).
- Ver el plan de ejecución de una consulta (cómo piensa la base de datos ejecutar tu SQL, útil para optimizar).
- Generar diagramas entidad-relación (ER) que muestran visualmente las tablas y sus relaciones.
- Exportar e importar datos entre diferentes formatos (CSV, Excel, JSON).

```bash
# Instalar en Ubuntu
sudo snap install dbeaver-ce

# Características:
# - Conecta MySQL, PostgreSQL, SQL Server, Oracle
# - Editor SQL con autocompletado
# - Visualizador de plan de ejecución
# - ER Diagramas
# - Exportación de datos
```

### TablePlus (Multiplataforma)

TablePlus es una herramienta más moderna y ligera que DBeaver. Su interfaz es más limpia y está optimizada para el trabajo diario. No es gratuita (aunque tiene una versión de prueba), pero muchos desarrolladores la prefieren por su velocidad y diseño.

### pgAdmin (PostgreSQL)

pgAdmin es la herramienta de administración **oficial de PostgreSQL**. Es muy completa y específica para este motor: si solo usas PostgreSQL, pgAdmin es probablemente tu mejor opción. Te permite administrar usuarios, respaldos, monitorear consultas en tiempo real, y mucho más.

```bash
# Instalar en Ubuntu
sudo apt install pgadmin4 -y
```

### MySQL Workbench

MySQL Workbench es la herramienta **oficial de MySQL**. Similar a pgAdmin pero para MySQL. Incluye modelado de datos (diseñar diagramas de base de datos), administración de usuarios, y monitoreo del rendimiento.

```bash
# Instalar en Ubuntu
sudo snap install mysql-workbench-community
```

### Azure Data Studio (SQL Server)

Azure Data Studio es la herramienta de Microsoft para SQL Server. Es ligera, multiplataforma (a diferencia de SQL Server Management Studio que solo funciona en Windows), y está enfocada en la productividad del desarrollador. Incluye integración con Git, notebooks (como Jupyter) y gráficos integrados.

```bash
# Descargar e instalar
wget https://azuredatastudio-update.azurewebsites.net/latest/linux-deb-x64/stable
sudo dpkg -i azuredatastudio-linux-*.deb
```

---

## Clientes de Línea de Comandos

Aunque las herramientas gráficas son cómodas, saber usar la línea de comandos es una habilidad fundamental. ¿Por qué? Porque en los servidores de producción (los que usan los clientes reales) normalmente no hay entorno gráfico. Solo tienes una terminal. Además, los comandos son más rápidos y fáciles de automatizar.

### MySQL CLI

La interfaz de línea de comandos de MySQL se invoca con el comando `mysql`. Vamos a ver cómo conectarte y los comandos más útiles:

**Conectar:** `mysql -u usuario -p -h servidor -P 3306 base_datos`
- `-u usuario`: El nombre de usuario.
- `-p`: Te pedirá la contraseña (nunca la escribas en el comando porque queda registrada en el historial).
- `-h servidor`: La dirección IP o nombre del servidor. `localhost` si es la misma máquina.
- `-P 3306`: El puerto (3306 es el de MySQL).
- `base_datos`: El nombre de la base de datos a la que quieres conectarte.

**Comandos dentro de MySQL:**
- **`SHOW DATABASES;`**: Lista todas las bases de datos del servidor.
- **`USE sistema_ventas;`**: Selecciona la base de datos con la que trabajarás.
- **`SHOW TABLES;`**: Muestra todas las tablas dentro de la base de datos actual.
- **`DESCRIBE clientes;`**: Muestra la estructura de la tabla `clientes` (sus columnas, tipos de datos, si acepta nulos, etc.).
- **`SHOW CREATE TABLE clientes;`**: Muestra el comando SQL completo que creó la tabla, útil para entender su estructura o para recrearla en otro lado.
- **`SOURCE /ruta/del/archivo.sql;`**: Ejecuta un archivo SQL. Es como copiar y pegar todo el contenido del archivo en la consola.
- **`SELECT * FROM clientes \G`**: El `\G` al final en lugar de `;` muestra los resultados en formato vertical (cada columna en una línea separada). Esto es útil cuando las tablas tienen muchas columnas y la vista horizontal es ilegible.
- **`\! clear`**: El `\!` ejecuta un comando del sistema operativo. `\! clear` limpia la pantalla de la terminal.
- **`exit;`**: Cierra la conexión y sale de MySQL.

```bash
# Conectar a base de datos
mysql -u usuario -p -h servidor -P 3306 base_datos

# Comandos útiles
mysql> SHOW DATABASES;
mysql> USE sistema_ventas;
mysql> SHOW TABLES;
mysql> DESCRIBE clientes;
mysql> SHOW CREATE TABLE clientes;
mysql> SOURCE /ruta/del/archivo.sql;
mysql> SELECT * FROM clientes \G  # Formato vertical
mysql> \! clear  # Limpiar pantalla
mysql> exit;
```

### PostgreSQL CLI (psql)

La herramienta de línea de comandos de PostgreSQL se llama `psql`. Es muy potente y tiene muchos comandos especiales que empiezan con `\` (barra invertida).

**Conectar:** `psql -h servidor -p 5432 -U usuario -d base_datos`
- Similar a MySQL, pero usas `-d` para la base de datos.

**Comandos especiales (empiezan con `\`):**
- **`\l`**: Lista todas las bases de datos (como `SHOW DATABASES` en MySQL).
- **`\c base_datos`**: Se conecta a otra base de datos (como `USE` en MySQL).
- **`\dt`**: Lista las tablas de la base de datos actual.
- **`\d+ tabla`**: Describe una tabla en detalle, incluyendo índices, restricciones, valores por defecto, etc. Es más detallado que `DESCRIBE` de MySQL.
- **`\di`**: Lista los índices.
- **`\dv`**: Lista las vistas.
- **`\df`**: Lista las funciones (procedimientos almacenados).
- **`\x`**: Activa/desactiva la vista expandida (vertical). Cuando activas `\x`, los resultados de las consultas se muestran en formato vertical, útil para tablas con muchas columnas.
- **`\i /ruta/archivo.sql`**: Ejecuta un archivo SQL.
- **`\o /ruta/salida.txt`**: Redirige la salida de los comandos a un archivo. Todo lo que se muestre después se guardará en ese archivo hasta que ejecutes `\o` sin argumentos.
- **`\e`**: Abre un editor externo (como nano o vim) para escribir consultas más cómodamente.
- **`\q`**: Sale de psql.

```bash
# Conectar
psql -h servidor -p 5432 -U usuario -d base_datos

# Comandos útiles
\l                    # Listar bases de datos
\c base_datos         # Conectar a base de datos
\dt                   # Listar tablas
\d+ tabla             # Describir tabla (detallado)
\di                   # Listar índices
\dv                   # Listar vistas
\df                   # Listar funciones
\x                    # Expandir resultados (formato vertical)
\i /ruta/archivo.sql  # Ejecutar script
\o /ruta/salida.txt   # Redirigir salida a archivo
\e                    # Abrir editor externo
\q                    # Salir
```

### SQL Server CLI (sqlcmd)

`sqlcmd` es la herramienta de línea de comandos de SQL Server.

**Conectar:** `sqlcmd -S servidor -U usuario -P contraseña -d base_datos`
- `-S`: Servidor (puede ser `localhost` o una IP).
- `-U` y `-P`: Usuario y contraseña.
- `-d`: Base de datos.

**Ejecutar consulta directa:** `sqlcmd -S localhost -U SA -Q "SELECT * FROM productos" -d SistemaInventarios`
- `-Q`: Ejecuta la consulta y sale.

**Ejecutar script:** `sqlcmd -S localhost -U SA -i script.sql -o salida.txt`
- `-i`: Archivo de entrada con las consultas SQL.
- `-o`: Archivo de salida donde se guardarán los resultados.

**Modo interactivo:** Cuando ejecutas `sqlcmd -S localhost -U SA` sin `-Q`, entras en modo interactivo. Aparece un prompt con números de línea (`1>`, `2>`, `3>`). Puedes escribir múltiples líneas de SQL, y cuando termines, escribes `GO` en una línea aparte para ejecutar todo.

```bash
# Conectar
sqlcmd -S servidor -U usuario -P contraseña -d base_datos

# Ejecutar consulta
sqlcmd -S localhost -U SA -Q "SELECT * FROM productos" -d SistemaInventarios

# Ejecutar script
sqlcmd -S localhost -U SA -i script.sql -o salida.txt

# Modo interactivo
sqlcmd -S localhost -U SA
1> SELECT * FROM productos
2> WHERE stock < 10
3> ORDER BY nombre
4> GO
```

---

## Scripts de Inicialización para el Proyecto

Cuando comienzas un proyecto nuevo, necesitas crear toda la estructura de base de datos desde cero: las tablas, los índices, las relaciones entre ellas. En lugar de escribir los comandos uno por uno cada vez, los guardas en un **archivo SQL** que puedes ejecutar de una sola vez. Es como tener una receta de cocina escrita: en lugar de recordar todos los ingredientes y pasos, solo sigues la receta.

### Script de carga inicial (MySQL)

El siguiente script es un ejemplo completo de inicialización para un sistema de ventas. Aunque es largo, no te asustes: es solo la repetición de un mismo patrón (CREAR TABLA) con diferentes nombres y columnas.

Primero, creamos la base de datos y la seleccionamos:

```sql
-- init_ventas.sql
-- Crear base de datos del proyecto de ejemplo

CREATE DATABASE IF NOT EXISTS proyecto_ventas;
USE proyecto_ventas;
```

Luego comenzamos con las **tablas maestras** (las tablas fundamentales que contienen los datos base del negocio, como sucursales, almacenes, proveedores, categorías y productos). Estas tablas normalmente existen antes de que ocurra cualquier transacción:

**Tabla `sucursales`:** Guarda las sucursales o tiendas físicas. Cada sucursal tiene un código único, nombre, dirección y ciudad. El campo `activo BOOLEAN DEFAULT TRUE` permite desactivar una sucursal sin eliminar sus datos históricos. Piensa en esto como "dar de baja" una sucursal.

```sql
-- Tablas maestras
CREATE TABLE sucursales (
    id_sucursal INT PRIMARY KEY AUTO_INCREMENT,
    codigo VARCHAR(10) UNIQUE NOT NULL,
    nombre VARCHAR(100) NOT NULL,
    direccion TEXT,
    ciudad VARCHAR(50),
    activo BOOLEAN DEFAULT TRUE
);
```

**Tabla `almacenes`:** Los almacenes están asociados a una sucursal mediante `FOREIGN KEY (id_sucursal) REFERENCES sucursales(id_sucursal)`. Una llave foránea (FOREIGN KEY) garantiza que no puedas crear un almacén asignado a una sucursal que no existe. El campo `tipo` es un ENUM que solo permite tres valores: 'principal', 'secundario' o 'transito'.

```sql
CREATE TABLE almacenes (
    id_almacen INT PRIMARY KEY AUTO_INCREMENT,
    id_sucursal INT NOT NULL,
    codigo VARCHAR(10) UNIQUE NOT NULL,
    nombre VARCHAR(100) NOT NULL,
    tipo ENUM('principal', 'secundario', 'transito') DEFAULT 'principal',
    FOREIGN KEY (id_sucursal) REFERENCES sucursales(id_sucursal)
);
```

**Tabla `proveedores`:** Guarda la información de los proveedores. El RFC (Registro Federal de Contribuyentes) es único para cada proveedor en México. `dias_credito INT DEFAULT 30` indica que por defecto los proveedores dan 30 días de crédito.

```sql
CREATE TABLE proveedores (
    id_proveedor INT PRIMARY KEY AUTO_INCREMENT,
    rfc VARCHAR(13) UNIQUE NOT NULL,
    nombre_comercial VARCHAR(150) NOT NULL,
    razon_social VARCHAR(200),
    telefono VARCHAR(20),
    email VARCHAR(150),
    dias_credito INT DEFAULT 30,
    activo BOOLEAN DEFAULT TRUE
);
```

**Tabla `categorias`:** Las categorías tienen una característica especial: pueden tener una **jerarquía**. `id_categoria_padre` es una llave foránea que apunta a la misma tabla `categorias`. Esto permite crear subcategorías (por ejemplo: "Electrónicos" es la categoría padre, y "Laptops", "Tablets", "Teléfonos" son categorías hijas). Es un patrón común llamado "auto-referencia".

```sql
CREATE TABLE categorias (
    id_categoria INT PRIMARY KEY AUTO_INCREMENT,
    nombre VARCHAR(100) NOT NULL,
    id_categoria_padre INT,
    FOREIGN KEY (id_categoria_padre) REFERENCES categorias(id_categoria)
);
```

**Tabla `productos`:** La tabla más importante del catálogo. `codigo_barras` es el código de barras del producto (EAN-13 o similar). `sku` es el SKU (Stock Keeping Unit), un código interno que usa la empresa para identificar el producto. Ambos son UNIQUE (no pueden repetirse). `precio_venta` es obligatorio (NOT NULL) porque no tendría sentido tener un producto sin precio. `iva DECIMAL(4,2) DEFAULT 16.00` representa el IVA del 16% en México.

```sql
CREATE TABLE productos (
    id_producto INT PRIMARY KEY AUTO_INCREMENT,
    codigo_barras VARCHAR(50) UNIQUE,
    sku VARCHAR(30) UNIQUE NOT NULL,
    nombre VARCHAR(200) NOT NULL,
    id_categoria INT,
    precio_venta DECIMAL(12,2) NOT NULL,
    precio_compra DECIMAL(12,2),
    iva DECIMAL(4,2) DEFAULT 16.00,
    unidad_medida VARCHAR(20) DEFAULT 'pieza',
    activo BOOLEAN DEFAULT TRUE,
    FOREIGN KEY (id_categoria) REFERENCES categorias(id_categoria)
);
```

Ahora pasamos a las **tablas de clientes**:

**Tabla `clientes`:** `tipo_persona` puede ser 'fisica' (persona física, un individuo) o 'moral' (persona moral, una empresa). `credito_limite` es el máximo que se le puede fiar al cliente, y `saldo_actual` es cuánto debe actualmente. `lista_precios` indica qué lista de precios aplica a este cliente ('publico', 'mayoreo', 'distribuidor').

```sql
-- Tablas de clientes
CREATE TABLE clientes (
    id_cliente INT PRIMARY KEY AUTO_INCREMENT,
    codigo_cliente VARCHAR(20) UNIQUE,
    tipo_persona ENUM('fisica', 'moral') DEFAULT 'fisica',
    nombre VARCHAR(150) NOT NULL,
    rfc VARCHAR(13),
    email VARCHAR(150),
    telefono VARCHAR(20),
    direccion TEXT,
    fecha_registro DATETIME DEFAULT CURRENT_TIMESTAMP,
    credito_limite DECIMAL(12,2) DEFAULT 0,
    saldo_actual DECIMAL(12,2) DEFAULT 0,
    lista_precios ENUM('publico', 'mayoreo', 'distribuidor') DEFAULT 'publico',
    activo BOOLEAN DEFAULT TRUE
);
```

Y finalmente las **tablas transaccionales** (las que registran eventos de negocio: facturas, órdenes de compra, movimientos de inventario):

**Tabla `facturas`:** Registra cada factura o ticket. `folio` es único y puede ser un número consecutivo como "F-0001". `tipo_documento` puede ser factura, ticket, nota de crédito (devolución) o nota de débito (cargo adicional). `estado` permite saber si la factura está activa, cancelada, o si se devolvieron parcial o totalmente los productos. Las columnas financieras (subtotal, descuento, iva, total) se almacenan por separado para poder reportarlas fácilmente.

```sql
-- Tablas transaccionales
CREATE TABLE facturas (
    id_factura INT PRIMARY KEY AUTO_INCREMENT,
    folio VARCHAR(30) UNIQUE NOT NULL,
    id_cliente INT NOT NULL,
    id_sucursal INT,
    fecha_emision DATETIME DEFAULT CURRENT_TIMESTAMP,
    tipo_documento ENUM('factura', 'ticket', 'nota_credito', 'nota_debito') DEFAULT 'factura',
    subtotal DECIMAL(12,2),
    descuento DECIMAL(12,2) DEFAULT 0,
    iva DECIMAL(12,2),
    total DECIMAL(12,2),
    metodo_pago ENUM('efectivo', 'tarjeta_credito', 'tarjeta_debito', 
                     'transferencia', 'cheque', 'credito'),
    forma_pago VARCHAR(50),
    estado ENUM('activa', 'cancelada', 'devuelta_parcial', 'devuelta_total') DEFAULT 'activa',
    notas TEXT,
    FOREIGN KEY (id_cliente) REFERENCES clientes(id_cliente),
    FOREIGN KEY (id_sucursal) REFERENCES sucursales(id_sucursal)
);
```

**Tabla `facturas_detalle`:** Es el **detalle** de cada factura. Una factura puede tener múltiples productos (por ejemplo, una factura con 3 productos diferentes). Este detalle se guarda aquí, no en la tabla de facturas. La relación es de "uno a muchos": una factura tiene muchos detalles. Cada línea del detalle tiene el producto, la cantidad, el precio unitario, los descuentos y los totales.

```sql
CREATE TABLE facturas_detalle (
    id_detalle INT PRIMARY KEY AUTO_INCREMENT,
    id_factura INT NOT NULL,
    id_producto INT NOT NULL,
    cantidad INT NOT NULL,
    precio_unitario DECIMAL(12,2) NOT NULL,
    descuento DECIMAL(12,2) DEFAULT 0,
    subtotal DECIMAL(12,2),
    iva DECIMAL(12,2),
    total DECIMAL(12,2),
    FOREIGN KEY (id_factura) REFERENCES facturas(id_factura),
    FOREIGN KEY (id_producto) REFERENCES productos(id_producto)
);
```

**Tabla `ordenes_compra`:** Similar a facturas, pero para compras a proveedores. El `estado` sigue un flujo: 'borrador' (se está armando), 'enviada' (se mandó al proveedor), 'confirmada' (el proveedor aceptó), 'recibida_parcial' (llegó parte), 'recibida_total' (llegó todo), 'cancelada'.

```sql
CREATE TABLE ordenes_compra (
    id_orden INT PRIMARY KEY AUTO_INCREMENT,
    folio VARCHAR(30) UNIQUE NOT NULL,
    id_proveedor INT NOT NULL,
    id_sucursal INT,
    fecha_emision DATETIME DEFAULT CURRENT_TIMESTAMP,
    fecha_estimada_entrega DATE,
    fecha_real_entrega DATE,
    subtotal DECIMAL(12,2),
    iva DECIMAL(12,2),
    total DECIMAL(12,2),
    estado ENUM('borrador', 'enviada', 'confirmada', 'recibida_parcial', 
                'recibida_total', 'cancelada') DEFAULT 'borrador',
    FOREIGN KEY (id_proveedor) REFERENCES proveedores(id_proveedor),
    FOREIGN KEY (id_sucursal) REFERENCES sucursales(id_sucursal)
);
```

**Tabla `ordenes_compra_detalle`:** El detalle de cada orden de compra. Nota que tiene `cantidad_solicitada` y `cantidad_recibida` por separado, porque puede que te envíen parcialmente los productos.

```sql
CREATE TABLE ordenes_compra_detalle (
    id_detalle INT PRIMARY KEY AUTO_INCREMENT,
    id_orden INT NOT NULL,
    id_producto INT NOT NULL,
    cantidad_solicitada INT NOT NULL,
    cantidad_recibida INT DEFAULT 0,
    precio_unitario DECIMAL(12,2) NOT NULL,
    subtotal DECIMAL(12,2),
    iva DECIMAL(12,2),
    total DECIMAL(12,2),
    FOREIGN KEY (id_orden) REFERENCES ordenes_compra(id_orden),
    FOREIGN KEY (id_producto) REFERENCES productos(id_producto)
);
```

**Tabla `inventario_movimientos`:** Lleva el registro de **cada movimiento de inventario**: entradas por compra, salidas por venta, ajustes, transferencias entre almacenes, mermas (productos dañados o caducados), devoluciones de clientes, etc. `referencia_tipo` y `referencia_id` permiten vincular el movimiento con el documento que lo originó (por ejemplo, una factura, una orden de compra, un ajuste manual).

```sql
CREATE TABLE inventario_movimientos (
    id_movimiento INT PRIMARY KEY AUTO_INCREMENT,
    id_producto INT NOT NULL,
    id_almacen INT,
    tipo_movimiento ENUM('entrada_compra', 'salida_venta', 'ajuste_entrada',
                         'ajuste_salida', 'transferencia_entrada', 
                         'transferencia_salida', 'produccion_entrada',
                         'produccion_salida', 'merma', 'devolucion_cliente',
                         'devolucion_proveedor'),
    cantidad INT NOT NULL,
    costo_unitario DECIMAL(12,2),
    referencia_tipo VARCHAR(50),
    referencia_id INT,
    fecha_movimiento DATETIME DEFAULT CURRENT_TIMESTAMP,
    usuario VARCHAR(100),
    notas TEXT,
    FOREIGN KEY (id_producto) REFERENCES productos(id_producto),
    FOREIGN KEY (id_almacen) REFERENCES almacenes(id_almacen)
);
```

Finalmente, los **índices**: son como los índices de un libro. Cuando buscas una palabra en un libro sin índice, tienes que hojear página por página. Con un índice, vas directamente a la página correcta. Los índices en una base de datos hacen exactamente eso: aceleran las búsquedas por las columnas que más se consultan.

Cada `CREATE INDEX` crea un índice sobre una o más columnas de una tabla:
- `idx_facturas_fecha ON facturas(fecha_emision)`: Acelera las búsquedas de facturas por fecha.
- `idx_facturas_cliente ON facturas(id_cliente)`: Acelera las búsquedas de facturas de un cliente específico.
- `idx_clientes_rfc ON clientes(rfc)`: Acelera la búsqueda de clientes por RFC.
- `idx_productos_sku ON productos(sku)`: Acelera la búsqueda de productos por SKU.

```sql
-- Índices para performance
CREATE INDEX idx_facturas_fecha ON facturas(fecha_emision);
CREATE INDEX idx_facturas_cliente ON facturas(id_cliente);
CREATE INDEX idx_facturas_estado ON facturas(estado);
CREATE INDEX idx_facturas_detalle_factura ON facturas_detalle(id_factura);
CREATE INDEX idx_facturas_detalle_producto ON facturas_detalle(id_producto);
CREATE INDEX idx_ordenes_compra_proveedor ON ordenes_compra(id_proveedor);
CREATE INDEX idx_ordenes_compra_estado ON ordenes_compra(estado);
CREATE INDEX idx_inventario_movimientos_producto ON inventario_movimientos(id_producto);
CREATE INDEX idx_inventario_movimientos_fecha ON inventario_movimientos(fecha_movimiento);
CREATE INDEX idx_inventario_movimientos_tipo ON inventario_movimientos(tipo_movimiento);
CREATE INDEX idx_clientes_rfc ON clientes(rfc);
CREATE INDEX idx_productos_sku ON productos(sku);
```

---

## Conexión desde Lenguajes de Programación

Una base de datos no sirve de mucho si no puedes conectarte a ella desde tu aplicación. Aquí te muestro cómo conectarte desde los lenguajes de programación más populares: Python, Node.js, Java y C#.

**El patrón general es el mismo en todos:**
1. Importas la librería o módulo del motor de base de datos que vas a usar.
2. Creas una conexión especificando el servidor, puerto, usuario, contraseña y base de datos.
3. Creas un "cursor" o "statement" para ejecutar consultas.
4. Ejecutas la consulta SQL (con parámetros para evitar inyección SQL).
5. Procesas los resultados.
6. Cierras la conexión.

### Python

Python es uno de los lenguajes más usados para trabajar con bases de datos por su sintaxis limpia y sus potentes librerías.

**Primero, instala las librerías necesarias:**
```bash
pip install mysql-connector-python psycopg2-binary pyodbc
```

**Para MySQL:**
- `mysql.connector.connect(...)`: Crea la conexión al servidor MySQL. Todos los parámetros son autoexplicativos: host, user, password, database.
- `conn.cursor(dictionary=True)`: Crea un cursor. Con `dictionary=True`, los resultados se devuelven como diccionarios (con nombres de columna como llaves), lo cual es más cómodo que el formato por defecto (tuplas).
- `cursor.execute("SELECT * FROM clientes WHERE activo = %s", (True,))`: Ejecuta la consulta. Nota que usamos `%s` como marcador de posición para el parámetro `True`, en lugar de escribirlo directamente en el string SQL. Esto es **fundamental para la seguridad**: previene la inyección SQL, un tipo de ataque donde un usuario malintencionado podría escribir código SQL en lugar de datos.
- `cursor.fetchall()`: Obtiene todas las filas del resultado.
- `cursor.close()` y `conn.close()`: Cierran el cursor y la conexión, liberando recursos.

**Para PostgreSQL:**
- Similar a MySQL, pero usando `psycopg2.connect(...)`. Nota que PostgreSQL usa `dbname` en lugar de `database`.

```python
# pip install mysql-connector-python psycopg2-binary pyodbc

# MySQL
import mysql.connector

conn = mysql.connector.connect(
    host='localhost',
    user='app_ventas',
    password='ContraseñaSegura123!',
    database='sistema_ventas'
)
cursor = conn.cursor(dictionary=True)
cursor.execute("SELECT * FROM clientes WHERE activo = %s", (True,))
clientes = cursor.fetchall()
cursor.close()
conn.close()

# PostgreSQL
import psycopg2

conn = psycopg2.connect(
    host='localhost',
    dbname='sistema_compras',
    user='app_compras',
    password='AppCompras2024!'
)
cursor = conn.cursor()
cursor.execute("SELECT * FROM proveedores WHERE activo = %s", (True,))
proveedores = cursor.fetchall()
cursor.close()
conn.close()
```

### Node.js

Node.js es el entorno de ejecución de JavaScript del lado del servidor. Es muy popular para aplicaciones web modernas.

**Primero instala las librerías:**
```bash
npm install mysql2 pg tedious
```

- `mysql.createConnection(...)`: Crea la conexión a MySQL. `mysql2` es una versión moderna del conector MySQL que soporta promesas.
- `conn.execute(...)`: Ejecuta la consulta. Usa `?` como marcador de posición para los parámetros. Devuelve un arreglo: en la posición `[0]` están las filas, en `[1]` los metadatos.
- `await conn.end()`: Cierra la conexión (como es una función asíncrona, se usa `await`).

```javascript
// npm install mysql2 pg tedious

// MySQL
const mysql = require('mysql2/promise');
const conn = await mysql.createConnection({
    host: 'localhost',
    user: 'app_ventas',
    password: 'ContraseñaSegura123!',
    database: 'sistema_ventas'
});
const [rows] = await conn.execute(
    'SELECT * FROM productos WHERE stock_actual < ?',
    [10]
);
await conn.end();
```

### Java (JDBC)

Java usa JDBC (Java Database Connectivity), una API estándar para conectarse a bases de datos.

- **`DriverManager.getConnection(url, user, pass)`**: Obtiene una conexión usando la URL de conexión. La URL tiene el formato `jdbc:mysql://host:puerto/base_datos`.
- **`PreparedStatement`**: Es la forma segura de ejecutar consultas con parámetros. Usa `?` como marcadores y los estableces con `stmt.setInt(1, 10)` (donde 1 es la posición del primer `?`).
- **`ResultSet`**: Contiene los resultados. `rs.next()` avanza a la siguiente fila y devuelve `false` cuando no hay más.
- **Try-with-resources**: El `try (...)` cierra automáticamente la conexión, el statement y el result set al salir del bloque, incluso si ocurre una excepción.

```java
// MySQL
String url = "jdbc:mysql://localhost:3306/sistema_ventas";
String user = "app_ventas";
String pass = "ContraseñaSegura123!";

try (Connection conn = DriverManager.getConnection(url, user, pass);
     PreparedStatement stmt = conn.prepareStatement(
         "SELECT * FROM productos WHERE stock_actual < ?")) {
    stmt.setInt(1, 10);
    ResultSet rs = stmt.executeQuery();
    while (rs.next()) {
        System.out.println(rs.getString("nombre_producto"));
    }
}
```

### C# (.NET)

En el mundo .NET, se usa `SqlConnection` para conectarse a SQL Server.

- **`connectionString`**: Una cadena de texto con todos los parámetros de conexión: servidor, base de datos, usuario, contraseña, y opciones adicionales como `TrustServerCertificate=true` (necesario para conexiones con certificados autofirmados en desarrollo).
- **`SqlConnection`**: Representa la conexión.
- **`SqlCommand`**: La consulta a ejecutar. `@minStock` es el marcador de parámetro (en lugar de `?` como en otros lenguajes, SQL Server usa `@nombre`).
- **`cmd.Parameters.AddWithValue("@minStock", 10)`**: Asigna el valor al parámetro.
- **`ExecuteReaderAsync()`**: Ejecuta la consulta y devuelve un lector para recorrer los resultados.
- **`reader["nombre_producto"]`**: Accede al valor de la columna por su nombre.

```csharp
// SQL Server
string connectionString = "Server=localhost;Database=SistemaInventarios;" +
    "User Id=app_inventario;Password=AppInv2024!;TrustServerCertificate=true;";

using var conn = new SqlConnection(connectionString);
await conn.OpenAsync();
var cmd = new SqlCommand(
    "SELECT * FROM productos WHERE stock_actual < @minStock", conn);
cmd.Parameters.AddWithValue("@minStock", 10);
var reader = await cmd.ExecuteReaderAsync();
while (await reader.ReadAsync()) {
    Console.WriteLine(reader["nombre_producto"]);
}
```

---

## Anterior: [02 Tipos De Bases De Datos](../01-introduccion/02-tipos-de-bases-de-datos.md)
## Siguiente: [01 Select Basico](../02-fundamentos/01-select-basico.md)

[Volver al índice](../README.md)
