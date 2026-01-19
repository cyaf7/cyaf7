# Docker compose

El objetivo de esta práctica es desplegar una aplicación web completa mediante Docker Compose, separando cada componente en un servicio independiente (Nginx como servidor web, PHP-FPM como intérprete de PHP, MySQL como base de datos y phpMyAdmin como panel de administración). Con Compose se consigue un despliegue reproducible y más fácil de gestionar, ya que toda la infraestructura queda definida en un único archivo YAML.



#### ¿Qué son los contenedores de Docker?

Los contenedores Docker son entornos aislados que ejecutan aplicaciones usando el kernel del sistema operativo del host. Incluyen el software necesario (dependencias, librerías y configuración), pero consumen menos recursos que una máquina virtual porque no necesitan un sistema operativo completo.

#### ¿Qué diferencias hay entre los contenedores de Docker y los LXC?

Ambos usan aislamiento a nivel de sistema (namespaces y cgroups), pero Docker está orientado a empaquetar y distribuir aplicaciones mediante imágenes versionadas y registro (Docker Hub). LXC está más orientado a “contenedores de sistema”, similares a una mini máquina Linux, gestionando un entorno más completo. En general, Docker simplifica el despliegue de aplicaciones con un flujo más automatizado (build, push, run).

#### ¿Cuál es la diferencia entre una imagen y un contenedor en Docker?

Una imagen es una plantilla inmutable con el sistema de archivos y la configuración base de una aplicación. Un contenedor es la instancia en ejecución creada a partir de una imagen, con una capa de escritura propia donde se generan cambios temporales mientras el contenedor existe.

#### ¿Qué sucede con los datos cuando un contenedor se elimina?

Los datos guardados en la capa de escritura del contenedor se pierden al eliminarlo. Para mantener datos persistentes se deben usar volúmenes (managed volumes) o bind mounts (carpetas del host montadas en el contenedor).

#### ¿Cuáles son las ventajas de utilizar contenedores Docker?

Permiten despliegues rápidos, portabilidad entre entornos, aislamiento entre servicios, menor consumo de recursos que máquinas virtuales y facilidad para reproducir entornos completos con Docker Compose. Además, mejoran el control de versiones y facilitan pruebas, desarrollo y despliegue.

#### ¿Qué tipo de aplicaciones y servicios se pueden desplegar con Docker?

Se pueden desplegar aplicaciones web (PHP, Node, Python, Java), bases de datos (MySQL, PostgreSQL), servidores web (Nginx, Apache), herramientas de administración (phpMyAdmin), servicios de red (DNS, proxy, VPN), monitorización (Prometheus, Grafana) y plataformas como WordPress, Nextcloud o GitLab.

#### ¿Qué otros tipos de contenedores existen además de Docker?

Existen LXC/LXD, Podman (compatible con Docker pero sin daemon), rkt (descontinuado), y tecnologías de contenedores gestionadas por Kubernetes/CRI (por ejemplo containerd y CRI-O). También existen contenedores a nivel de sistema usados en virtualización ligera.

### INCIDENCIAS EN DOCKER COMPOSE y solución aplicada

1. No se disponía inicialmente del ZIP oficial del ejercicio Situación: al principio no estaba el proyecto oficial (LoginRegister) en formato ZIP, por lo que se hizo una primera aproximación de despliegue con un ejemplo de PHP + MySQL para entender la lógica general. Solución: al recibir el ZIP, se repitió el despliegue siguiendo estrictamente la estructura del ejercicio, montando el proyecto real desde el host y usando configuración de Nginx.
2. Portainer inaccesible por olvido de credenciales Situación: no se recordaba el usuario/contraseña del panel, lo que impedía gestionar stacks. Solución: se eliminó el contenedor y el volumen portainerdata para reiniciar la instalación y crear un nuevo usuario administrador.
3. Nginx no arrancaba (Exited 1) por “host not found in upstream” Situación: el contenedor Nginx fallaba porque en la configuración se usaba un upstream con nombre incorrecto (`phpfpm`), que no coincidía con el nombre real del servicio en Compose. Solución: se cambió `fastcgi_pass phpfpm:9000;` por `fastcgi_pass app:9000;`, usando el nombre del servicio PHP dentro de la red interna.
4. Error 404 Not Found aunque el contenedor Nginx estaba activo Situación: la web respondía pero devolvía 404 debido a configuración duplicada (dos bloques server) y/o un root apuntando a otra ruta. Solución: se dejó un único bloque server, se marcó como default\_server y se fijó el root correcto: /var/www/loginregister.
5. Conflictos por puertos y despliegues en paralelo Situación: al querer repetir el ejercicio sin borrar el stack anterior, era necesario evitar colisiones de puertos y nombres. Solución: se creó un segundo despliegue “LoginRegister2” con stack independiente (loginregister2) y puertos nuevos (por ejemplo 90 y 8090), manteniendo ambos entornos funcionales.

