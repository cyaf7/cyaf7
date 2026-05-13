# Traefik como reverse proxy

### Qué es Traefik

Traefik es un reverse proxy y balanceador de carga de código abierto diseñado específicamente para entornos de contenedores e infraestructuras dinámicas. A diferencia de soluciones tradicionales como Nginx, Traefik descubre automáticamente los servicios disponibles en el entorno y configura el enrutamiento de forma dinámica, sin necesidad de recargar la configuración manualmente cada vez que un servicio cambia.

Un reverse proxy actúa como intermediario entre los clientes externos y los servicios internos. Cuando una petición llega desde internet, Traefik la recibe, analiza a qué servicio va dirigida en función del dominio o la ruta, y la reenvía al backend correspondiente. El cliente nunca se comunica directamente con el servicio interno, lo que permite centralizar el control de acceso, el cifrado TLS y el enrutamiento en un único punto de entrada.

### Por qué se eligió Traefik en el proyecto

En el contexto del proyecto Spotly, la necesidad de un reverse proxy surge de la arquitectura de red: las VMs de aplicación residen en la red `ovn-app`, completamente aisladas del exterior, y solo la VM `dmz-receiver` en `ovn-dmz` tiene exposición al tráfico entrante. Traefik resuelve el problema de cómo distribuir ese tráfico hacia los servicios internos de forma controlada y segura.

Se eligió Traefik frente a alternativas como Nginx por varias razones concretas. La primera es su integración nativa con entornos de contenedores, lo que simplifica la configuración en un cluster LXD como el del proyecto. La segunda es la capacidad de enrutar tráfico hacia múltiples servicios desde un único punto de entrada utilizando reglas basadas en rutas o subdominios, sin necesidad de gestionar bloques de configuración individuales por cada servicio. La tercera es la flexibilidad para modificar el enrutamiento sin interrumpir el servicio, ya que Traefik aplica los cambios en caliente.

Adicionalmente, Traefik permite manipular las cabeceras HTTP de las respuestas antes de que lleguen al cliente, lo que resulta relevante para la aplicación de políticas de seguridad como Content Security Policy. Si en el futuro el backend se migra a otro servidor o se añaden nuevos servicios, basta con modificar la configuración de Traefik sin necesidad de reconfigurar Cloudflare ni el resto de la infraestructura.

### Posición en la arquitectura

Traefik se despliega en la VM `dmz-receiver`, que actúa como el único punto de entrada público del sistema. Esta VM tiene dos interfaces de red: `eth0` en `ovn-dmz`, que recibe el tráfico proveniente del túnel de Cloudflare, y `eth1` en `ovn-app` con IP estática `10.10.20.6`, que le permite comunicarse directamente con los servicios internos de la capa de aplicación.

El flujo de tráfico es el siguiente. Las peticiones externas llegan a Cloudflare a través del dominio `app.spotly.cat`. Cloudflare las reenvía por el túnel `cloudflared` activo en `dmz-receiver`. Una vez dentro de la VM, Traefik recibe las peticiones y las enruta hacia el servicio correspondiente: las peticiones a la API se dirigen hacia `app-core` en `10.10.20.3:8000`, y las peticiones al frontend se sirven desde Nginx en el propio `app-core` en el puerto 3000. Todo este enrutamiento ocurre dentro de la red `ovn-app`, sin que el tráfico externo tenga acceso directo a ninguno de esos servicios.

### Relación con Cloudflare y el túnel

Traefik y Cloudflare cumplen funciones complementarias pero distintas. Cloudflare gestiona el DNS, termina el TLS externo y protege el sistema ante ataques de denegación de servicio mediante su red de distribución. El túnel `cloudflared`establece una conexión saliente cifrada desde `dmz-receiver` hacia los servidores de Cloudflare, lo que permite recibir tráfico público sin necesidad de exponer ningún puerto al exterior ni de disponer de una IP pública fija.

Traefik, por su parte, opera en la capa interna: recibe las peticiones que llegan por el túnel y decide a qué servicio reenviarlas. Esta separación de responsabilidades es deliberada. Cloudflare cubre la exposición pública y la protección perimetral; Traefik cubre el enrutamiento interno y la separación de servicios. El resultado es que ningún servicio de la capa de aplicación está directamente accesible desde el exterior, y cualquier cambio en la distribución interna del tráfico se gestiona exclusivamente en Traefik sin afectar a la configuración de Cloudflare.

### Distinción con Nginx

En el proyecto coexisten Traefik y Nginx, pero con funciones completamente distintas que no se solapan. Traefik actúa como reverse proxy en la VM `dmz-receiver`: recibe el tráfico entrante del túnel de Cloudflare y lo distribuye hacia los servicios internos según la ruta solicitada. Nginx, por su parte, está desplegado en la VM `app-core` y cumple exclusivamente la función de servidor de ficheros estáticos: sirve el `frontend-user` y el `frontend-admin` en el puerto 3000, respondiendo directamente a las peticiones que Traefik le reenvía.

La diferencia conceptual es la siguiente: Traefik decide a dónde va cada petición; Nginx sirve el contenido cuando la petición ya ha llegado a su destino. Nginx en este proyecto no actúa como reverse proxy ni gestiona enrutamiento — esa responsabilidad recae íntegramente en Traefik.

### Estado actual

Traefik está desplegado y activo en la VM `dmz-receiver`. El enrutamiento hacia `app-core` y el frontend de usuario está operativo. El flujo completo desde `app.spotly.cat` hasta los servicios internos funciona a través del túnel `cloudflared` con dominio verificado en Cloudflare Zero Trust.
