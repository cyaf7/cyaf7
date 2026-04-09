# Spotly

## 1.Descripcion general del proyecto

Spotly es una plataforma de aparcamiento inteligente diseñada para mostrar en tiempo real el estado de las plazas de aparcamiento en la vía pública. El sistema integra sensores IoT, cámaras de visión artificial con detección mediante YOLO y OpenCV, una infraestructura cloud privada basada en MicroCloud y una aplicación móvil que permite a los usuarios conocer la disponibilidad de plazas antes de llegar.

La infraestructura del proyecto se construye enteramente sobre hardware físico disponible en el centro educativo, sin dependencia de proveedores de nube pública. Este enfoque permite demostrar un modelo de nube privada completo, con segmentación de red mediante VLANs, alta disponibilidad, almacenamiento distribuido y despliegue automatizado de servicios.

### 2. Inventario de hardware

#### 2.1 Equipos disponibles en laboratorio

| Sistema operativo         | Rol asignado                          | VLANs                     |
| ------------------------- | ------------------------------------- | ------------------------- |
| Ubuntu Server 22.04.5 LTS | Nodo MicroCloud                       | VLAN 10, VLAN 20, VLAN 50 |
| Ubuntu Server 22.04.5 LTS | Nodo MicroCloud                       | VLAN 10, VLAN 20, VLAN 50 |
| Ubuntu Server 22.04.5 LTS | Nodo MicroCloud                       | VLAN 10, VLAN 20, VLAN 50 |
| OPNsense 26.1.2 amd64     | Firewall, router, VPN, IDS            | Todas las VLANs (gateway) |
| Ubuntu Desktop 22.04 LTS  | Administracion y gestion SSH          | VLAN 20                   |
| Por definir               | Servidor de backup programado         | VLAN 40                   |
| Windows 11                | Interfaz grafica switch HP (temporal) | 192.168.2.x (directa)     |
| Windows 11                | Interfaz grafica OPNsense (temporal)  | 192.168.1.x (LAN)         |

Los dos ordenadores con Windows cumplen un papel temporal durante la fase de configuración: uno se conecta directamente a la red del switch (192.168.2.x) para acceder a su interfaz web, y el otro se conecta a la LAN del firewall (192.168.1.x) para acceder a la interfaz web de OPNsense. Una vez que la configuración de red esté completada y estabilizada, el ordenador utilizado para gestionar el switch pasará a desempeñar el rol de servidor de backup, utilizando el hardware sin necesidad de equipo adicional.

Para acceder a la interfaz web del switch, se configura manualmente una IP en el rango 192.168.2.x en el ordenador Windows correspondiente, ya que el switch HP tiene la IP 192.168.2.10 por defecto. Para acceder a OPNsense, basta con estar en la LAN (192.168.1.x) y abrir https://192.168.1.1 en el navegador. Ambas interfaces muestran advertencias de certificado SSL autofirmado que deben aceptarse para continuar.

#### 2.2 Dispositivos de red y periféricos

| **Dispositivo**      | **Modelo / Especificación**                                              | **Función**                                                                                                                                                                |
| -------------------- | ------------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Switch gestionado    | HP 1810-24G (J9801A)                                                     | Segmentacion de red mediante VLANs 802.1Q                                                                                                                                  |
| Adaptador de red USB | Realtek RTL8153 — Tosuny USB 3.0 to RJ45 Gigabit 10/100/1000 Mbps 40 Pin | Segunda interfaz de red para el firewall. Necesario porque el PC tiene una sola NIC integrada y se requieren dos interfaces: una para WAN y otra para LAN hacia el switch. |
| Camara USB           | ELP-USBFHD01M-BL170 (170 grados fisheye, 1080p)                          | Captura de imagen para detección visual de vehículos con YOLO y OpenCV                                                                                                     |
| Sensor IoT           | ESP32 + HC-SR04 ultrasonico                                              | Detección de presencia en plazas de aparcamiento                                                                                                                           |
| Disco externo        | SanDisk 1 TB                                                             | Backup externo fisico como segunda capa de proteccion                                                                                                                      |
| Pendrives USB        | Varios                                                                   | Instalacion de sistemas operativos via arranque desde USB                                                                                                                  |

El adaptador Realtek RTL8153 es especialmente relevante en este proyecto porque el ordenador destinado a firewall dispone de una unica NIC integrada. Para que OPNsense pueda actuar como router entre la red de la escuela (WAN) y la red interna (LAN hacia el switch), necesita obligatoriamente dos interfaces fisicas separadas. El adaptador USB Gigabit resuelve esta limitacion de hardware sin necesidad de instalar una tarjeta PCIe adicional.

\
3\. Sistemas operativos e instalacion

### 3.1 Ubuntu Server 22.04.5 LTS en los nodos

Los tres nodos ejecutan Ubuntu Server 22.04.5 LTS, versión de soporte extendido mantenida por Canonical. Se eligió esta versión porque es la recomendada por la documentación oficial de MicroCloud para entornos de producción. La instalación se realizó arrancando cada máquina desde un pendrive USB creado con Balena Etcher, herramienta de escritura de imágenes ISO multiplataforma. Durante la instalación se selecciono la opcion de servidor minimo sin entorno grafico, con el servidor SSH habilitado desde el primer arranque.

#### 3.2 Ubuntu Desktop 22.04 LTS en el PC de administración

El ordenador de administración ejecuta Ubuntu Desktop 22.04 LTS con entorno gráfico. Esta máquina forma parte de la VLAN 20 (management) y es el punto de entrada principal para las operaciones de administración del sistema cuando el equipo trabaja presencialmente en el laboratorio. Desde aquí se ejecutan los playbooks de Ansible, se gestionan las conexiones SSH a los nodos y se accede a los paneles de monitorización.

#### 3.3 OPNsense 26.1.2 en el firewall

