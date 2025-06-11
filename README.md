# Prueba-Distribuidas
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
