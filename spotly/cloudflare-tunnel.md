# Cloudflare Tunnel

### Introducción

Cloudflare Tunnel es un servicio que establece un túnel de salida desde la infraestructura de Spotly hacia los servidores de Cloudflare sin requerir puertos abiertos en el firewall OPNsense ni una dirección IP pública fija. El usuario externo accede a la aplicación Spotly mediante un dominio DNS (app.spotly.cat) que apunta a la red de Cloudflare, que entonces enruta el tráfico a través del túnel hacia dmz-receiver. Este enfoque permite exponer servicios internos sin exponerse a escaneos de puertos ni a intentos de acceso directo a OPNsense.

A diferencia de WireGuard, que crea una conexión VPN privada entre el usuario y la red interna, Cloudflare Tunnel no es una VPN: es un proxy inverso remoto controlado por Cloudflare. El usuario accede a través de la red pública de Cloudflare, que actúa como intermediario de confianza. Para aplicaciones web (HTTP/HTTPS), este modelo es más seguro y requiere menos configuración que gestionar certificados y acceso VPN.

### Registro del Dominio y Delegación de DNS

El dominio spotly.cat fue registrado a través de Nominalia, un registrador de dominios que ofrece registro de dominios genéricos a coste cero para usuarios en contextos educativos. Nominalia es el registrador (la entidad que administra la compra y renovación del dominio), pero no es el proveedor DNS.

Para que Cloudflare pueda gestionar todas las consultas DNS de spotly.cat (es decir, resolver spotly.cat, app.spotly.cat, etc.), es necesario cambiar los nameservers en el panel de control de Nominalia. Este cambio dice al sistema DNS global: "las consultas sobre spotly.cat no las responda Nominalia, respóndelas Cloudflare". Sin este cambio, aunque Cloudflare estuviera configurado, las consultas DNS seguirían llegando a los servidores de Nominalia, que no sabrían cómo responder.

Los nameservers de Cloudflare asignados son arturo.ns.cloudflare.com y daphne.ns.cloudflare.com. Estos nameservers están registrados en los servidores raíz de ICANN y permiten que Cloudflare resuelva cualquier consulta sobre spotly.cat y todos sus subdominios. Una vez que se cambian los nameservers en Nominalia (desde su panel de control, sección "Gestión DNS" o similar), el cambio se propaga globalmente en ICANN en 24 a 48 horas. En la práctica, el cambio es efectivo en minutos.

### Creación de la Cuenta Cloudflare y Configuración del Plan

La cuenta Cloudflare utiliza la dirección de correo electrónico spotly.cci@gmail.com, compartida entre los miembros del equipo. La cuenta está registrada bajo el equipo spottly (con dos 't') en el dashboard de Cloudflare. El plan de servicio activo es el plan Free de Cloudflare, que incluye:

* DNS gestionado para dominio.
* Un túnel Cloudflare Tunnel.
* SSL/TLS automático (certificados Let's Encrypt renovados automáticamente).
* Firewall básico con reglas personalizadas.
* Acceso a Cloudflare Zero Trust (necesario para gestionar rutas del túnel).

El plan Free tiene limitaciones: no permite CNAME personalizado hacia otros proveedores (la resolución DNS debe apuntar directamente a Cloudflare), y el acceso Zero Trust es limitado. Estas limitaciones no afectan a este proyecto.

### Cloudflare Zero Trust — Gestión de Rutas

Zero Trust es la plataforma de Cloudflare para control de acceso y enrutamiento avanzado. Para Spotly, Zero Trust se utiliza exclusivamente para definir y gestionar las rutas que el túnel puede servir. Una ruta en Zero Trust es una regla que dice: "cuando una solicitud llegue a este dominio, enrútala a este destino a través del túnel".

Inicialmente, se intentó gestionar las rutas desde el archivo de configuración cloudflared (`/etc/cloudflared/config.yml`). Sin embargo, cuando se instala cloudflared con el método `--token` (recomendado para servidores sin interacción humana), el daemon ignora el archivo de configuración y obtiene las rutas exclusivamente del dashboard de Zero Trust. Las rutas deben configurarse a través de la interfaz web.

El token de autenticación del túnel (un string criptográfico de \~100 caracteres) se proporciona durante la primera ejecución interactiva de `cloudflared tunnel create` y se almacena en `/root/.cloudflared/<tunnel-id>.json`. Este archivo es crítico para que cloudflared pueda autenticarse con Cloudflare.

### Configuración del Túnel — ID y Hostname

El túnel de Spotly tiene el ID: `ecdb38db-fc83-4bda-a7a1-c2eee99e8da9` y el nombre interno: `spotly-tunnel`. El hostname registrado en Cloudflare es `app.spotly.cat`. Cuando un usuario navega a `https://app.spotly.cat`, la consulta DNS se resuelve en los nameservers de Cloudflare, que responden con la dirección IP de los servidores de Cloudflare. El tráfico HTTP/HTTPS entonces se enruta a través del túnel hacia dmz-receiver.

El estado del túnel se verifica en el dashboard: Status: Healthy indica que el daemon cloudflared en dmz-receiver está conectado y activo. Un estado Down indica desconexión: cloudflared no pudo establecer conexión con Cloudflare, probablemente por falta de acceso a internet desde dmz-receiver.

### Rutas y Destinos — Configuración del Dashboard

Las rutas están configuradas en Zero Trust bajo la sección Tunnels, en la pestaña Public Hostname. Cada ruta especifica un patrón de dominio/ruta y un destino local. Para Spotly:

* `app.spotly.cat/*` → `http://localhost:80` (Traefik en dmz-receiver escucha en el puerto 80, recibe tráfico HTTP y lo reencamina a HTTPS)
* `app.spotly.cat/api/*` → `http://localhost:8000` (FastAPI en app-core, pero Traefik es el intermediario, no se configura directo)

En realidad, la configuración correcta es que todas las rutas apunten a Traefik (`http://localhost:80`), y Traefik internamente enruta según el path y el hostname:

```
app.spotly.cat/ → Traefik → decisión de enrutamiento interno → frontend-user:3000
app.spotly.cat/api/* → Traefik → fastapi:8000
app.spotly.cat/admin* → Traefik → localhost:8081 (Traefik dashboard)
app.spotly.cat/stream* → Traefik → frame_buffer WebSocket (app-core)
```

### Traefik en dmz-receiver — Reverse Proxy

Traefik en dmz-receiver escucha en 0.0.0.0:80 (HTTP) y reencamina automáticamente a HTTPS usando el certificado SSL generado por Cloudflare. Las reglas de enrutamiento en Traefik se definen en su archivo de configuración o a través de labels de Docker Compose.

En la configuración de Spotly, Traefik está configurado para reconocer headers HTTP específicos (X-Forwarded-For, X-Forwarded-Proto) que Cloudflare inserta en el tráfico. Esto permite que Traefik sepa el IP original del usuario y el protocolo utilizado antes del proxy.

Las respuestas de los servicios internos (FastAPI, Nginx) son interceptadas por Traefik, que inserta headers de respuesta adicionales (CORS, seguridad) antes de devolver al cliente a través de Cloudflare.

### SSL/TLS — Certificados y Encriptación

Cloudflare emite automáticamente un certificado SSL para app.spotly.cat usando Let's Encrypt. El certificado se renueva cada 90 días automáticamente sin intervención. El cliente (navegador o aplicación móvil) establece una conexión TLS/SSL hacia Cloudflare (servidor-a-cliente), que es el punto visible. Internamente, entre Cloudflare y cloudflared (en dmz-receiver), la conexión es un túnel QUIC (Quick UDP Internet Connection), que también está encriptado pero es un protocolo propietario de Cloudflare.

Entre cloudflared y Traefik (localhost:80), la comunicación es HTTP sin encriptación, ya que es tráfico local dentro de la misma máquina virtual. Este es un escenario seguro porque:

1. Traefik escucha exclusivamente en 127.0.0.1 (loopback) o en la interfaz de red interna de la VM.
2. No hay acceso de red desde otras máquinas a ese tráfico HTTP.

Si en el futuro el tráfico entre cloudflared y Traefik debiera encriptarse (por ejemplo, si estuvieran en máquinas separadas), se configurarían certificados mutuos (mTLS) y ambos componentes usarían HTTPS.

### Flujo Completo de una Solicitud

1. Cliente (navegador): solicita `https://app.spotly.cat/api/spots`.
2. Resolución DNS: se consulta `app.spotly.cat` a los nameservers de Cloudflare (arturo/daphne).
3. Cloudflare DNS: devuelve una dirección IP de Cloudflare (perteneciente a su red CDN).
4. Cliente TLS: el navegador establece una conexión TLS/SSL con Cloudflare y envía el HTTP/2 request.
5. Cloudflare Zero Trust: verifica que existe una ruta pública para `app.spotly.cat` y enruta a través del túnel.
6. Túnel QUIC: Cloudflare envía la solicitud encriptada a través del túnel QUIC hacia cloudflared en dmz-receiver.
7. cloudflared: desencripta y reenvía a Traefik en `http://localhost:80`.
8. Traefik: analiza el método HTTP (GET), la ruta (`/api/spots`), el hostname (`app.spotly.cat`) y reenvía a FastAPI en app-core:8000.
9. FastAPI: procesa la solicitud, consulta PostgreSQL en data, genera una respuesta JSON.
10. Respuesta inversa: JSON → Traefik → cloudflared → Cloudflare → Cliente (HTTP/2 sobre TLS).

### Errores Encontrados y Resoluciones

#### CNAME Parcial no Disponible en Plan Free

Inicialmente, se intentó configurar un CNAME personalizado desde spotly.cat hacia un servicio externo de Cloudflare, manteniendo el resto del DNS en otro registrador. Cloudflare no permite esto en el plan Free: requiere transferencia completa de los nameservers. La solución fue transferir completamente los nameservers a Cloudflare, como se describe en la sección anterior.

#### cloudflared Ignora config.yml Cuando se Instala con --token

Cuando cloudflared se instala usando `cloudflared tunnel create --token <token-string>`, el daemon no lee el archivo `/etc/cloudflared/config.yml`. Las rutas deben configurarse exclusivamente a través del dashboard de Cloudflare Zero Trust. Inicialmente, se intentó definir rutas en config.yml, pero fueron ignoradas completamente.

La solución fue acceder al dashboard de Cloudflare, navegar a Tunnels, seleccionar spotly-tunnel, y crear las rutas públicas manualmente en la interfaz web. Una vez hecho esto, cloudflared lee las rutas del servidor de Cloudflare cada vez que se inicia o periódicamente mientras se ejecuta.

#### Cloudflared No Inicia Sin Acceso a Internet

Si dmz-receiver no tiene acceso a internet (por ejemplo, si las reglas de firewall en OPNsense bloquean todo tráfico saliente), cloudflared no puede autenticarse con Cloudflare y falla al iniciar. Los logs muestran errores como "Failed to dial tunnel authority" o "Connection timeout".

La solución es verificar que OPNsense permite tráfico saliente desde ovn-dmz (10.10.30.0/24) hacia internet. En la práctica, OPNsense tiene una regla implícita que permite tráfico de salida desde todas las VLANs, por lo que esto no debería ser un problema a menos que se añadan reglas explícitas más restrictivas.

### Estado Actual

Cloudflare Tunnel está completamente operacional. El daemon cloudflared en dmz-receiver está RUNNING, el estado del túnel en el dashboard es Healthy, y las solicitudes a app.spotly.cat se enrutan correctamente a través del túnel hacia Traefik y los servicios internos.

El tráfico desde la cámara en el MacBook de Ivan hacia la aplicación (PATCH app.spotly.cat/api/spots/{celda}) fluye correctamente a través de: MacBook → 4G hotspot → Cloudflare → cloudflared (dmz-receiver) → Traefik → FastAPI (app-core) → PostgreSQL (data).

Las respuestas en tiempo real vuelven por el mismo camino, y los clientes móviles reciben eventos en tiempo real a través del WebSocket en ovn-app.

### Próximos Pasos

WireGuard en OPNsense está pendiente de implementación. Una vez configurado, permitirá acceso VPN privado a la red de management (VLAN 20) para administración SSH de los nodos, complementando (no reemplazando) Cloudflare Tunnel. Cloudflare Tunnel seguirá siendo el punto de entrada público para usuarios finales y aplicaciones móviles.