OPNsense es una distribución de firewall y router de código abierto basada en FreeBSD, desarrollada por Deciso B.V. Utiliza el sistema de filtrado de paquetes pf de FreeBSD, ampliamente utilizado en entornos de producción por su estabilidad y rendimiento. La versión instalada es la 26.1.2 para arquitectura amd64.

La instalación de OPNsense se realizó creando un pendrive USB de arranque con la imagen ISO oficial, disponible en https://opnsense.org/download/. El proceso de instalación es guiado por consola y permite configurar las interfaces de red, la contraseña de root y el particionado del disco.

### &#x20;4. OPNsense: firewall y router central

#### 4.1 Qué es OPNsense y por qué se usa

OPNsense actúa como el único punto de routing entre todas las redes del sistema. Cada paquete que necesite cruzar de una VLAN a otra pasa obligatoriamente por el firewall, que aplica las reglas de seguridad correspondientes antes de permitir o denegar el tráfico. Esta arquitectura garantiza que no exista comunicación directa entre redes distintas sin inspección previa.

Además del routing entre VLANs, OPNsense gestiona en este proyecto los siguientes servicios: asignación de IPs por DHCP en la LAN base, inspección de tráfico con Suricata (IDS), VPN con WireGuard para acceso remoto del equipo, y los Firewall Schedules que limitan temporalmente el acceso de la VLAN de backup.

#### 4.2 Interfaces fisicas y asignacion de roles

| Interfaz             | Tipo fisico                   | Rol en OPNsense           | IP                               |
| -------------------- | ----------------------------- | ------------------------- | -------------------------------- |
| em0                  | NIC integrada                 | WAN — red de la escuela   | 192.168.109.x (DHCP)             |
| ue0                  | Adaptador USB Realtek RTL8153 | LAN — enlace al switch HP | 192.168.1.1/24                   |
| vlan0.10 (sobre ue0) | Subinterfaz virtual           | Gateway VLAN 10 (cloud)   | 192.168.10.1/24                  |
| vlan0.20 (sobre ue0) | Subinterfaz virtual           | Gateway VLAN 20 (mgmt)    | 192.168.20.1/24                  |
| vlan0.30 (sobre ue0) | Subinterfaz virtual           | Gateway VLAN 30 (dmz)     | 192.168.30.1/24                  |
| vlan0.40 (sobre ue0) | Subinterfaz virtual           | Gateway VLAN 40 (backup)  | 192.168.40.1/24                  |
| vlan0.50 (sobre ue0) | Subinterfaz virtual           | OVN uplink — sin IP       | Sin IP (gestionada por MicroOVN) |

Las subinterfaces VLAN se crean en OPNsense a través de Interfaces > Devices > VLAN, especificando la interfaz padre (ue0), el identificador de VLAN y una descripción. A continuación, en Interfaces > Assignments, se asigna cada subinterfaz como una interfaz del sistema, se habilita y se configura su IP estática. Este proceso convierte a OPNsense en el gateway de cada VLAN, permitiendo el routing controlado entre ellas.

### 5. Switch HP 1810-24G

#### 5.1 Descripcion del equipo

El switch HP 1810-24G (modelo J9801A) es un switch gestionado de 24 puertos Gigabit Ethernet con soporte para VLANs IEEE 802.1Q. Dispone de interfaz web de administracion accesible en la IP 192.168.2.10 por defecto. No requiere software adicional: la gestion se realiza completamente desde el navegador web.<br>

Fuente: HP 1810 Switch Series Management and Configuration Guide — Hewlett-Packard

### 5.2 Distinción critica: Trunk Configuration vs VLAN Tagging

Durante el proceso de configuración se identificó una confusión que consumió una cantidad significativa de tiempo de trabajo: la sección Trunk Configuration del HP 1810-24G no hace referencia al concepto de VLAN trunk comúnmente utilizado en redes 802.1Q. En este switch específico, Trunk Configuration corresponde exclusivamente a LACP (Link Aggregation Control Protocol), que es la técnica de combinar múltiples cables físicos para formar un único enlace lógico más rápido y con redundancia.

El VLAN trunk en terminología 802.1Q, es decir, un único cable físico que transporta múltiples VLANs etiquetadas simultáneamente, se configura en este switch exclusivamente a través de VLANs > Participation/Tagging, asignando el modo Tagged (T) al puerto correspondiente para cada VLAN. Esta distinción no está claramente documentada en la interfaz del switch y fue la causa principal de varios intentos fallidos de configuración.

### 5.3 Asignación de puertos

| Puerto | Maquina conectada           | Funcion                                                   |
| ------ | --------------------------- | --------------------------------------------------------- |
| 7      | Node 2                      | Nodo MicroCloud — VLAN 10 + 20 + 50 tagged                |
| 11     | Node 1                      | Nodo MicroCloud — VLAN 10 + 20 + 50 tagged                |
| 19     | Node 3                      | Nodo MicroCloud — VLAN 10 + 20 + 50 tagged                |
| 6      | Admin PC (Ubuntu Desktop)   | Administracion MGMT — VLAN 20 tagged                      |
| 4      | Windows PC (gestion switch) | Acceso GUI switch — VLAN 20 untagged (acceso directo)     |
| 23     | Firewall OPNsense (ue0)     | Enlace principal firewall-switch — todas las VLANs tagged |
| 10     | Laptop                      | Aceder a VLAN 20 por SSH por comodidad                    |



### 5.4 Configuracion de participacion VLAN por puerto

| VLAN             | P7 (N2) | P19 (N3) | P6 (Admin) | P4 (Win Opn) | P23 (FW) |
| ---------------- | ------- | -------- | ---------- | ------------ | -------- |
| VLAN 10 (cloud)  | T       | T        | E          | E            | T        |
| VLAN 20 (mgmt)   | T       | T        | T          | U            | T        |
| VLAN 30 (dmz)    | E       | E        | E          | E            | T        |
| VLAN 40 (backup) | E       | E        | E          | E            | T        |
| VLAN 50 (ovn)    | T       | T        | E          | E            | T        |

