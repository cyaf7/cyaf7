# SPOTLY -  Infraestructura

### Descripción General

La infraestructura de Spotly ha sido diseñada siguiendo un modelo de segmentación de red mediante VLANs. Este enfoque permite separar de forma lógica los distintos tipos de tráfico, mejorando tanto la seguridad como la organización del sistema.

Se han definido las siguientes VLANs:

* **VLAN 10 (Cloud/DMZ):** destinada a la comunicación entre microservicios y componentes de la microcloud.
* **VLAN 20 (Management):** utilizada para la gestión interna del clúster y la comunicación entre nodos.
* **VLAN 30 (Administración):** reservada para la administración de dispositivos, incluyendo el switch y el equipo de gestión.
* **VLAN 40 (Backup):** dedicada al tráfico de copias de seguridad y replicación de datos.

El control de seguridad se centraliza en OPNsense, que actúa como firewall y gateway. Todo el tráfico entre VLANs pasa obligatoriamente por este punto, permitiendo aplicar políticas de filtrado y control de acceso.

***

### Topología de Red

La arquitectura física de la red se basa en un esquema centralizado donde OPNsense conecta la red externa con la red interna segmentada.

OPNsense dispone de una interfaz WAN conectada a la red de la escuela y una interfaz LAN conectada al switch mediante un enlace troncal. A partir de esta interfaz LAN se crean subinterfaces VLAN que actúan como gateways para cada segmento.

El switch HP 1810-24 distribuye el tráfico hacia los distintos nodos del clúster y hacia el equipo de administración. Los nodos están configurados para trabajar con múltiples VLANs a través de interfaces virtuales.

#### Asignación de Puertos

* **Puerto 23:** conexión troncal con OPNsense, transportando todas las VLANs.
* **Puertos 7, 12 y 18:** conexión de los nodos del clúster, con acceso a VLAN 10 y VLAN 20.
* **Puerto 19:** conexión del equipo de administración, asociado a la VLAN 30.

Este diseño permite mantener aislamiento entre entornos, al mismo tiempo que facilita la comunicación controlada a través del firewall.

***

### Configuración del Switch HP 1810-24

El switch se gestiona mediante interfaz web, accesible a través de su dirección IP dentro de la red de administración.

#### VLANs Definidas

Se han creado cuatro VLANs principales, además de la VLAN por defecto:

| VLAN ID | Nombre     | Función                          |
| ------- | ---------- | -------------------------------- |
| 1       | default    | Red por defecto                  |
| 10      | cloud      | Microservicios / DMZ             |
| 20      | management | Comunicación interna del clúster |
| 30      | admin      | Gestión de dispositivos          |
| 40      | backup     | Tráfico de respaldo              |

#### Configuración de Trunks

El switch utiliza enlaces troncales para transportar múltiples VLANs a través de un único puerto o grupo de puertos:

* **TRK1 (OPNsense):** transporta todas las VLANs (10, 20, 30 y 40).
* **TRK2 (Cluster):** permite a los nodos acceder a las VLANs 10 y 20.
* **TRK3 (Admin-PC):** proporciona acceso a la VLAN 30.

Todas las VLANs en los trunks se configuran como **tagged**, lo que permite que múltiples redes coexistan en el mismo enlace físico.

***

### Configuración de OPNsense

OPNsense funciona como el punto central de la red, gestionando tanto el enrutamiento como la seguridad.

#### Interfaces

* **WAN (em0):** conectada a la red externa mediante DHCP.
* **LAN (ue0):** conectada al switch mediante un trunk.
* **WiFi (opcional):** utilizada como respaldo de conectividad.

#### Subinterfaces VLAN

Sobre la interfaz LAN se crean subinterfaces para cada VLAN, asignando a cada una su correspondiente dirección IP, que actuará como gateway:

* VLAN 10 → 192.168.10.1/24
* VLAN 20 → 192.168.20.1/24
* VLAN 30 → 192.168.30.1/24
* VLAN 40 → 192.168.40.1/24

Estas configuraciones deben realizarse de forma persistente desde la interfaz web de OPNsense.

#### Estado del Firewall

