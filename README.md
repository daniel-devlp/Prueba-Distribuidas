# Prueba-Distribuidas
https://github.com/daniel-devlp/Prueba-Distribuidas
# Configuración de la Replicación con MySQL

Este documento describe los pasos necesarios para configurar la replicación maestro-esclavo con MySQL en sistemas basados en Ubuntu/Debian.

---

## MASTER

### 1. Actualiza los repositorios
```bash
sudo apt update
```
### 2. Instala los paquetes necesarios para trabajar con repositorios HTTPS
```bash
sudo apt install wget lsb-release gnupg -y
```
### 3. Agrega el repositorio oficial de MySQL
```bash
wget https://dev.mysql.com/get/mysql-apt-config_0.8.29-1_all.deb
```
### 4. Instala el repositorio descargado
```bash
sudo dpkg -i mysql-apt-config_0.8.29-1_all.deb
```
### 5. Vuelve a actualizar los repositorios (ahora con los de MySQL)
```bash
sudo apt update
```
### 6. Instala MySQL Server
```bash
sudo apt install mysql-server -y
```
### 7. Verifica que el servicio esté activo
```bash
sudo systemctl status mysql
```

---

### 8. Configura el archivo `my.cnf` (o `my.ini`) en el maestro

Asegúrate de incluir o modificar estas líneas en el archivo de configuración (`/etc/mysql/my.cnf` o `/etc/mysql/mysql.conf.d/mysqld.cnf`):

```ini
[mysqld]
server-id=1
log_bin=mysql-bin
bind-address=0.0.0.0
```

Reinicia el servicio después de modificar el archivo:

```bash
sudo systemctl restart mysql
```

---

### 9. Crear el usuario de replicación

En la consola de MySQL:

```sql
CREATE USER 'replicator'@'%' IDENTIFIED BY 'admin123';
GRANT REPLICATION SLAVE ON *.* TO 'replicator'@'%';
FLUSH PRIVILEGES;
```

---

### 10. Mostrar el estado del maestro

En la consola de MySQL:

```sql
SHOW MASTER STATUS;
```

---

### 11. Pruebas de funcionamiento

Inserta un registro de prueba para verificar la replicación:

```sql
INSERT INTO CentrosMedicos (Nombre, Ciudad, Direccion, Telefono)
VALUES ('Centro Ejemplo', 'Ciudad Ejemplo', 'Dirección', '123456789');
```

---

## SLAVE

### 1. Actualiza los repositorios
```bash
sudo apt update
```
### 2. Instala los paquetes necesarios para trabajar con repositorios HTTPS
```bash
sudo apt install wget lsb-release gnupg -y
```
### 3. Agrega el repositorio oficial de MySQL
```bash
wget https://dev.mysql.com/get/mysql-apt-config_0.8.29-1_all.deb
```
### 4. Instala el repositorio descargado
```bash
sudo dpkg -i mysql-apt-config_0.8.29-1_all.deb
```
### 5. Vuelve a actualizar los repositorios (ahora con los de MySQL)
```bash
sudo apt update
```
### 6. Instala MySQL Server
```bash
sudo apt install mysql-server -y
```
### 7. Verifica que el servicio esté activo
```bash
sudo systemctl status mysql
```

---

### 8. Configura el archivo `my.cnf` (o `my.ini`) en el esclavo

Asegúrate de incluir o modificar estas líneas en el archivo de configuración (`/etc/mysql/my.cnf` o `/etc/mysql/mysql.conf.d/mysqld.cnf`):

```ini
[mysqld]
server-id=2
relay-log=relay-log
```

Reinicia el servicio después de modificar el archivo:

```bash
sudo systemctl restart mysql
```

---

### 9. Configuración del SLAVE

En la consola de MySQL, ejecuta (reemplaza los valores según corresponda):