T = Tagged: el puerto transporta esta VLAN con etiqueta 802.1Q. U = Untagged: el puerto pertenece a esta VLAN sin etiqueta (acceso directo, para dispositivos sin soporte VLAN). E = Excluded: el puerto no participa en esta VLAN.



### 6. Segmentacion de red: VLANs

#### 6.1 Que es una VLAN

Una VLAN (Virtual Local Area Network) es una red lógica creada sobre una infraestructura física compartida. Permite separar el tráfico de distintos grupos de dispositivos como si estuvieran conectados a switches físicos distintos, aunque en realidad compartan el mismo hardware de red. El protocolo IEEE 802.1Q regula este etiquetado añadiendo 4 bytes a cada trama Ethernet para identificar a qué VLAN pertenece.

La principal ventaja de seguridad es que el tráfico entre VLANs distintas no puede fluir directamente: obligatoriamente debe pasar por el firewall, que aplica las reglas de control correspondientes. Esto significa que incluso si un atacante compromete un servicio en la VLAN DMZ, no podría acceder a los nodos del cluster en la VLAN 10 sin que el tráfico pase por OPNsense.

\
6.2 Tabla de VLANs del sistema

| **ID** | **Nombre** | **Subred**      | **Proposito**                                                                                                                                                  | **Miembros**                  |
| ------ | ---------- | --------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------- |
| 10     | cloud      | 192.168.10.0/24 | Red intra-cluster MicroCloud. Los nodos la usan para comunicarse entre si y para la inicializacion del cluster.                                                | Node1, Node2, Node3           |
| 20     | mgmt       | 192.168.20.0/24 | Red de administracion. Acceso SSH a los nodos y acceso remoto via VPN desde el exterior.                                                                       | Node1, Node2, Node3, Admin PC |
| 30     | dmz        | 192.168.30.0/24 | Zona desmilitarizada. Aqui se ubica el reverse proxy Traefik que recibe el trafico publico de internet.                                                        | VM dmz-receiver               |
| 40     | backup     | 192.168.40.0/24 | Red de backup. El servidor de backup tiene acceso temporal a las otras VLANs unicamente durante la ventana horaria nocturna programada.                        | Backup PC (192.168.40.10)     |
| 50     | ovn-uplink | Sin IP          | Red de enlace para MicroOVN. No tiene IP asignada porque es gestionada directamente por MicroCloud para conectar las redes virtuales internas con el exterior. | Node1, Node2, Node3           |

#### 6.3 Cómo funciona el trafico entre VLANs

El tráfico entre VLANs distintas no puede fluir directamente a través del switch: debe pasar por el firewall OPNsense, que actúa como router entre ellas. Por ejemplo, cuando un cliente de la aplicación movil envia una petición desde internet, el tráfico entra por la WAN de OPNsense, este lo enruta hacia la VLAN 30 (DMZ) donde esta el reverse proxy, y el reverse proxy lo reenvía internamente hacia la VLAN 10 donde esta el backend. En ningún momento hay comunicación directa entre la WAN y los nodos del cluster.

Las reglas de firewall de OPNsense se configuran por interfaz. Cada VLAN tiene su propio conjunto de reglas que definen qué tráfico se permite entrar y salir. Por defecto, todo el tráfico que no coincida con una regla de permiso explícita es descartado.

### 7. Pruebas de conectividad: proceso real y errores

La configuración de la red fue el proceso más complejo de esta fase del proyecto. Se realizaron tres pruebas de conectividad secuenciales, cada una añadiendo un componente adicional al circuito. A continuación se documenta el proceso tal como ocurrió, incluyendo los errores encontrados, las hipótesis manejadas y las soluciones aplicadas.

#### 7.1 Prueba 1: Internet a través del firewall hacia una computadora

El objetivo de esta primera prueba era verificar que OPENsense estaba correctamente instalado y configurado, que la interfaz WAN recibió conexion de internet desde la red de la escuela, y que la interfaz LAN distribuye correctamente esa conexión a los dispositivos conectados.

La prueba consistió en conectar un cable desde el puerto de red de la escuela a la interfaz WAN del firewall (em0, NIC integrada), y otro cable desde la interfaz LAN (ue0, adaptador USB Realtek) directamente a una computadora, sin switch de por medio.

En el primer intento, la computadora no obtenía ninguna IP y no había conectividad. Tras revisar las conexiones físicas, se identificó que el cable utilizado para la conexión WAN era defectuoso. Al sustituirlo por un cable en buen estado, la computadora obtuvo la IP 192.168.1.149 por DHCP desde la LAN de OPNsense, y el firewall recibio correctamente una IP del rango de la escuela (192.168.109.x) en su interfaz WAN. La prueba se dio por superada.

#### 7.2 Prueba 2: Internet a través del switch sin configuración VLAN

El objetivo de la segunda prueba era verificar que el switch HP 1810-24G pasaba trafico correctamente en su estado por defecto, antes de aplicar ninguna configuración de VLANs. El circuito probado fue: red de la escuela > firewall > switch > computadora.

Al conectar la computadora al switch, se esperaba que obtuviera una IP del rango de la LAN de OPNsense (192.168.1.x). Sin embargo, la computadora obtuvo la IP 192.168.109.49, que pertenecía directamente al rango de la red de la escuela. Esto indicaba que el trafico no estaba pasando por el firewall sino que llegaba directamente desde la red de la escuela.

La investigación reveló que el problema no era el switch, que funcionaba correctamente en modo no gestionado, sino la configuración de interfaces en OPNsense: la asignación de WAN y LAN estaba invertida. La interfaz que debía ser WAN (conectada a la escuela) estaba configurada como LAN, y viceversa. Al corregir la asignación de interfaces en la consola de OPNsense mediante la opción 1 (Assign Interfaces), el tráfico empezó a fluir correctamente a través del firewall.<br>

