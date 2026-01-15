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

