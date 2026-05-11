# hardware requerido

## 1.1 Hardware Requerido

### Visión General

La infraestructura de Spotly se compone de tres máquinas servidoras, un firewall central, un dispositivo de red, un equipo administrativo y componentes periféricos. Toda la arquitectura se construye sobre hardware físico disponible en el laboratorio de la escuela, sin dependencia de proveedores de nube pública.

### Nodos del Clúster MicroCloud (3 máquinas idénticas)

El clúster funciona sobre tres servidores físicos denominados nodo1, nodo2 y nodo3. Cada uno ejecuta Ubuntu Server 22.04.5 LTS y contribuye recursos de CPU, RAM y almacenamiento al sistema distribuido.

Cada nodo dispone de 8 núcleos de procesamiento a 2.4 GHz aproximadamente, lo que permite ejecutar entre 3 y 4 máquinas virtuales medianas simultáneamente sin degradación de rendimiento. Los 16 GB de RAM se distribuyen entre el sistema operativo base (2-3 GB), los componentes de MicroCloud —LXD, MicroCeph y MicroOVN— (8-10 GB) y espacio de operación (3-4 GB restantes). Este balance es crítico porque si la RAM disponible cae por debajo de 14 GB, MicroCeph entra en estado HEALTH\_WARN y comienza a rechazar escrituras.

El almacenamiento primario es un SSD de 500 GB por nodo. De este espacio, el sistema operativo ocupa 20 GB en la partición raíz, dejan 350 GB mínimo disponibles para los dispositivos loop que funcionan como OSDs de Ceph. Los 80 GB restantes sirven como espacio de trabajo para logs, temporales y buffers del sistema. La velocidad SSD es crítica: MicroCeph requiere latencia de escritura menor a 5 milisegundos. Si se utilizara almacenamiento mecánico o muy lento, el cluster entraría en estado HEALTH\_ERR por timeout de operaciones.

Cada nodo tiene una única interfaz de red Gigabit Ethernet física conectada al switch. Esta interfaz única es suficiente porque toda la segmentación en VLANs ocurre en el switch HP usando etiquetado 802.1Q: el kernel Linux del nodo crea subinterfaces virtuales (vlan10, vlan20) sobre esa interfaz física única. Todo el tráfico del cluster corre a través de esta interfaz de 1000 Mbps.

La capacidad de procesamiento requiere soporte hardware para virtualización completa. Cada nodo debe tener CPU que soporte VT-x (procesadores Intel Xeon) o AMD-V (EPYC), aunque todos los servidores del laboratorio cumplen este requisito.

### Firewall OPNsense

OPNsense es una distribución de firewall basada en FreeBSD que actúa como el único punto de enrutamiento entre todas las redes del sistema. Cada comunicación entre VLANs pasa obligatoriamente por él, donde se aplican las reglas de seguridad antes de permitir o denegar el tráfico.

El hardware del firewall es un servidor de recursos más modestos: 2 núcleos de CPU, 4 GB de RAM y 32 GB de almacenamiento SSD es suficiente. La versión instalada es OPNsense 26.1.2\_5 para arquitectura amd64, que ejecuta en FreeBSD 13.2.

OPNsense requiere dos interfaces de red. La interfaz em0 es una NIC integrada Gigabit que se conecta directamente al router de la escuela en modo DHCP. Esta interfaz recibe asignación dinámica en el rango 192.168.109.x y sirve como puerta de salida (WAN) hacia internet.

La segunda interfaz, denominada ue0, es donde surge un detalle importante sobre la arquitectura real del laboratorio. El servidor OPNsense disponible en el laboratorio tiene solo una NIC física integrada. Para conseguir una segunda interfaz, el equipo adquirió un adaptador USB Realtek RTL8153 que proporciona una salida RJ45 Gigabit adicional. Este adaptador conecta a través de USB 3.0 y presenta latencia de menos de 1 milisegundo, imperceptible para operaciones de red interna. Aunque puede parecer improvisado, funciona de forma estable en producción: el adaptador soporta VLAN trunking (etiquetado 802.1Q) sin problemas.