#### 7.3 Prueba 3: Circuito completo con configuracion VLAN

La tercera prueba fue la más extensa y compleja. El objetivo era verificar que las VLANs configuradas tanto en OPNsense como en el switch HP funcionaban correctamente, permitiendo que los nodos se comunicarán con el gateway del firewall a través de sus respectivas VLANs.

El circuito probado fue: red de la escuela > em0 (WAN) > OPNsense > ue0 (LAN) > switch HP > nodos. Para que este circuito funcionara con VLANs, era necesario que: el switch tuviera los puertos correctamente etiquetados (tagged) en cada VLAN, OPNsense tuviera las interfaces VLAN creadas con las IPs gateway correctas, y los nodos tuvieran configuradas las IPs estáticas en cada VLAN mediante Netplan.

#### Error 1: VLANs configuradas sobre la interfaz WAN

Al crear las interfaces VLAN en OPNsense (Interfaces > Devices > VLAN), se seleccionó inicialmente em0 como interfaz padre en lugar de ue0. Esto significaba que el tráfico VLAN se intentaba enviar por la interfaz WAN, conectada a internet, en lugar de por la interfaz LAN, conectada al switch. Como resultado, los nodos no reciben ninguna respuesta del gateway aunque el switch estuviera correctamente configurado. El error se corrigió editando cada subinterfaz VLAN para cambiar la interfaz padre de em0 a ue0.

#### Error 2: Configuracion incorrecta de Trunk Configuration

Durante la configuración del switch se accedió a la sección Trunk Configuration con la intención de habilitar el transporte de múltiples VLANs por el puerto 23 (conexión con el firewall). Se crearon y configuraron trunks en esa sección, sin obtener resultado. Tras varias horas de pruebas, se identificó que Trunk Configuration en el HP 1810-24G corresponde a LACP (Link Aggregation Control Protocol), no a VLAN trunking. Esta función agrupa múltiples puertos físicos en un enlace lógico para aumentar el ancho de banda, que no era lo que se buscaba. El VLAN trunking se configura en este switch exclusivamente a través de VLANs > Participation/Tagging. Toda la configuración de Trunk Configuration se eliminó y se configuró el tagging correcto por puerto.

#### Error 3: Reglas de firewall bloqueando el tráfico durante las pruebas

OPNsense aplica por defecto una política de deny all en las interfaces OPT (las VLANs). Esto significa que aunque el routing estuviera correctamente configurado, los pings de los nodos al gateway eran descartados por las reglas de firewall. Para verificar que el problema era la conectividad de red y no las reglas, se deshabilitaron temporalmente las reglas de firewall en las interfaces VLAN. Una vez confirmada la conectividad, las reglas se volvieron a habilitar y se configuraron correctamente para permitir únicamente el tráfico necesario

**Resultado final**

Tras corregir los tres errores descritos, se verificó conectividad completa entre los nodos y el gateway del firewall en la VLAN 10 y la VLAN 20. Los nodos podían hacer ping a 192.168.10.1 y 192.168.20.1, y el firewall podía hacer ping a los nodos. La segmentación de red quedó operativa.

### 8. Configuración de los nodos Ubuntu Server

#### 8.1 Configuración de red con Netplan

Ubuntu Server 22.04 utiliza Netplan como herramienta de configuración de red. Netplan trabaja con ficheros YAML ubicados en /etc/netplan/ que definen las interfaces, direcciones IP y rutas. La configuración de cada nodo incluye la interfaz física sin IP propia y subinterfaces VLAN con IPs estáticas.

El fichero de configuración para el nodo 1 (/etc/netplan/00-installer-config.yaml) es el siguiente:

```
network:
  version: 2
  renderer: networkd
  ethernets:
    enp3s0:
      dhcp4: false
  vlans:
    vlan10:
      id: 10
      link: enp3s0
      addresses: 192.168.10.11/24
      routes:
        - to: default
          via: 192.168.10.1
      nameservers:
        addresses: 8.8.8.8
    vlan20:
      id: 20
      link: enp3s0
      addresses: 192.168.20.11/24

```

Los cambios se aplican ejecutando sudo netplan apply. Los nodos 2 y 3 tienen configuraciones idénticas con las IPs .12 y .13 respectivamente.

#### 8.3 Machine ID: requisito previo a MicroCloud

El machine-id es un identificador único que Ubuntu genera durante la instalación y que MicroCloud utiliza internamente para distinguir cada nodo del cluster. Si dos nodos tienen el mismo machine-id, lo que puede ocurrir cuando se clonan instalaciones, el cluster no puede inicializarse correctamente porque el sistema no sabe distinguirlos.

Se verifica el machine-id de cada nodo con el comando:

```
cat /etc/machine-id
```

Si dos nodos tienen el mismo valor, se regenera con:

```
truncate -s 0 /etc/machine-id
truncate -s 0 /var/lib/dbus/machine-id
systemd-machine-id-setup
```

Los tres nodos tienen machine-ids únicos verificados, lo que es un requisito previo obligatorio antes de ejecutar microcloud init.<br>

### 9. Almacenamiento: particiones LVM para MicroCeph

#### 9.1 El problema del hardware disponible

MicroCeph, el sistema de almacenamiento distribuido de MicroCloud, requiere que cada nodo del cluster aporte al menos un disco dedicado exclusivamente al almacenamiento. En un entorno de producción ideal, cada nodo tendría tres discos físicos: uno para el sistema operativo, uno para almacenamiento local y uno para MicroCeph.

En este proyecto, los ordenadores del laboratorio disponen de un único disco físico por máquina. Para poder cumplir el requisito de MicroCeph sin hardware adicional, se optó por crear una partición lógica dedicada dentro del espacio libre del disco mediante LVM (Logical Volume Manager).

#### 9.2 Qué es LVM y por qué se usa

