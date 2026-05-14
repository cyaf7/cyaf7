# OPNsense - Firewall y Router Central

OPNsense es una distribución de firewall basada en FreeBSD que actúa como el único punto de enrutamiento entre todas las redes del sistema. Cada comunicación entre VLANs pasa obligatoriamente por él, donde se aplican las reglas de seguridad antes de permitir o denegar el tráfico.

OPNsense requiere dos interfaces de red. La interfaz em0 es una NIC integrada Gigabit que se conecta directamente al router de la escuela en modo DHCP. Esta interfaz recibe asignación dinámica en el rango 192.168.109.x y sirve como puerta de salida (WAN) hacia internet.

La segunda interfaz, denominada ue0, es donde surge un detalle importante sobre la arquitectura real del laboratorio. El servidor OPNsense disponible en el laboratorio tiene solo una NIC física integrada. Para conseguir una segunda interfaz, el equipo adquirió un adaptador USB Realtek RTL8153 que proporciona una salida RJ45 Gigabit adicional. Este adaptador conecta a través de USB 3.0 y presenta latencia de menos de 1 milisegundo, imperceptible para operaciones de red interna. Aunque puede parecer improvisado, funciona de forma estable en producción: el adaptador soporta VLAN trunking (etiquetado 802.1Q) sin problemas.

Sobre la interfaz ue0 se crean subinterfaces VLAN etiquetadas que actúan como gateways de red. OPNsense configura: vlan0.10 con IP 192.168.10.1/24 (VLAN 10, cluster), vlan0.20 con IP 192.168.20.1/24 (VLAN 20, management) y vlan0.40 con IP 192.168.40.1/24 (VLAN 40, backup).

El firewall realiza también funciones adicionales en el proyecto: genera DHCP en las VLANs 10 y 20 para asignación automática de IPs, ejecuta IDS (detección de intrusiones) con Suricata, y gestiona firewall schedules que activan reglas en horarios específicos (principalmente la ventana de backup nocturno de la VLAN 40).