Sobre la interfaz ue0 se crean subinterfaces VLAN etiquetadas que actúan como gateways de red. OPNsense configura: vlan0.10 con IP 192.168.10.1/24 (VLAN 10, cluster), vlan0.20 con IP 192.168.20.1/24 (VLAN 20, management) y vlan0.40 con IP 192.168.40.1/24 (VLAN 40, backup).

El firewall realiza también funciones adicionales en el proyecto: genera DHCP en las VLANs 10 y 20 para asignación automática de IPs, ejecuta IDS (detección de intrusiones) con Suricata, y gestiona firewall schedules que activan reglas en horarios específicos (principalmente la ventana de backup nocturno de la VLAN 40).

### PC Administrativo

El equipo administrativo ejecuta Ubuntu Desktop 22.04 LTS y reside en la VLAN 20 con IP estática 192.168.20.20. Este PC cumple dos roles fundamentales: actúa como servidor LDAP centralizado (slapd corre aquí con los usuarios del equipo) y como host desde el cual se ejecutan los playbooks de Ansible que configuran los nodos.

La elección de no ejecutar LDAP dentro del cluster como máquina virtual responde a un problema de dependencia circular: si LDAP estuviera en una VM dentro del cluster, los nodos necesitarían conectar a LDAP durante el arranque para autenticarse vía SSSD, pero las VMs a su vez dependen de que LXD esté funcionando, y LXD depende de que los nodos estén correctamente inicializados. Mantener LDAP en hardware real rompe esta dependencia circular.

### PC de Respaldo

Existe un equipo destinado para backup ubicado en la VLAN 40 (backup) con IP 192.168.40.10. El sistema ejecuta respaldos automáticos por la VLAN 20 cada lunes, miércoles y viernes a las 02:00 AM mediante un playbook de Ansible. El proceso realiza tres capas de protección: exportación completa de máquinas virtuales (disco + configuración), respaldo de configuración de nodos (Netplan, hosts, bases de datos OVN) y política de retención automática que mantiene siempre las 2 versiones más recientes para optimizar los 118 GB disponibles en el servidor de backup.

### Switch HP 1810-24G

El switch es un concentrador gestionado de 24 puertos Gigabit Ethernet que implementa soporte completo para VLANs IEEE 802.1Q. Ofrece una velocidad de backplane de 48 Gbps, suficiente para manejar el tráfico esperado de 3 nodos simultáneamente. Incluye interfaz de administración web accesible en IP 192.168.2.10.

La tabla siguiente detalla la distribución de puertos:

| Puerto                       | Dispositivo     | Función                | Rol                     |
| ---------------------------- | --------------- | ---------------------- | ----------------------- |
| 4                            | Admin PC        | Management del clúster | VLAN 20                 |
| 7                            | nodo02          | Nodo del clúster       | VLANs 10, 20            |
| 11                           | nodo01          | Nodo del clúster       | VLANs 10, 20            |
| 12                           | Servidor Backup | Respaldos programados  | VLAN 40, 20             |
| 19                           | nodo03          | Nodo del clúster       | VLANs 10, 20            |
| 23                           | OPNsense (ue0)  | Firewall central       | Todos los VLANs (trunk) |
| 24                           | (Disponible)    | Internet / Redundancia | —                       |
| 1-3, 5-6, 8-10, 13-18, 20-22 | (Disponibles)   | Expansión futura       | —                       |

El puerto 23 (donde se conecta OPNsense) es especial: funciona como trunk VLAN, transportando todas las VLANs etiquetadas simultáneamente. Los puertos 23 y 24 son los candidatos naturales para conexiones de internet o redundancia en futuras expansiones.

En cada puerto se configura la participación en VLANs usando etiquetado 802.1Q. El etiquetado funciona mediante tres modos: T (Tagged) significa que el puerto transporta tramas con etiqueta VLAN explícita, el dispositivo conectado debe entender etiquetado 802.1Q; U (Untagged) significa que el puerto pertenece a una VLAN específica pero las tramas viajan sin etiqueta, útil para dispositivos legacy que no entienden VLANs; E (Excluded) significa que el puerto no participa en esa VLAN en absoluto, el tráfico es descartado en nivel de switch.