LVM es una capa de abstracción entre el hardware de almacenamiento físico y el sistema de ficheros del sistema operativo. Permite crear, redimensionar y eliminar particiones lógicas de forma flexible sin modificar la tabla de particiones del disco. Ubuntu Server utiliza LVM por defecto durante la instalación, creando un grupo de volúmenes llamado ubuntu-vg.

Tras la instalación de Ubuntu, el sistema operativo ocupa aproximadamente 100 GB del disco en un volumen lógico (ubuntu--vg-ubuntu--lv), dejando libre el espacio restante dentro del grupo de volúmenes. En los ordenadores del laboratorio, con discos de 465 GB, quedan disponibles aproximadamente 363 GB que pueden usarse para crear volúmenes adicionales sin riesgo para el sistema operativo existente.

#### 9.3 Creación del volumen LVM para MicroCeph

En cada nodo se crea un volumen lógico de 200 GB dedicado a MicroCeph con el siguiente comando:

```
sudo lvcreate -L 200G -n ceph-data ubuntu-vg
```

Este comando crea un volumen llamado ceph-data dentro del grupo de volumenes ubuntu-vg. El volumen aparece como /dev/ubuntu-vg/ceph-data y está disponible como dispositivo de bloque para MicroCeph. Es fundamental no formatear este volumen ni montarlo: MicroCeph necesita el dispositivo completamente en bruto (raw) para gestionarlo de forma nativa.

Una vez creado, se puede verificar con:

```
lsblk | grep ceph
```

La salida debe mostrar el volumen de 200 GB con tipo lvm y sin punto de montaje.

### 9.4 Estado de las particiones por nodo

| **Nodo**  | **Volumen OS**        | **Volumen Ceph**      | **Tamano Ceph** | **Estado**                 |
| --------- | --------------------- | --------------------- | --------------- | -------------------------- |
| Node 1    | ubuntu--vg-ubuntu--lv | ubuntu--vg-ceph--data | 200 GB          | Creado                     |
| Node 2    | ubuntu--vg-ubuntu--lv | ubuntu--vg-ceph--data | 200 GB          | Creado                     |
| Node 3    | ubuntu--vg-ubuntu--lv | ubuntu--vg-ceph--data | 200 GB          | Creado                     |
| Backup PC | Por definir           | No aplica             | —               | Pendiente de configuración |

Esta solución implica que el sistema operativo y MicroCeph comparten el mismo disco físico, lo cual no es la configuración óptima para producción ya que un fallo del disco afectaría ambos componentes. Sin embargo, es completamente funcional para el entorno de laboratorio y, gracias a la replicación de MicroCeph en los tres nodos, los datos permanecen disponibles aunque un nodo completo falle.

### 10. Usuarios, grupos y seguridad de acceso SSH

#### 10.1 Modelo de acceso y principio de mínimo privilegio

El sistema aplica el principio de mínimo privilegio: cada usuario y proceso dispone únicamente de los permisos necesarios para su función. No existe acceso directo como root desde el exterior, y los servicios se ejecutan con usuarios sin privilegios de administración.

#### 10.2 Estructura de usuarios y grupos (pendiente de implementación completa)

| **Usuario**             | **Función**                                        |
| ----------------------- | -------------------------------------------------- |
| user1 (infraestructura) | Administracion de red, VLANs, firewall, MicroCloud |
| user2 (backend)         | Desarrollo y despliegue del backend                |
| user3 (vision)          | Vision artificial, camara, integracion IoT         |
| prof                    | Supervision sin capacidad de modificación          |

#### 10.3 Comandos de configuración ejecutados en cada nodo

Creacion de grupos:

```
groupadd spotly-admin
groupadd spotly-readonly
```

Creacion de usuario administrador (y se repite para cada miembro):

```
useradd -m -s /bin/bash -G spotly-admin,sudo user1
passwd user1
chage -d 0 user1
```

Fichero de permisos sudo (/etc/sudoers.d/spotly):

```
%spotly-admin ALL=(ALL:ALL) ALL
%spotly-readonly ALL=(ALL) NOPASSWD: /bin/systemctl status *,
    /usr/bin/journalctl *, /snap/bin/lxc list, /snap/bin/lxc info *
```

#### 10.4 Autenticacion SSH por clave publica (pendiente)

El acceso SSH se realizará exclusivamente mediante pares de claves criptográficas ED25519. La autenticación por contraseña estará deshabilitada en todos los nodos mediante PasswordAuthentication no en /etc/ssh/sshd\_config. Esto elimina la posibilidad de ataques de fuerza bruta por contraseña, ya que sin la clave privada correspondiente no existe mecanismo de autenticación alternativo.

Generación del par de claves en el equipo personal de cada administrador:

```
ssh-keygen -t ed25519 -C 'usuario@spotly' -f ~/.ssh/spotly_usuario
```

Copia de la clave pública a cada nodo (mientras la autenticación por contraseña esté activa):

```
ssh-copy-id -i ~/.ssh/spotly_usuario.pub usuario@192.168.20.11
```

Una vez verificado el acceso por clave, se deshabilita la autenticación por contraseña y se reinicia el servicio SSH. Este paso está pendiente de ejecución porque requiere que todos los miembros del equipo estén presentes en el laboratorio con sus equipos personales para generar y distribuir las claves.

### 11. MicroCloud: plataforma de nube privada

#### 11.1 Que es MicroCloud y por que se eligio

MicroCloud es una plataforma de nube privada desarrollada por Canonical que integra en un único sistema de instalación tres componentes: LXD para virtualización, MicroCeph para almacenamiento distribuido y MicroOVN para redes virtuales. El objetivo de MicroCloud es reducir la complejidad de configuración de estos tres sistemas, que de forma independiente requieren conocimientos avanzados y muchas horas de configuración.

