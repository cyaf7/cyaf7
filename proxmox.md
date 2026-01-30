# Proxmox

Este trabajo fue realizado en clase por los alumnos **Camilly Aoki** e **Iván Figueroa**. La práctica consistió en la instalación de Proxmox en un equipo físico mediante un dispositivo USB, utilizando el instalador oficial de Proxmox junto con el entorno de arranque incluido en la imagen de instalación, el cual es esencial para la correcta puesta en funcionamiento del sistema.

#### Proxmox y su aplicación en entornos profesionales:

**Proxmox** es una plataforma de virtualización de código abierto que permite administrar infraestructuras virtualizadas desde una única interfaz web. Proporciona soporte tanto para máquinas virtuales completas como para contenedores LXC, lo que la convierte en una solución flexible y ampliamente utilizada en entornos educativos y empresariales.

En el ámbito profesional, Proxmox se emplea habitualmente para la consolidación de servidores, el despliegue de firewalls virtuales, la segmentación de redes, la provisión de servicios internos y la creación de entornos de pruebas.

{% hint style="info" %}
Proxmox permite combinar máquinas virtuales y contenedores dentro de una misma infraestructura, optimizando el uso de recursos según las necesidades del servicio
{% endhint %}

#### Ventajas&#xD;

* Gestión centralizada mediante interfaz web.
* Alto rendimiento y eficiencia en el uso de recursos.
* Flexibilidad avanzada en la configuración de redes.
* Integración nativa con contenedores ligeros.

#### Desventajas

* Curva de aprendizaje inicial elevada, especialmente en aspectos de red.
* Algunas configuraciones requieren ajustes manuales avanzados.
* Una mala comprensión de la arquitectura puede provocar errores complejos de diagnosticar.



#### Máquinas virtuales y contenedores LXC

En la fase inicial del proyecto, el grupo trabajó con máquinas virtuales en Proxmox. Esta decisión se tomó debido a que todavía no se comprendía plenamente la diferencia de rendimiento y consumo de recursos entre máquinas virtuales y contenedores.

**Diferencias principales**

**Máquinas virtuales**

* Virtualizan hardware completo.
* Cada sistema ejecuta su propio kernel.
* Mayor consumo de CPU, memoria y almacenamiento.

**Contenedores LXC**

* Virtualizan a nivel de sistema operativo.
* Comparten el kernel del sistema anfitrión.
* Son más rápidos, ligeros y eficientes.

{% hint style="info" %}
Para servicios de red como firewalls o routers virtuales, los contenedores LXC resultan más adecuados debido a su menor sobrecarga y mayor rendimiento.

Una vez comprendidas estas diferencias, se decidió implementar el escenario final utilizando contenedores LXC.
{% endhint %}



#### Descripción general del entorno

El entorno se implementó sobre Proxmox utilizando dos contenedores LXC con Alpine Linux, una distribución minimalista orientada a la seguridad y al rendimiento.

**Contenedores desplegados**

**Contenedor Firewall** Actúa como router, firewall y dispositivo NAT. Proporciona conectividad a Internet a la red interna y gestiona la asignación de direcciones IP.

**Contenedor Cliente** Forma parte de la red interna y depende completamente del Firewall para acceder a Internet.

Topología de red y conectividad

#### Firewall

* Interfaz WAN (eth0) Conectada al router del aula. Obtiene dirección IP mediante DHCP.
* Interfaz LAN interna (eth1) Red privada gestionada por el propio Firewall.

#### Cliente

* Conectado únicamente a la red interna.
* Todo su tráfico pasa obligatoriamente por el Firewall.

El Firewall actúa como punto central de la red, gestionando el enrutamiento, la traducción de direcciones y el acceso a Internet.



#### Direccionamiento IP

#### Firewall

* Interfaz WAN con IP dinámica obtenida por DHCP en el rango 192.168.109.0/24.
* Interfaz interna con IP estática 10.10.10.10.

#### Cliente