Los nodos (puertos 7, 11, 19) tienen configuración idéntica: participan como Tagged en VLAN 10 y VLAN 20. El Admin PC (puerto 4) participa como Tagged en VLAN 20. El servidor de backup (puerto 12) participa como Tagged en VLAN 40. El firewall OPNsense (puerto 23) participa como Tagged en todas las VLANs (10, 20, 40).

### Adaptador USB Realtek RTL8153

Como se mencionó en la sección de firewall, existe un adaptador USB 3.0 con salida Gigabit Ethernet que proporciona la segunda interfaz de red al servidor OPNsense. Este adaptador soporta VLAN trunking y funciona con latencia imperceptible para operaciones de red interna. La razón técnica de su uso es que el servidor no tenía segunda NIC integrada disponible.

### Cámara USB

La cámara USB es el dispositivo de captura de imágenes del sistema. Su especificación exacta será proporcionada posteriormente. Para este documento, se entiende como un periférico de captura conectado al equipo que ejecuta el procesamiento de visión artificial.

### Disco Externo SanDisk

Como segunda capa de protección física, existe un disco externo SanDisk de capacidad suficiente (1 TB aproximadamente) que se conecta manualmente de forma periódica para realizar respaldos externos de los datos más críticos. Este dispositivo se almacena en el laboratorio y se utiliza únicamente en procedimientos de mantenimiento.

### Almacenamiento Distribuido: Loop Devices y OSDs

Cada nodo necesita aportar almacenamiento al cluster MicroCeph. Como cada servidor físico tiene solo un disco SSD que ya está ocupado completamente por el sistema operativo, la solución implementada utiliza loop devices.

Un loop device es un fichero regular almacenado en disco que el kernel de Linux presenta al sistema como si fuera un dispositivo de bloque físico, igual que un pendrive o un disco duro. MicroCeph no distingue entre un loop device y un disco real: lo ve como un dispositivo de bloque válido y lo utiliza como OSD (Object Storage Daemon) para almacenamiento distribuido.

En la práctica, cada nodo tiene un fichero sparse de 200 GB ubicado en `/mnt/ceph-disks/ceph-osd.img`. Durante el arranque del nodo, el script `/etc/rc.local` ejecuta `losetup -fP /mnt/ceph-disks/ceph-osd.img`, que convierte ese fichero en un dispositivo loop (típicamente `/dev/loop11`, `/dev/loop12`, `/dev/loop13` en los tres nodos). MicroCeph añade este dispositivo como OSD con el comando `microceph disk add /dev/loopX --wipe`, y a partir de ese momento Ceph lo trata como almacenamiento real, replicando datos entre los tres OSDs.

Con factor de replicación 3 (que es el estándar de Ceph), cada bloque de datos existe simultáneamente en los tres nodos. La capacidad usable es 600 GB brutos (200 GB × 3) dividido por factor 3, resultando en 200 GB de almacenamiento realmente disponible para las máquinas virtuales del sistema.

La ventaja principal es que no requiere hardware adicional: con los servidores disponibles en el laboratorio se alcanza almacenamiento distribuido de 200 GB. La desventaja es que el rendimiento es menor que un disco físico dedicado, y existe una dependencia crítica: si el disco del sistema operativo (que contiene el fichero loop) se llena, el OSD deja de funcionar. Este problema fue experimentado en nodo02 cuando logs del sistema ocuparon casi la totalidad del espacio disponible.

### Resumen de Capacidades

El clúster resultante ofrece 24 núcleos de procesamiento combinados (3 nodos × 8 cores), 24 GB de RAM total distribuida, 200 GB de almacenamiento distribuido tolerante a fallos de un nodo, y redes virtuales completamente aisladas mediante OVN. Estos recursos son suficientes para ejecutar el MVP (producto mínimo viable) de Spotly con máquinas virtuales para backend, base de datos, detección visual y monitoreo.