Se eligió MicroCloud en lugar de alternativas como Proxmox VE por las siguientes razones: integra almacenamiento distribuido y redes virtuales avanzadas de forma nativa; esta disenado especificamente para clusters de tres o más nodos con alta disponibilidad; su arquitectura se alinea con los modelos de nube pública; y es el entorno de referencia para despliegue de LXD en produccion segun la documentacion oficial de Canonical. Tambien parecio interesante la idea de&#x20;

#### 11.2 LXD: virtualizacion

LXD es el hipervisor que gestiona las máquinas virtuales del cluster. Permite crear tanto contenedores del sistema (que comparten el kernel del anfitrión) como máquinas virtuales completas con emulación de hardware mediante QEMU/KVM. En este proyecto se usan máquinas virtuales completas para garantizar aislamiento total entre los distintos servicios.

#### 11.3 MicroCeph: almacenamiento distribuido

MicroCeph implementa Ceph, el sistema de almacenamiento distribuido de código abierto. Distribuye los datos en bloques que se replican automáticamente entre todos los nodos del clúster. Con el factor de replicación por defecto de 3 y tres nodos, cada bloque existe en lostres nodos simultáneamente, lo que garantiza que los datos siguen disponibles aunque un nodo falle completamente.

#### 11.4 MicroOVN: redes virtuales

MicroOVN implementa OVN (Open Virtual Network), el sistema de redes virtuales de la comunidad Open vSwitch. Permite crear redes lógicas completamente aisladas entre máquinas virtuales, independientemente de la topología de red física. La VLAN 50 (ovn-uplink) actúa como puente entre las redes virtuales internas y el exterior, sin IP asignada porque es gestionada directamente por MicroOVN.

#### 11.5 Máquinas virtuales del cluster

| VM           | Red OVN                    | IP OVN      | Servicios                                        |
| ------------ | -------------------------- | ----------- | ------------------------------------------------ |
| dmz-receiver | ovn-dmz (10.10.10.0/24)    | 10.10.10.11 | Traefik reverse proxy — punto de entrada publico |
| app-core     | ovn-app (10.10.20.0/24)    | 10.10.20.11 | FastAPI, auth service JWT, WebSocket             |
| vision       | ovn-app (10.10.20.0/24)    | 10.10.20.12 | Validacion interna OpenCV+YOLO                   |
| data         | ovn-data (10.10.30.0/24)   | 10.10.30.11 | PostgreSQL, workers background                   |
| monitoreo    | ovn-mgmt (10.10.40.0/24)   | 10.10.40.11 | Prometheus, Grafana, Loki                        |
| admin-tools  | ovn-mgmt (10.10.40.0/24)   | 10.10.40.12 | Scripts de administracion, rsync                 |
| backup       | ovn-backup (10.10.50.0/24) | 10.10.50.11 | Servidor de backup programado                    |

#### 11.6 Alta disponibilidad: arquitectura ideal vs entorno de laboratorio

MicroCloud proporciona alta disponibilidad a nivel de datos mediante MicroCeph: con el factor de replicación por defecto de 3, cada bloque de datos existe simultáneamente en los tres nodos del clúster. Si un nodo falla, los datos siguen siendo accesibles desde los otros dos. LXD también permite la migración automática de máquinas virtuales entre nodos cuando uno de ellos queda fuera de servicio.

Sin embargo, existe una distinción importante entre alta disponibilidad a nivel de software y alta disponibilidad a nivel de hardware físico. En un entorno de producción real, cada nodo del clúster debería tener una máquina física idéntica como réplica de respaldo. Si el hardware de un nodo falla completamente, por ejemplo por un fallo de la placa base o de la fuente de alimentación, ese nodo queda fuera del cluster hasta ser reparado o sustituido. Durante ese tiempo, el cluster opera con dos nodos en lugar de tres, lo que reduce la tolerancia a fallos adicionales.

La arquitectura ideal para un despliegue de producción sería la siguiente: por cada nodo del cluster existiría una máquina física idéntica en standby, con el mismo hardware, la misma configuración de red y el mismo software instalado. Si el nodo principal falla, la máquina de standby entra en el cluster automáticamente sin intervención manual, manteniendo el clúster en tres nodos activos en todo momento. Este modelo se denomina N+1 redundancy, donde N es el número de nodos necesarios para operar y el 1 adicional garantiza continuidad ante un fallo.

En este proyecto no fue posible implementar esta arquitectura debido a la disponibilidad limitada de hardware en el laboratorio. Los seis ordenadores Dell disponibles se distribuyeron entre los roles esenciales del sistema: tres nodos de MicroCloud, un firewall, un equipo de administración y uno destinado a backup. No quedaban equipos disponibles para actuar como réplicas físicas de los nodos del clúster. Esta limitación queda documentada como una diferencia conocida respecto a un despliegue de producción real, y no afecta a la validez funcional del sistema en el entorno de laboratorio.



### 12. Herramientas utilizadas dentro del cluster

#### 12.1 Podman en lugar de Docker

Todos los servicios se ejecutan como contenedores Podman dentro de las máquinas virtuales LXD. Podman es un motor de contenedores compatible con la especificación OCI que presenta ventajas de seguridad respecto a Docker: los contenedores se ejecutan sin privilegios de root por diseño (rootless), no existe un daemon persistente que se ejecute como root, y es completamente compatible con los comandos Docker existentes.

#### 12.2 FastAPI, PostgreSQL y WebSocket

El backend se implementa con FastAPI, framework de Python de alto rendimiento para APIs REST. PostgreSQL actúa como base de datos principal con triggers que se ejecutan automáticamente al cambiar el estado de una plaza, notificando al backend mediante pg\_notify sin necesidad de polling. El servicio WebSocket mantiene conexiones abiertas con los clientes de la aplicación móvil para enviar actualizaciones en tiempo real.

#### 12.3 YOLO y OpenCV