Actualmente no se han definido reglas específicas. En fases posteriores se implementarán políticas de control de tráfico entre VLANs, aplicando el principio de mínimo privilegio.

***

### Configuración de Nodos Ubuntu Server

Cada nodo del clúster ejecuta Ubuntu Server 22.04 y está configurado para operar en múltiples VLANs mediante Netplan.

#### Configuración de Red

Cada nodo dispone de una interfaz física sobre la que se crean interfaces virtuales VLAN:

* **VLAN 10:** utilizada para tráfico de servicios.
* **VLAN 20:** utilizada para gestión interna.

Cada VLAN tiene su propia dirección IP, gateway y servidor DNS, apuntando siempre a OPNsense.

#### Ejemplo de Asignación

* Nodo1:
  * VLAN 10 → 192.168.10.11
  * VLAN 20 → 192.168.20.11
* Nodo2:
  * VLAN 10 → 192.168.10.12
  * VLAN 20 → 192.168.20.12
* Nodo3:
  * VLAN 10 → 192.168.10.13
  * VLAN 20 → 192.168.20.13

La configuración se aplica mediante Netplan, y posteriormente se verifican las interfaces, rutas y conectividad.

***

### Configuración de Tailscale

Tailscale se incorpora como solución de acceso remoto seguro basada en VPN mesh. Su principal ventaja es que permite conectividad directa entre dispositivos sin depender de la topología interna de la red.

#### Objetivo

Facilitar el acceso remoto a los nodos y servicios del sistema, independientemente de las VLANs o reglas del firewall.

#### Funcionamiento

Cada nodo se autentica en la red de Tailscale mediante una clave generada desde el panel de administración. Una vez conectado, recibe una dirección IP dentro de la red privada de Tailscale (rango 100.x.x.x).

Esto permite, por ejemplo:

* Acceso SSH remoto a los nodos
* Comunicación directa entre dispositivos sin configuración adicional de NAT
* Gestión centralizada desde cualquier ubicación

#### Uso recomendado

Se recomienda utilizar una cuenta compartida del proyecto para evitar dependencias de credenciales personales y facilitar la administración del sistema.

***

### Pruebas y Validación

Actualmente se han completado las configuraciones básicas de red, pero aún faltan pruebas críticas de conectividad.

#### Estado actual

Se han configurado correctamente:

* OPNsense y sus interfaces VLAN
* Switch con VLANs y trunks
* Nodo principal con Netplan

#### Pendiente

La validación más importante es comprobar que el switch está transmitiendo correctamente el tráfico etiquetado entre dispositivos.

Esto se realiza mediante pruebas de conectividad (ping) entre:

* OPNsense y los nodos
* Nodos entre sí dentro de la misma VLAN
* Nodos hacia su gateway

***

### Plan de Próximos Pasos

El desarrollo de la infraestructura se divide en varias fases:

#### Fase 1: Validación de Red

Verificar conectividad básica y corregir posibles errores en la configuración de VLANs o trunks.

#### Fase 2: Acceso Remoto

Implementar Tailscale en todos los nodos y validar acceso externo.

#### Fase 3: Clúster

Inicializar Microcloud, configurar redes OVN y preparar el entorno distribuido.

#### Fase 4: Seguridad

Definir reglas de firewall, automatizar configuraciones mediante Ansible y reforzar el control de acceso.

#### Fase 5: Despliegue

Implementar máquinas virtuales, almacenamiento distribuido y balanceadores de carga.

***

### Troubleshooting

Uno de los problemas más comunes en este tipo de arquitectura es la falta de conectividad entre VLANs debido a errores en la configuración del switch o del firewall.

Los síntomas suelen incluir la imposibilidad de hacer ping al gateway o entre nodos.

Las causas más habituales son:

* Configuración incorrecta de trunks
* VLANs no etiquetadas correctamente
* Subinterfaces VLAN inactivas en OPNsense

La verificación debe realizarse tanto en los nodos como en el firewall, comprobando interfaces, direcciones IP y rutas.

***

### Referencias

* Documentación oficial de Netplan
* Documentación de OPNsense
* Guías de Tailscale
* Manual del switch HP 1810-24

