# OPNsense - Firewall y Router Central

OPNsense es una distribución de firewall basada en FreeBSD que actúa como el único punto de enrutamiento entre todas las redes del sistema. Cada comunicación entre VLANs pasa obligatoriamente por él, donde se aplican las reglas de seguridad antes de permitir o denegar el tráfico.

OPNsense requiere dos interfaces de red. La interfaz em0 es una NIC integrada Gigabit que se conecta directamente al router de la escuela en modo DHCP. Esta interfaz recibe asignación dinámica en el rango 192.168.109.x y sirve como puerta de salida (WAN) hacia internet.

La segunda interfaz, denominada ue0, es donde surge un detalle importante sobre la arquitectura real del laboratorio. El servidor OPNsense disponible en el laboratorio tiene solo una NIC física integrada. Para conseguir una segunda interfaz, el equipo adquirió un adaptador USB Realtek RTL8153 que proporciona una salida RJ45 Gigabit adicional. Este adaptador conecta a través de USB 3.0 y presenta latencia de menos de 1 milisegundo, imperceptible para operaciones de red interna. Aunque puede parecer improvisado, funciona de forma estable en producción: el adaptador soporta VLAN trunking (etiquetado 802.1Q) sin problemas.

Sobre la interfaz ue0 se crean subinterfaces VLAN etiquetadas que actúan como gateways de red. OPNsense configura: vlan0.10 con IP 192.168.10.1/24 (VLAN 10, cluster), vlan0.20 con IP 192.168.20.1/24 (VLAN 20, management) y vlan0.40 con IP 192.168.40.1/24 (VLAN 40, backup).

El firewall realiza también funciones adicionales en el proyecto: genera DHCP en las VLANs 10 y 20 para asignación automática de IPs, ejecuta IDS (detección de intrusiones) con Suricata, y gestiona firewall schedules que activan reglas en horarios específicos (principalmente la ventana de backup nocturno de la VLAN 40).

#### **Interfaces: Devices: VLAN**

<figure><img src="../.gitbook/assets/Captura desde 2026-05-14 16-12-37.png" alt=""><figcaption></figcaption></figure>

Esta pantalla muestra las cinco subinterfaces VLAN creadas sobre `ue0`. Todas tienen como interfaz padre `ue0` (el adaptador USB Realtek) y prioridad Best Effort por defecto. OPNsense las asigna internamente como OPT1 a OPT5: vlan0.10 para el tráfico intra-cluster, vlan0.20 para management, vlan0.30 para la DMZ física, vlan0.40 para backup y vlan0.50 como uplink de MicroOVN, configurada sin IP ya que es gestionada directamente por MicroOVN.



#### Interfaces: Assignments

<figure><img src="../.gitbook/assets/Captura desde 2026-05-14 16-09-02.png" alt=""><figcaption></figcaption></figure>

Aquí se ve el mapeo entre las interfaces lógicas de OPNsense y los dispositivos físicos o virtuales. LAN apunta a `ue0`, WAN apunta a `em0`, y cada OPT apunta a su subinterfaz VLAN correspondiente. Esta pantalla es donde se cometió uno de los errores más costosos del proyecto: las VLANs se crearon inicialmente con `em0` como interfaz padre en lugar de `ue0`, lo que las asociaba a la WAN. Hasta corregir esta asignación, ningún dispositivo interno podía comunicarse con el gateway.



**Firewall: Rules: OPT1 (VLAN 10 — cloud)**

<figure><img src="../.gitbook/assets/Captura desde 2026-05-14 16-10-26.png" alt=""><figcaption></figcaption></figure>

La VLAN 10 tiene una única regla explícita: permitir todo el tráfico IPv4 originado en la propia red OPT1 hacia cualquier destino. Esta regla permite que los nodos del cluster se comuniquen entre sí y tengan salida a internet. Todo lo demás — incluyendo el tráfico de VLAN 10 hacia VLAN 20 — queda bloqueado por la política implícita deny-all que OPNsense aplica en todas las interfaces OPT cuando no hay regla que lo permita explícitamente.



#### **Firewall: Rules: OPT2 (VLAN 20 — management)**

<figure><img src="../.gitbook/assets/Captura desde 2026-05-14 16-09-43.png" alt=""><figcaption></figcaption></figure>

Esta interfaz tiene el conjunto de reglas más detallado del sistema, reflejo de que es la red de administración con más relaciones de confianza. Las primeras tres reglas permiten que el Admin PC (192.168.20.20) acceda por TCP a cada nodo del cluster. Las siguientes seis reglas permiten que los tres nodos se conecten al Admin PC en el puerto 636 (LDAPS, para autenticación contra el servidor OpenLDAP) y en el puerto 1514 (Wazuh, para el envío de eventos de seguridad al servidor de monitorización). A continuación hay una regla que permite al Admin PC salida general a cualquier destino. El bloque al final actúa como regla de cierre explícita, aunque OPNsense ya aplica deny-all implícito. Las reglas se evalúan en orden de primera coincidencia, por lo que el orden importa.



#### **Firewall: Rules: OPT4 (VLAN 40 — backup)**

<figure><img src="../.gitbook/assets/Captura desde 2026-05-14 16-10-09.png" alt=""><figcaption></figcaption></figure>

Las reglas de la VLAN 40 implementan el acceso temporal nocturno del servidor de backup. Hay tres reglas de paso que permiten al servidor de backup (192.168.40.10) conectarse por SSH al puerto 22 de cada uno de los tres nodos (192.168.20.11, .12 y .13). Hay también una regla de paso general desde 192.168.40.10 hacia cualquier destino. Las reglas de bloqueo al inicio y al final actúan como contención. En un despliegue completo, estas reglas llevarían un Firewall Schedule asociado que las activaría únicamente durante la ventana nocturna definida, bloqueando el acceso del servidor de backup durante el resto del día.