* Dirección IP asignada dinámicamente por dnsmasq.
* IP reservada 10.10.10.58.

El Firewall funciona como puerta de enlace lógica para toda la red interna.



#### Gestión de DHCP en Alpine Linux

En este entorno no se utilizó **dhclient** para la red interna. dhclient es un cliente DHCP cuya función es solicitar una dirección IP a un servidor externo, por lo que no resulta adecuado cuando el sistema debe asignar direcciones a otros equipos.

Para este escenario se utilizó **dnsmasq**, que actúa como servidor DHCP y DNS ligero.

{% hint style="info" %}
apk add dnsmasq #para descargar dnsmasq em alpine linux
{% endhint %}

#### Justificación del uso de dnsmasq

* Permite asignar direcciones IP a clientes.
* Funciona de forma estable en Alpine Linux.
* Consume pocos recursos.
* Se integra correctamente con OpenRC y contenedores LXC.

{% hint style="info" %}
dnsmasq es una solución habitual en routers, firewalls y laboratorios debido a su simplicidad y fiabilidad.
{% endhint %}



#### Configuración de dnsmasq

Las configuraciones de dnsmaq pueden ser encontradas em ese website: [https://www.ochobitshacenunbyte.com/2024/11/25/dnsmasq-configuracion-de-dns-y-dhcp-en-linux/](https://www.ochobitshacenunbyte.com/2024/11/25/dnsmasq-configuracion-de-dns-y-dhcp-en-linux/)

Archivo de configuración  /etc/dnsmasq.conf:

{% hint style="info" %}


interface=enps08 #Indica que dnsmasq solo escucha en la interfaz interna

bind-interfaces #Fuerza a dnsmasq a enlazarse únicamente a la interfaz indicada

dhcp-range=10.10.10.50,10.10.10.60,255.255.255.0,12h

dhcp-host=BC:24:11:E9:9C:66,10.10.10.58

dhcp-option=3, 10.10.10.10&#x20;
{% endhint %}

**Explicación de la configuración**

Adding `domain-needed` blocks incomplete requests from leaving your network, such as _google_ instead of _google.com_. `bogus-priv` prevents non-routable private addresses from being forwarded out of your network. Using these is simply good netizenship.

**dhcp-range** Define el rango de direcciones IP dinámicas disponibles, la máscara de red y el tiempo de concesión.

**dhcp-host** Asigna una IP fija al cliente identificado por su dirección MAC, garantizando estabilidad y facilidad de diagnóstico.

dhcp-option=3\
La opción 3 en DHCP corresponde al router por defecto. Indica al cliente cuál es su gateway.

Se usa 10.10.10.10 porque el gateway del cliente debe estar en su misma red. El cliente no puede usar directamente una IP de la red 172.20.10.0/24 porque no pertenece a su segmento de red.

{% hint style="info" %}
rc-service dnsmasq restart
{% endhint %}

#### Configuración de red en Alpine Linux y particularidades

Limitaciones en /etc/network/interfaces

Durante la configuración de la red interna se detectaron comportamientos específicos de Alpine Linux:

* No fue posible definir la puerta de enlace en la interfaz interna, ya que provocaba fallos de conectividad.
* El sistema actúa como router y gestiona el enrutamiento mediante NAT y reglas de firewall, no mediante la declaración directa de gateway en la interfaz interna.

**Uso obligatorio de la máscara de red**

Al definir la red interna en la maquina server en /etc/network/interfaces, se comprobó que utilizar únicamente notación CIDR, por ejemplo:

{% hint style="info" %}
10.10.10.10/24
{% endhint %}

provocaba errores de funcionamiento.

Fue obligatorio definir explícitamente la máscara de red:

{% hint style="info" %}
netmask 255.255.255.0
{% endhint %}

**En Alpine Linux, especialmente en contenedores LXC y sin gateway definido, la notación CIDR puede no interpretarse correctamente en interfaces que actúan como redes internas.**



#### Configuración del firewall y NAT con iptables&#xD;<br>

Para permitir la comunicación entre la red interna y el exterior, el contenedor Firewall fue configurado utilizando **iptables**, que actúa como sistema de filtrado y traducción de direcciones en Linux.

Durante el proceso de aprendizaje y pruebas se aplicaron distintas reglas de forma iterativa. Esto provocó la existencia de configuraciones residuales que interferían con el funcionamiento correcto del firewall. Por este motivo, fue imprescindible partir de un estado completamente limpio antes de aplicar la configuración final.

**Primera limpieza de reglas iptables (flush inicial)**

En una primera fase se eliminaron todas las reglas existentes en las distintas tablas de iptables con el objetivo de eliminar cualquier configuración previa que pudiera afectar al comportamiento del sistema.

Se ejecutaron los siguientes comandos:

{% hint style="info" %}
iptables -F\
iptables -t nat -F\
iptables -t mangle -F
{% endhint %}

* Se eliminaron todas las reglas de la tabla filter.
* Se eliminaron todas las reglas de la tabla nat.
* Se eliminaron todas las reglas de la tabla mangle.

Este primer flush elimina únicamente las reglas, pero **no modifica las políticas por defecto** de las cadenas. Por sí solo, no garantiza un comportamiento controlado del firewall.

**Definición de políticas por defecto**

Tras la limpieza inicial, se establecieron explícitamente las políticas por defecto de las cadenas principales. Este paso fue necesario para evitar comportamientos implícitos no deseados tras el flush de reglas.

Se configuraron las siguientes políticas:

{% hint style="info" %}
iptables -P INPUT ACCEPT\
iptables -P FORWARD ACCEPT\
iptables -P OUTPUT ACCEPT
{% endhint %}

Estas políticas permitieron durante el laboratorio:

* la entrada de tráfico al Firewall,
* el reenvío de paquetes entre interfaces,
* la salida de tráfico hacia el exterior.

En un entorno productivo estas políticas no serían adecuadas. En este laboratorio se mantuvieron permisivas para facilitar el diagnóstico y centrarse en el funcionamiento del NAT y el enrutamiento.

**Segunda limpieza de reglas iptables (flush de consolidación)**

Tras definir las políticas por defecto y después de varias pruebas fallidas, fue necesario realizar **un segundo flush completo** de iptables.

Este segundo flush fue un paso crítico. Sin él, el firewall quedaba completamente abierto de forma implícita, heredando comportamientos no deseados de configuraciones anteriores.

Se volvieron a ejecutar los siguientes comandos de flush ejecutados anteriormente.

**Justificación del segundo flush**

* Garantiza la eliminación total de reglas residuales.
* Asegura que el comportamiento del firewall no dependa de pruebas anteriores.
* Permite partir de un estado completamente predecible.
* Evita que el sistema acepte tráfico de forma no controlada.

Este paso fue clave para asegurar que el funcionamiento final del firewall dependiera exclusivamente de las reglas definidas a continuación.

#### Activación del reenvío IP

Para que el Firewall pudiera reenviar tráfico entre la red interna y la interfaz WAN, fue necesario habilitar el reenvío IP a nivel del kernel.

Se ejecutó el siguiente comando:

{% hint style="info" %}
sysctl -w net.ipv4.ip\_forward=1
{% endhint %}

Para que esta configuración fuera persistente tras reinicios del contenedor, se añadió la siguiente línea al archivo /etc/sysctl.conf:

{% hint style="info" %}
net.ipv4.ip\_forward=1
{% endhint %}

Sin el reenvío IP activado, el sistema puede recibir paquetes, pero no los reenvía entre interfaces, lo que impide completamente la salida a Internet de los clientes.

#### Configuración de NAT mediante MASQUERADE

Dado que la interfaz WAN del Firewall obtiene su dirección IP mediante DHCP, se utilizó la acción **MASQUERADE**, que es adecuada para direcciones IP dinámicas.

Se añadió la siguiente regla de NAT:

{% hint style="info" %}
iptables -t nat -A POSTROUTING -s 10.10.10.0/24 -o eth0 -j MASQUERADE


{% endhint %}

**Explicación de la regla**

* Se trabaja sobre la tabla nat.
* La cadena POSTROUTING aplica la regla justo antes de que el paquete salga del sistema.
* Se especifica la red interna 10.10.10.0/24.
* Se indica la interfaz de salida hacia Internet.
* MASQUERADE sustituye la IP de origen (10.10.10.x) por la IP actual del Firewall (192.168.109.58).

Esta configuración permite que los equipos de la red interna accedan a Internet utilizando la dirección IP del Firewall.

**Guardado de las reglas iptables**

En Alpine Linux, las reglas de iptables **no se conservan automáticamente** tras un reinicio del sistema. Por este motivo, fue necesario guardarlas manualmente.

Se ejecutó el siguiente comando:

{% hint style="info" %}
iptables-save > /etc/iptables/rules.v4
{% endhint %}

Si las reglas no se guardan, el Firewall pierde toda la configuración de NAT tras reiniciar el contenedor, dejando a la red interna sin conectividad.

**Incidencia relacionada con iptables**

Cliente sin acceso a Internet debido a reglas incorrectas

**Problema** El cliente no tenía acceso a Internet a pesar de disponer de una dirección IP válida y conectividad con el Firewall.

**Causa** Existían reglas residuales en iptables que interferían con el NAT y el reenvío de tráfico.

**Solución**

* Realizar un flush completo de iptables.
* Definir explícitamente las políticas por defecto.
* Ejecutar un segundo flush para consolidar la limpieza.
* Reaplicar las reglas de NAT desde cero.
* Verificar que el reenvío IP estuviera activado.

**Verificación del funcionamiento del firewall**

Desde el Firewall se realizaron las siguientes comprobaciones:

{% hint style="info" %}
iptables -t nat -vnL POSTROUTING\
cat /proc/sys/net/ipv4/ip\_forward
{% endhint %}

El aumento de los contadores en la tabla NAT confirmó que el tráfico del cliente estaba siendo correctamente traducido y reenviado hacia Internet.

Incidencias encontradas y resolución

#### Incidencia 1: Acceso incorrecto a la interfaz web de Proxmox

**Problema** Se intentó acceder a la interfaz web indicando solo la dirección IP del servidor.

**Causa** Proxmox utiliza un puerto específico para su interfaz web.

**Solución** Acceder siempre mediante:

```
https://IP_DEL_PROXMOX:8006
```

***

#### Incidencia 2: Firewall sin acceso a Internet

**Problema** La interfaz WAN se configuró inicialmente con una IP estática.

**Causa** El router del aula solo permitía salida a Internet mediante DHCP.

**Solución** Configurar la interfaz WAN como cliente DHCP.

***

#### Incidencia 3: Cliente sin acceso a Internet

**Problema** Configuración incorrecta del enrutamiento interno.

**Solución** Verificar la asignación de IP por dnsmasq, la activación del reenvío IP y la regla de NAT en el Firewall.

***

### 12. Verificación del funcionamiento

Desde el cliente:

{% hint style="info" %}
ping 10.10.10.10\
ping 8.8.8.8
{% endhint %}

Desde el Firewall:

{% hint style="info" %}
iptables -t nat -vnL POSTROUTING\
cat /proc/sys/net/ipv4/ip\_forward
{% endhint %}

***

### 13. Conclusión

Se implementó con éxito un entorno funcional basado en Proxmox y contenedores LXC, utilizando Alpine Linux como sistema operativo. El Firewall configurado proporciona servicios de enrutamiento, DHCP y NAT de forma eficiente y estable.

El uso de contenedores LXC permitió reducir el consumo de recursos y mejorar el rendimiento, mientras que la correcta configuración de la red y del firewall garantizó la conectividad del cliente hacia Internet. Este laboratorio refleja un escenario realista y alineado con prácticas habituales en entornos profesionales.







