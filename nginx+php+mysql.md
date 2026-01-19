# NGINX+PHP+MYSQL

***

## &#x20;NGINX + PHP + MySQL&#x20;

### 1. Configuración de red y acceso a la VM

```bash
ip a
```

• Muestra la IP asignada a la VM.\
• La necesitas para probar desde navegador o curl.

```bash
sudo apt install openssh-server
sudo systemctl enable ssh
sudo systemctl start ssh
```

• Instala y activa SSH para conectarte remotamente desde tu ordenador.

***

### 2. Instalación de Nginx

```bash
sudo apt update
sudo apt install nginx
```

• Instala el servidor web Nginx.

```bash
sudo systemctl start nginx
sudo systemctl status nginx
```

• Inicia el servicio y comprueba que esté activo.

```bash
curl http://localhost
```

• Si funciona, devuelve la página por defecto de Nginx.

***

### 3. Estructura del sitio

```bash
sudo mkdir -p /var/www/camilly.com
```

• Carpeta donde vivirá tu sitio web.

```bash
sudo chown -R www-data:www-data /var/www/camilly.com
sudo chmod -R 755 /var/www/camilly.com
```

• Permisos correctos para que Nginx pueda leer los archivos.\
• Si fallan los permisos → error 500.

***

### 4. Virtual Host

```bash
sudo cp /etc/nginx/sites-available/default \
        /etc/nginx/sites-available/camilly.com.conf
```

• Creas una copia base del host del sitio.

_(Aquí agregas dentro de ese archivo la config final que hicimos: server\_name, root, fastcgi\_pass, etc.)_

```bash
sudo ln -s /etc/nginx/sites-available/camilly.com.conf \
           /etc/nginx/sites-enabled/
```

• Activa el sitio.

```bash
sudo nginx -t
```

• Verifica errores sintácticos.

```bash
sudo systemctl reload nginx
```

• Recarga Nginx para aplicar cambios.

***

### 5. Instalación de PHP-FPM

```bash
sudo apt install php php-fpm php-mysql
```

• Instala PHP, PHP-FPM y php-mysql.\
• php-mysql es necesario para usar _mysqli\_connect()_.

```bash
sudo systemctl status php8.4-fpm
```

• Verifica que PHP-FPM esté corriendo.

```bash
ls /run/php/
```

• Aquí ves el socket (ej: `php8.4-fpm.sock`) que necesitas poner en `fastcgi_pass`.

***

### 6. Pruebas PHP

```bash
sudo nano /var/www/camilly.com/info.php
```

• Archivo de prueba.

```php
<?php phpinfo();
```

• Muestra la configuración de PHP desde el navegador.

***

### 7. Base de datos: MariaDB/MySQL

```bash
sudo apt install mariadb-server
```

• Instala MariaDB.

```bash
sudo mysql
```

• Accede al cliente SQL.

```sql
CREATE DATABASE camilly;
CREATE USER 'cami'@'localhost' IDENTIFIED BY '1234';
GRANT ALL PRIVILEGES ON camilly.* TO 'cami'@'localhost';
USE camilly;
CREATE TABLE usuarios (id INT PRIMARY KEY AUTO_INCREMENT, nombre VARCHAR(50));
INSERT INTO usuarios(nombre) VALUES ('Ana');
SELECT * FROM usuarios;
```

• Crea BD, usuario, permisos y tabla de prueba.

***

### 8. Scripts PHP del proyecto

```php
conexion.php
usuarios.php
index.php
```

• Archivos principales del flujo PHP→MySQL.\
• `conexion.php` maneja la conexión.\
• `usuarios.php` consulta la BD.\
• `index.php` muestra los datos en HTML.

***

### 9. Logs y depuración

```bash
sudo tail -f /var/log/nginx/error.log
```

• Errores del sitio.

```bash
sudo tail -f /var/log/php8.4-fpm.log
```

• Errores de PHP.

```bash
sudo systemctl status php8.4-fpm
```

• Revisa que PHP-FPM no tenga fallas.

***

### 10. Problemas comunes que resolvimos

**Error 500**\
• Socket PHP-FPM incorrecto.\
• Permisos mal configurados.\
• fastcgi\_pass mal escrito.

**Puerto 80 en conflicto**\
• Dos sitios usando el mismo Perfecto. Aquí lo tienes **sin comentarios dentro del bloque**, y **los comentarios explicativos debajo**, estilo GitBook claro y limpio.