Preparación del proyecto

* Crear carpeta de trabajo y mover ZIP&#x20;
* mkdir -p \~/test-docker-compose
* &#x20;mv \~/Downloads/LoginRegister.zip \~/test-docker-compose/&#x20;
* cd \~/test-docker-compose&#x20;
* ls
* Extraer el ZIP&#x20;
* unzip LoginRegister.zip ls

Duplicado que he hecho para hacer el ejercicio de nuevo

* Copiar proyecto a LoginRegister2 cp -a LoginRegister LoginRegister2 ls -la

Verificación de estructura para bind mounts

* Comprobar carpetas y logs
* &#x20;ls -la /home/amelia/test-compose/LoginRegister2/log&#x20;
* ls -la /home/amelia/test-compose/LoginRegister2/nginx/conf.d
* &#x20;touch /home/amelia/test-compose/LoginRegister2/log/php.log

Portainer (reinicio de credenciales)

* Borrar Portainer y reinstala
* &#x20;docker stop portainer&#x20;
* docker rm portainer&#x20;
* docker volume rm -f portainer\_data&#x20;
* docker run -d -p 9000:9000 --name portainer --restart=always -v /var/run/docker.sock:/var/run/docker.sock -v portainer\_data:/data portainer/portainer-ce
* Ver logs del contenedor
* &#x20;Nginx docker logs miAppNginx2
* Reiniciar Nginx tras cambios en la configuración&#x20;
* docker restart miAppNginx2
* Comprobar qué está cargando Nginx (para detectar dos server blocks)
* &#x20;docker exec -it miAppNginx2 sh -lc 'nginx -T 2>/dev/null | grep -n "server {"; nginx -T 2>/dev/null | grep -n "root "'

Verificación final