```sql
CHANGE MASTER TO
  MASTER_HOST='34.42.193.190',
  MASTER_USER='replicator',
  MASTER_PASSWORD='admin123',
  MASTER_LOG_FILE='mysql-bin.000001',
  MASTER_LOG_POS=5317;
START SLAVE;
```

> **Nota:** Los valores de `MASTER_LOG_FILE` y `MASTER_LOG_POS` deben coincidir con los obtenidos del comando `SHOW MASTER STATUS;` en el maestro.

---

### 10. Verificar el estado del esclavo

En la consola de MySQL:

```sql
SHOW SLAVE STATUS\G
```

Revisa que `Slave_IO_Running` y `Slave_SQL_Running` estén en `Yes`.

---

## Notas Finales

- Asegúrate de que los puertos y el firewall permitan la comunicación entre el master y el slave (por defecto, MySQL usa el puerto 3306).
- Recuerda cambiar las contraseñas y configuraciones sensibles antes de usar en producción.

---
# Configuración de la Replicación con Postgres

Este documento describe los pasos para configurar la replicación síncrona en PostgreSQL entre un servidor maestro (**master**) y un servidor esclavo (**slave**).

---

## Requisitos Previos

- Dos servidores con PostgreSQL instalado (en este ejemplo, versión 17).
- Acceso como usuario `postgres` en ambos servidores.
- Conectividad de red entre ambos servidores.

---

## 1. Configuración en el Maestro

### 1.1 Editar `postgresql.conf`

Ajusta los siguientes parámetros:

```conf
listen_addresses = '*'
wal_level = replica
max_wal_senders = 10
wal_keep_size = 64
synchronous_commit = on
synchronous_standby_names = 'pgslave'
```

### 1.2 Editar `pg_hba.conf`

Agrega una línea para permitir la conexión de replicación desde el esclavo:

```
host    replication     replicator     <IP_SLAVE>/32      md5
```

### 1.3 Crear el usuario de replicación

En el prompt de `psql`:

```sql
CREATE ROLE replicator WITH REPLICATION LOGIN PASSWORD 'tu_password';
```

### 1.4 Reiniciar el servicio

```sh
sudo systemctl restart postgresql@17-main
```

---

## 2. Configuración en el Esclavo

### 2.1 Detener el servicio y limpiar el directorio de datos

```sh
sudo systemctl stop postgresql@17-main
sudo rm -rf /var/lib/postgresql/17/main/*
```

### 2.2 Realizar la copia base desde el maestro

```sh
sudo -u postgres pg_basebackup -h <IP_MAESTRO> -D /var/lib/postgresql/17/main -U replicator -P -R
```

### 2.3 Verificar archivos y parámetros

- Confirma que exista el archivo `standby.signal`.
- Revisa el parámetro `primary_conninfo` en `postgresql.auto.conf`.

### 2.4 Habilitar consultas de solo lectura (opcional)

En `postgresql.conf` del esclavo:

```conf
hot_standby = on
```

### 2.5 Iniciar el servicio

```sh
sudo systemctl start postgresql@17-main
```

---

## 3. Verificación de la Replicación

### 3.1 En el Maestro

Verifica la conexión del esclavo:

```sql
SELECT client_addr, state, sync_state, application_name FROM pg_stat_replication;
```

Debes ver una fila con `state = streaming` y `sync_state = sync`.

### 3.2 Prueba de replicación

En el maestro:

```sql
INSERT INTO CentrosMedicos (Nombre, Ciudad, Direccion, Telefono)
VALUES ('Clínica Central', 'Lima', 'Av. Principal 1234', '012345678');
```

En el esclavo:

```sql
SELECT * FROM CentrosMedicos;
```

El registro debe aparecer automáticamente en el esclavo.

---

## Notas

- El esclavo está en modo solo lectura.
- La replicación síncrona garantiza alta disponibilidad y consistencia, ya que el maestro solo confirma las transacciones cuando el esclavo las ha recibido.

---
