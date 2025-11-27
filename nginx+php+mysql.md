# NGINX+PHP+MYSQL

***

## **1. Configuración de red y acceso a la VM**

#### **Ver IP de la máquina**

```
ip a
```

Permite conocer la dirección IP asignada. Necesario para pruebas con navegador o curl.

#### **Habilitar SSH en Debian**

```
sudo apt install openssh-server
sudo systemctl enable ssh
sudo systemctl start ssh
```

Activa el servicio SSH para conectar desde el equipo anfitrión.

***

## **2. Instalación de Nginx**

#### **Instalar Nginx**

```
sudo apt update
sudo apt install nginx
```

Instala el servidor web que servirá contenido estático y que enviará las peticiones PHP a PHP-FPM.

#### **Iniciar y verificar Nginx**

```
sudo systemctl start nginx
sudo systemctl status nginx
```

Comprueba que el servicio esté activo.

#### **Probar Nginx**

```
curl http://localhost
```

Debe devolver la página por defecto de Nginx.

***

## **3. Estructura de sitios web**

#### **Crear directorio del sitio**

```
sudo mkdir -p /var/www/camilly.com
```

#### **Asignar permisos correctos**

```
sudo chown -R www-data:www-data /var/www/camilly.com
sudo chmod -R 755 /var/www/camilly.com
```

Es fundamental: si el usuario del servicio web no puede leer los archivos, Nginx devuelve error 500.

***

## **4. Configuración del Virtual Host**

#### **Crear archivo de configuración**

**Puedes simplesmente copiar el con sudo cp /**&#x65;tc/nginx/sites-available/default  **/**&#x65;tc/nginx/sites-available/**camilly.com.conf**

```
sudo nano /etc/nginx/sites-available/camilly.com.conf
```

Contenido final funcional:

```
server {
    listen 80;
    root /var/www/camilly.com;
    index index.php index.html;
    server_name camilly.com;

    location / {
        try_files $uri $uri/ =404;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php8.4-fpm.sock;
    }

    access_log /var/log/nginx/camilly.access.log;
    error_log  /var/log/nginx/camilly.error.log;
}
```

#### **Activar el sitio**

```
sudo ln -s /etc/nginx/sites-available/camilly.com.conf /etc/nginx/sites-enabled/
```

#### **Probar configuración**

```
sudo nginx -t
```

Comprueba errores sintácticos.

#### **Recargar servicio**

```
sudo systemctl reload nginx
```

***

## **5. Instalación y configuración de PHP-FPM**

#### **Instalar PHP y extensiones**

```
sudo apt install php php-fpm php-mysql
```

`php-mysql` es fundamental: sin él aparece el error\
&#xNAN;**"undefined function mysqli\_connect()"**.

#### **Verificar que PHP-FPM esté activo**

```
sudo systemctl status php8.4-fpm
```

#### **Ubicación del socket PHP**

```
ls /run/php/
```

Nos mostró:\
`php8.4-fpm.sock` → este es el que debe ir en `fastcgi_pass`.

***

## **6. Pruebas PHP**

#### **Crear archivo phpinfo()**

```
sudo nano /var/www/camilly.com/info.php
```

Contenido:

```
<?php phpinfo();
```

#### **Probar**

```
curl http://localhost:8081/info.php
```

***

## **7. Base de Datos: MariaDB/MySQL**

#### **Instalar MariaDB**

```
sudo apt install mariadb-server
```

#### **Acceder al cliente**

```
sudo mysql
```

#### **Crear base de datos**

```
CREATE DATABASE camillydb CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

#### **Crear usuario**

```
CREATE USER 'camillyuser'@'localhost' IDENTIFIED BY 'Camilly';
```

#### **Dar permisos**

```
GRANT ALL PRIVILEGES ON camillydb.* TO 'camillyuser'@'localhost';
FLUSH PRIVILEGES;
```

#### **Usar la base**

```
USE camillydb;
```

#### **Crear tabla**

```
CREATE TABLE users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    nombre VARCHAR(100),
    email VARCHAR(100),
    password VARCHAR(100)
);
```

#### **Insertar datos**

```
INSERT INTO users (nombre, email, password) VALUES
('Ivan', 'ivan@figuera', '1234'),
('Yadir', 'Yadir@yabar', 'abcd');
```

#### **Listar**

```
SELECT * FROM users;
```

***

## **8. Scripts PHP del proyecto**

#### **conexion.php**

```
<?php
$host = "localhost";
$user = "camillyuser";
$pass = "Camilly";
$db   = "camillydb";

$conn = mysqli_connect($host,$user,$pass,$db);

if(!$conn){
    die("Error de conexión: " . mysqli_connect_error());
}

echo "Conexión OK";
```

#### **usuarios.php**

```
<?php
require 'conexion.php';

$result = mysqli_query($conn, "SELECT * FROM users");

while($fila = mysqli_fetch_assoc($result)){
    echo $fila['id']." - ".$fila['nombre']." - ".$fila['email']."<br>";
}
```

#### **index.php**

```
<?php
require 'conexion.php';
echo "<h1>Bienvenido a Camilly</h1>";
echo "<p>Conexión con la base de datos OK.</p>";
```

***

## **9. Logs y depuración**

#### **Ver logs del sitio**

```
sudo tail -20 /var/log/nginx/camilly.error.log
```

#### **Ver errores globales**

```
sudo tail -20 /var/log/nginx/error.log
```

#### **Ver estado de PHP-FPM**

```
sudo journalctl -u php8.4-fpm -n 20
```

***

## **10. Principales problemas que hemos resolvimos y por qué**

#### **Error 500**

Causado por:

* socket PHP-FPM incorrecto
* permisos incorrectos
* fastcgi mal configurado

#### **Puerto 80 en conflicto (bind() failed)**

Había varios sitios escuchando en el mismo puerto sin configuración adecuada.

#### **mysqli\_connect() no existía**

Faltaba instalar `php-mysql`.

#### **Archivos propiedad de root**

PHP-FPM no podía leerlos.

***



***