YOLO (You Only Look Once) es un modelo de detección de objetos en tiempo real. OpenCV complementa a YOLO gestionando la captura de frames y la lógica de confirmación de estados. Para reducir falsos positivos, el sistema requiere que el mismo estado sea detectado en tres frames consecutivos antes de generar un evento. El script se ejecuta en el MacBook de uno de los miembros del equipo, conectado a un hotspot 4G, simulando un despliegue real en la vía pública.

#### 12.4 Traefik como reverse proxy

Traefik es un reverse proxy diseñado para entornos de contenedores. A diferencia de Nginx, descubre automáticamente los servicios disponibles y configura el enrutamiento de forma dinámica. Se despliega en la VM dmz-receiver dentro de la VLAN DMZ y es el único punto de entrada público del sistema.

### 13. Estrategia de backup

#### 13.1 Cómo funciona el acceso temporal de la VLAN de backup

El servidor de backup está ubicado exclusivamente en la VLAN 40. Los nodos del cluster no forman parte de esta VLAN y no necesitan hacerlo. Lo que permite el sistema de backup es que, durante una ventana horaria específica, OPNsense activa una regla de firewall que permite al servidor de backup (192.168.40.10) acceder a las VLANs donde estan los datos que necesita respaldar (VLAN 10 y VLAN 20).

Fuera de esa ventana horaria, la regla no está activa y la VLAN 40 queda completamente aislada del resto del sistema. Esto significa que aunque el servidor de backup fuera comprometido por un atacante, este no tendría acceso a los datos de producción la mayor parte del tiempo.

OPNsense implementa esta funcionalidad mediante Firewall Schedules: se define un horario (por ejemplo, de 02:30 a 04:00 todos los días) y se asocia a la regla de firewall que permite el tráfico desde VLAN 40 hacia VLAN 10 y VLAN 20. Cuando el horario no está activo, la regla se ignora automáticamente.

#### 13.2 Backup interno con rsync

El rsync es una herramienta de sincronización de ficheros que transfiere únicamente los datos que han cambiado desde la última sincronización, lo que la hace eficiente en términos de tiempo y ancho de banda. En el servidor de backup, una tarea cron ejecuta rsync automáticamente durante la ventana horaria en que OPNsense tiene activa la regla de acceso:

```
0 3 * * * /usr/local/bin/backup.sh
```

El script backup.sh realiza rsync de los directorios críticos de las máquinas virtuales del cluster hacia el servidor de backup. Preserva permisos, fechas de modificación y enlaces simbólicos, y registra el resultado en un fichero de log para auditoría.

#### 13.3 Backup externo en disco SanDisk

Como segunda capa de protección, se realiza periódicamente una copia de los datos más críticos en un disco SanDisk externo de 1 TB. Este backup físico proporciona protección ante fallos que afecten a toda la infraestructura del laboratorio, como un corte de corriente prolongado o un fallo múltiple de hardware simultáneo en los tres nodos.

### 14. Automatizacion con Ansible y Semaphore

#### 14.1 Ansible

Ansible es una herramienta de automatización de infraestructura que permite gestionar la configuración de múltiples servidores de forma simultánea mediante ficheros YAML llamados playbooks. No requiere la instalación de ningún agente en los servidores gestionados: se conecta por SSH y ejecuta los comandos necesarios de forma remota. Un único playbook puede instalar paquetes, configurar ficheros, gestionar servicios y verificar el estado de las tres máquinas en paralelo.

#### 14.2 Semaphore

Semaphore es una interfaz web de código abierto para Ansible que permite ejecutar playbooks, gestionar inventarios y visualizar el historial de ejecuciones sin necesidad de acceder a la línea de comandos. Centraliza la ejecución de tareas de administración y facilita la colaboración entre los miembros del equipo.

### 15. Acceso remoto: Tailscale y WireGuard

#### 15.1 Tailscale

Tailscale es una VPN mesh basada en WireGuard que simplifica la conexión entre dispositivos. Gestiona automáticamente el descubrimiento de peers y el establecimiento de túneles cifrados. Se utiliza actualmente durante el desarrollo para acceder a los nodos desde fuera del laboratorio, usando una cuenta compartida del equipo para evitar el uso de credenciales personales.

#### 15.2 WireGuard en OPNsense

WireGuard es el protocolo VPN que se implementará en OPNsense como solución de acceso remoto definitiva. El administrador se conecta a la IP pública del firewall, y desde ahí accede a la VLAN 20 (management) para administrar los nodos por SSH. A diferencia de Tailscale, WireGuard en OPNsense no depende de servidores externos y es gestionado completamente por la infraestructura propia.

## 16. Protocolos utilizados en el sistema

| **Protocolo**           | **Puerto** | **Uso en el proyecto**                                                                                                                    |
| ----------------------- | ---------- | ----------------------------------------------------------------------------------------------------------------------------------------- |
| HTTPS / TLS             | 443/TCP    | Trafico cifrado entre clientes externos y Traefik en la DMZ. Certificado gestionado por Cloudflare.                                       |
| HTTP                    | 80/TCP     | Redirección automatica a HTTPS. Traefik redirige todo el trafico HTTP entrante.                                                           |
| WebSocket (WSS)         | 443/TCP    | Comunicación bidireccional en tiempo real entre el servidor y la aplicación móvil para actualizaciones de estado de plazas.               |
| REST / JSON             | 8000/TCP   | API HTTP de FastAPI para operaciones sobre plazas, usuarios, reservas y eventos.                                                          |
| SSH                     | 22/TCP     | Acceso administrativo a los nodos. Solo por clave publica ED25519, sin autenticación por contraseña.                                      |
| WireGuard               | 51820/UDP  | Túnel VPN cifrado para acceso remoto del equipo administrador.                                                                            |
| IEEE 802.1Q VLAN        | N/A        | Etiquetado de tramas Ethernet para segmentación lógica de la red en el switch HP.                                                         |
| DHCP                    | 67-68/UDP  | Asignación automática de IPs en la WAN (desde el router de la escuela a OPNsense).                                                        |
| DNS                     | 53/UDP-TCP | Resolución de nombres. DNS público (8.8.8.8) para los nodos. Cloudflare gestiona el DNS público del proyecto.                             |
| ICMP                    | N/A        | Diagnóstico de conectividad (ping) entre nodos, VLANs y firewall durante la configuración y verificación.                                 |
| HTTP (switch)           | 80/TCP     | Acceso a la interfaz web de administración del switch HP 1810-24G.                                                                        |
| pg\_notify (PostgreSQL) | 5432/TCP   | Notificaciones internas de PostgreSQL. Los triggers publican eventos de cambio de estado de plazas que el backend consume en tiempo real. |
| MQTT                    | 1883/TCP   | Protocolo ligero de mensajería IoT utilizado por el sensor ESP32 para enviar eventos de detección al endpoint publico.                    |
| rsync / SSH             | 22/TCP     | Transferencia eficiente de datos para el backup programado. Solo activo durante la ventana horaria nocturna.                              |