***

## NGINX + PHP + MySQL&#x20;

### 1. Configuración de red y acceso a la VM

```bash
ip a
```

• Muestra la IP asignada a la VM.\
• La necesitas para probar desde navegador o curl.

```bash
sudo apt install openssh-server
sudo systemctl enable ssh
sudo systemctl start ssh
```

• Instala y activa SSH para conectarte remotamente desde tu ordenador.

***

### 2. Instalación de Nginx

```bash
sudo apt update
sudo apt install nginx
```

• Instala el servidor web Nginx.

```bash
sudo systemctl start nginx
sudo systemctl status nginx
```

• Inicia el servicio y comprueba que esté activo.

```bash
curl http://localhost
```

• Si funciona, devuelve la página por defecto de Nginx.

***

### 3. Estructura del sitio

```bash
sudo mkdir -p /var/www/camilly.com
```

• Carpeta donde vivirá tu sitio web.

```bash
sudo chown -R www-data:www-data /var/www/camilly.com
sudo chmod -R 755 /var/www/camilly.com
```

• Permisos correctos para que Nginx pueda leer los archivos.\
• Si fallan los permisos → error 500.

***

### 4. Virtual Host

```bash
sudo cp /etc/nginx/sites-available/default \
        /etc/nginx/sites-available/camilly.com.conf
```

• Creas una copia base del host del sitio.

_(Aquí agregas dentro de ese archivo la config final que hicimos: server\_name, root, fastcgi\_pass, etc.)_

```bash
sudo ln -s /etc/nginx/sites-available/camilly.com.conf \
           /etc/nginx/sites-enabled/
```

• Activa el sitio.

```bash
sudo nginx -t
```

• Verifica errores sintácticos.

```bash
sudo systemctl reload nginx
```

• Recarga Nginx para aplicar cambios.

***

### 5. Instalación de PHP-FPM

```bash
sudo apt install php php-fpm php-mysql
```

• Instala PHP, PHP-FPM y php-mysql.\
• php-mysql es necesario para usar _mysqli\_connect()_.

```bash
sudo systemctl status php8.4-fpm
```

• Verifica que PHP-FPM esté corriendo.

```bash
ls /run/php/
```

• Aquí ves el socket (ej: `php8.4-fpm.sock`) que necesitas poner en `fastcgi_pass`.

***

### 6. Pruebas PHP

```bash
sudo nano /var/www/camilly.com/info.php
```

• Archivo de prueba.

```php
<?php phpinfo();
```

• Muestra la configuración de PHP desde el navegador.

***

### 7. Base de datos: MariaDB/MySQL

```bash
sudo apt install mariadb-server
```

• Instala MariaDB.

```bash
sudo mysql
```

• Accede al cliente SQL.

```sql
CREATE DATABASE camilly;
CREATE USER 'cami'@'localhost' IDENTIFIED BY '1234';
GRANT ALL PRIVILEGES ON camilly.* TO 'cami'@'localhost';
USE camilly;
CREATE TABLE usuarios (id INT PRIMARY KEY AUTO_INCREMENT, nombre VARCHAR(50));
INSERT INTO usuarios(nombre) VALUES ('Ana');
SELECT * FROM usuarios;
```

• Crea BD, usuario, permisos y tabla de prueba.

***

### 8. Scripts PHP del proyecto

```php
conexion.php
usuarios.php
index.php
```

• Archivos principales del flujo PHP→MySQL.\
• `conexion.php` maneja la conexión.\
• `usuarios.php` consulta la BD.\
• `index.php` muestra los datos en HTML.

***

### 9. Logs y depuración

```bash
sudo tail -f /var/log/nginx/error.log
```

• Errores del sitio.

```bash
sudo tail -f /var/log/php8.4-fpm.log
```

• Errores de PHP.

```bash
sudo systemctl status php8.4-fpm
```

• Revisa que PHP-FPM no tenga fallas.

***

### 10. Problemas comunes que resolvimos

**Error 500**\
• Socket PHP-FPM incorrecto.\
• Permisos mal configurados.\
• fastcgi\_pass mal escrito.

**Puerto 80 en conflicto**\
• Dos sitios usando el mismo listen 80.

**mysqli\_connect() undefined**\
• Faltaba php-mysql.

**Archivos con propietario root**\
• PHP-FPM no podía leerlos.