* Estado de contenedores docker ps
* Test HTTP desde terminal curl -I [http://localhost:90](http://localhost:90)

### Con mi aplicacion (paso a paso)

#### 1) Descomprimir el ZIP en la carpeta del proyecto

```bash
unzip ~/Downloads/clone.zip -d ~/test-compose/LoginRegister2
```

Esto extrae el proyecto dentro del directorio que Docker Compose monta en los contenedores, de forma que Nginx pueda servir los ficheros directamente.

#### 2) Comprobar que se ha extraído correctamente

```bash
ls -la ~/test-compose/LoginRegister2
ls -la ~/test-compose/LoginRegister2/clone
```

Aquí verifico que existen los PHP principales (`index.php`, `config.php`, etc.) y las carpetas de recursos (`partials`, `img`, etc.).

#### 3) Corregir la conexión a MySQL en la aplicación (evitar `localhost`)

En Docker, `localhost` apunta al propio contenedor, no al contenedor de MySQL. Por eso el host debe ser el nombre del servicio en el compose (normalmente `db`).

Editar el fichero:

```bash
nano ~/test-compose/LoginRegister2/clone/config.php
```

Cambiar a:

```php
$host = 'db';
$dbname = 'CasaDelLibro';
$username = 'root';
$password = '1234';
```

Si también se usa el login/register fuera de `clone/`, revisar igualmente:

```bash
nano ~/test-compose/LoginRegister2/conexion.php
```

y asegurar:

```php
$servername="db";
$username="root";
$password="1234";
```

#### 4) Preparar importación automática de la base de datos

Crear carpeta de inicialización y copiar el SQL:

```bash
mkdir -p ~/test-compose/LoginRegister2/db
cp ~/test-compose/LoginRegister2/clone/CasaLibroClon/CasaLibroUpdated.sql ~/test-compose/LoginRegister2/db/init.sql
ls -la ~/test-compose/LoginRegister2/db
```

#### 5) Montar el SQL en MySQL desde `docker-compose.yml`

Editar el compose:

```bash
nano ~/test-compose/LoginRegister2/docker-compose.yml
```

Dentro del servicio `db:`, añadir:

```yaml
volumes:
  - ./db:/docker-entrypoint-initdb.d
```

MySQL ejecuta automáticamente los `.sql` de esa carpeta durante la primera inicialización.

#### 6) Levantar los contenedores

```bash
cd ~/test-compose/LoginRegister2
docker compose up -d
```

Comprobar estado:

```bash
docker ps
```

#### 7) Incidencia: error “Table already exists” al ejecutar el SQL

Si MySQL no arranca y en logs aparece un error tipo:\
`ERROR 1050: Table 'Libro_Autor' already exists`

Ver logs:

```bash
docker ps -a --format "table {{.Names}}\t{{.Image}}\t{{.Status}}"
docker logs NOMBRE_DEL_MYSQL --tail 120
```

Solución rápida aplicada: editar el SQL y eliminar el `CREATE TABLE` duplicado:

```bash
nano ~/test-compose/LoginRegister2/db/init.sql
```

Reiniciar MySQL:

```bash
docker restart NOMBRE_DEL_MYSQL
```

#### 8) Verificar la base de datos en phpMyAdmin

Abrir:\
[http://localhost:8090](http://localhost:8090/)

Conectar con:

* Servidor: `db`
* Usuario: `root`
* Contraseña: `1234`

Aquí confirmo que existe `CasaDelLibro` y que sus tablas están creadas.

#### 9) Incidencia: “could not find driver” al abrir `/clone/index.php`

Este error aparece porque la imagen estándar `php:8-fpm` no trae instalados por defecto los drivers de MySQL para PHP (`pdo_mysql` / `mysqli`).

#### 10) Solución: crear una imagen PHP propia con drivers MySQL (Dockerfile + build)

Crear Dockerfile:

```bash
nano ~/test-compose/LoginRegister2/Dockerfile
```

Contenido:

```dockerfile
FROM php:8-fpm
RUN docker-php-ext-install pdo_mysql mysqli
```

Editar el servicio `app` en `docker-compose.yml` para usar `build` en lugar de `image`:

```yaml
app:
  build: .
```

Con `build: .` Docker construye una imagen nueva usando el Dockerfile del proyecto, incluyendo los módulos necesarios.

Reconstruir y levantar:

```bash
cd ~/test-compose/LoginRegister2
docker compose down
docker compose up -d --build
```

Comprobar que los módulos están activos:

```bash
docker exec -it NOMBRE_DEL_PHP php -m | grep -E "pdo_mysql|mysqli"
```

#### 11) Acceso final a la aplicación

* Sitio principal: [http://localhost:90/](http://localhost:90/)
* Aplicación del ZIP: [http://localhost:90/clone/index.php](http://localhost:90/clone/index.php)

Para ver rápidamente todas las páginas PHP disponibles:

```bash
cd ~/test-compose/LoginRegister2
find . -maxdepth 2 -name "*.php" | sed 's|^\./||'
```

Incidencias encontradas y cómo las resolvimos

1. La aplicación no conectaba con MySQL usando “localhost” Al principio la conexión fallaba porque en Docker “localhost” no apunta a la base de datos, sino al propio contenedor. Lo arreglamos cambiando el host de conexión a “db”, que es el nombre del servicio MySQL dentro del docker-compose.
2. MySQL no arrancaba al importar la base de datos automáticamente Cuando MySQL intentaba ejecutar el archivo init.sql, se paraba porque una tabla ya existía (table already exists). Para solucionarlo, editamos el init.sql y eliminamos el CREATE TABLE que estaba duplicado, así MySQL pudo terminar la importación.
3. Error “could not find driver” en páginas PHP con base de datos La imagen de PHP que usamos (php:8-fpm) no traía instalados los drivers para MySQL. Por eso salía el error. Lo solucionamos creando un Dockerfile propio e indicando build en el servicio PHP, para construir una imagen con pdo\_mysql y mysqli.
4. Comprobación final del despliegue Después de corregir todo, verificamos que los contenedores estaban en ejecución con docker ps, entramos a la web en el puerto 90 y confirmamos la base de datos desde phpMyAdmin en el puerto 8090 conectando al servidor “db”.