### 12. Acceso inicial a la interfaz de OPNsense

Para acceder a la interfaz gráfica de administración de OPNsense fue necesario asignar manualmente una dirección IP en la máquina de gestión dentro del rango de la red LAN del firewall, concretamente 192.168.1.5, con gateway 192.168.1.1. Una vez establecida esta conectividad básica, se accedió a la interfaz web desde el navegador en https://192.168.1.1. OPNsense utiliza un certificado SSL autofirmado, por lo que es necesario aceptar la advertencia del navegador para continuar.

Durante este proceso se identificó que la red LAN del firewall estaba asociada a la VLAN 1, que es la VLAN por defecto en el switch HP 1810-24G. Por tanto, el puerto del switch al que se conectaba la máquina de gestion debia estar configurado como puerto de acceso (Untagged) dentro de dicha VLAN 1. Esto es imprescindible para dispositivos que no gestionan VLANs de forma nativa, como los equipos con Windows utilizados durante la fase de configuración.

#### 12.1 Adaptación del puerto de management para VLAN 20

La VLAN 20 fue definida como red de management, destinada al acceso SSH a los nodos y a la ejecución de playbooks de Ansible. Durante la configuración se detectó que el puerto del switch asignado a la maquina de gestion habia sido configurado inicialmente como Tagged en la VLAN 20. Esto impedía la comunicación, ya que los sistemas Windows no gestionan el etiquetado 802.1Q de forma nativa sin configuraciones adicionales específicas, a diferencia de los nodos Ubuntu, que disponen de subinterfaces VLAN configuradas mediante Netplan.

Como solución, se reconfiguró ese puerto como Untagged en la VLAN 20, eliminando su pertenencia a la VLAN 1. Con este cambio, la máquina obtuvo conectividad dentro del rango 192.168.20.x y fue posible establecer conexión SSH con los nodos del clúster.

Fuente: IEEE 802.1Q-2018, Bridges and Bridged Networks --- https://standards.ieee.org/ieee/802.1Q

#### 12.2  Configuración de NAT de salida

Una vez establecida la conectividad en las VLANs, se verifico el estado del NAT en OPNsense desde Firewall > NAT > Outbound. El modo activo era Automatic outbound NAT rule generation, que genera automáticamente reglas de traducción de direcciones para las subredes conocidas por el sistema. Sin embargo, se añadió una regla manual explícita para la VLAN 10 (cloud) con el objetivo de garantizar cobertura completa:

* Interface: WAN
* Protocol: any
* Source: 192.168.10.0/24
* Destination: any
* Translation: Interface address

Esta regla permite que el tráfico originado en los nodos del cluster MicroCloud salga a internet con la IP de la interfaz WAN como origen visible. La VLAN 50 (OVN uplink) no requiere NAT ya que no tiene dirección IP asignada y es gestionada directamente por MicroOVN.

#### 12.3 Reglas de firewall por interfaz VLAN

OPNsense aplica una politica de denegacion implicita en todas las interfaces OPT, lo que significa que aunque el routing y el NAT esten correctamente configurados, el trafico es descartado por defecto si no existe una regla de paso explicita. Este comportamiento fue identificado durante las pruebas de conectividad cuando los pings hacia 8.8.8.8 no obtenian respuesta pese a que la configuracion de red era correcta.

Para resolver esto, se crearon reglas de paso en cada interfaz VLAN desde Firewall > Rules > \[interfaz]. La configuración aplicada en cada caso fue idéntica en estructura: accion Pass, direction in, protocolo any, origen la red de la propia interfaz (OPT net) y destino any. Las reglas se aplicaron sobre OPT1 (VLAN 10) y OPT2 (VLAN 20). Las reglas de VLAN 30 (DMZ) y VLAN 40 (backup con ventana horaria programada) se configuran en una fase posterior del proyecto junto con Firewall Schedules.

Es importante tener en cuenta que las reglas de OPNsense se evaluan por orden de primera coincidencia: la primera regla que coincide con un paquete se aplica y el resto se ignora. Por ello, el orden de las reglas tiene impacto directo en el comportamiento del firewall.

#### 12.4 Validacion de conectividad con internet

Tras aplicar la configuración de NAT y las reglas de firewall, se realizaron pruebas de conectividad desde los nodos ejecutando ping hacia 8.8.8.8 y hacia google.com. Los resultados fueron satisfactorios en todos los nodos, confirmando que el trafico seguia el flujo esperado: los nodos envian paquetes hacia el gateway de su VLAN en el firewall, OPNsense permite el trafico mediante las reglas configuradas, aplica la traduccion de direcciones NAT y reenvfa el trafico hacia la WAN con conexion a internet de la red del centro educativo.

<br>

<figure><img src="../.gitbook/assets/Screenshot 2026-04-09 at 3.16.02 pm.png" alt=""><figcaption></figcaption></figure>
