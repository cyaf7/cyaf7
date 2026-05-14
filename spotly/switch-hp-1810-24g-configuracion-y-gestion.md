# Switch HP 1810-24G - Configuración y Gestión

### OPNsense — Firewall y router central

#### Qué es OPNsense y por qué se usa

OPNsense es una distribución de firewall y router de código abierto basada en FreeBSD, desarrollada por Deciso B.V. Utiliza el sistema de filtrado de paquetes pf de FreeBSD y ofrece una interfaz web completa para gestionar todas las funciones de red desde un único punto.

En el proyecto Spotly, OPNsense actúa como el núcleo de toda la red. Todo el tráfico que necesita cruzar de una VLAN a otra pasa obligatoriamente por él, que aplica las reglas de seguridad antes de permitir o denegar el paso. Sin OPNsense correctamente configurado, la segmentación de red no tiene ningún efecto real — los dispositivos podrían comunicarse libremente entre segmentos sin ningún control.

Además del enrutamiento entre VLANs, OPNsense gestiona el NAT de salida para que los dispositivos internos puedan acceder a internet, las reglas de firewall por interfaz, y los Firewall Schedules que controlan el acceso temporal de la VLAN de backup.

#### Interfaces físicas

OPNsense dispone de dos interfaces físicas con roles distintos.

La interfaz `em0` es la NIC integrada del equipo y actúa como WAN. Está conectada directamente a la red de la escuela y recibe una IP dinámica por DHCP. Es la salida a internet del sistema y no se utiliza para comunicación interna entre dispositivos del proyecto.

La interfaz `ue0` es un adaptador USB Ethernet Realtek que actúa como LAN. Está conectada al switch HP 1810-24G y es el enlace principal entre OPNsense y toda la infraestructura interna. Sobre esta interfaz se crean todas las subinterfaces VLAN.

La necesidad de un adaptador externo no estaba planificada desde el inicio. El equipo destinado al firewall solo disponía de una NIC física, lo que impedía separar WAN y LAN correctamente. La adquisición del adaptador USB fue la solución que permitió continuar con la arquitectura prevista.

#### Subinterfaces VLAN

Sobre la interfaz `ue0` se crean subinterfaces virtuales, una por cada VLAN activa en el sistema. Cada subinterfaz tiene asignada una dirección IP estática que actúa como gateway para los dispositivos de esa VLAN.

Cuando una trama Ethernet llega a `ue0` con una etiqueta VLAN 802.1Q, OPNsense la dirige a la subinterfaz correspondiente según el identificador de VLAN. Esto permite que un único cable físico transporte el tráfico de múltiples redes lógicas simultáneamente, con cada una completamente aislada de las demás a nivel de capa 2.

Las subinterfaces configuradas y sus gateways son los siguientes. La VLAN 10 (cloud) tiene gateway en 192.168.10.1/24 y gestiona el tráfico intra-cluster de MicroCloud. La VLAN 20 (management) tiene gateway en 192.168.20.1/24 y es la red de administración SSH y Ansible. La VLAN 40 (backup) tiene gateway en 192.168.40.1/24 y gestiona el tráfico de copias de seguridad con acceso temporalmente restringido. La VLAN 50 (ovn-uplink) no tiene dirección IP asignada en OPNsense — su función es exclusivamente servir como canal físico para que MicroOVN conecte las redes virtuales internas del cluster con el exterior, y es gestionada directamente por MicroOVN sin intervención del firewall.

#### Distinción crítica: Trunk Configuration vs VLAN Tagging

Durante el proceso de configuración se identificó una confusión que consumió una cantidad significativa de tiempo de trabajo: la sección Trunk Configuration del HP 1810-24G no hace referencia al concepto de VLAN trunk comúnmente utilizado en redes 802.1Q. En este switch específico, Trunk Configuration corresponde exclusivamente a LACP (Link Aggregation Control Protocol), que es la técnica de combinar múltiples cables físicos para formar un único enlace lógico más rápido y con redundancia.

El VLAN trunk en terminología 802.1Q, es decir, un único cable físico que transporta múltiples VLANs etiquetadas simultáneamente, se configura en este switch exclusivamente a través de VLANs > Participation/Tagging, asignando el modo Tagged (T) al puerto correspondiente para cada VLAN. Esta distinción no está claramente documentada en la interfaz del switch y fue la causa principal de varios intentos fallidos de configuración.

#### NAT de salida

Para que los dispositivos de las VLANs internas puedan acceder a internet, OPNsense aplica NAT de salida, traduciendo las IPs privadas a la IP pública de la interfaz WAN antes de enviar el tráfico hacia el exterior.

Se configuró una regla manual explícita para la VLAN 10 desde Firewall > NAT > Outbound, especificando la red 192.168.10.0/24 como origen y la dirección de la interfaz `em0` como traducción. Esto garantiza que el tráfico del cluster salga correctamente a internet para descargas de imágenes, actualizaciones y comunicaciones externas.

#### Reglas de firewall

OPNsense aplica por defecto una política de denegación implícita en todas las interfaces OPT: cualquier tráfico que no coincida con una regla de paso explícita es descartado sin aviso. Este comportamiento fue responsable de varios fallos de conectividad durante la fase de configuración, cuando los pings no respondían pese a que el routing y el NAT estaban correctos. La causa era simplemente la ausencia de reglas de paso en las interfaces VLAN.

Las reglas se configuran por interfaz desde Firewall > Rules. Para la VLAN 10 se definieron dos reglas: una que permite la comunicación intra-VLAN entre los nodos del cluster, y otra que permite el tráfico de salida hacia internet. El acceso desde VLAN 10 hacia VLAN 20 queda bloqueado por la política implícita.

Para la VLAN 20 se definió una regla que permite SSH desde la red de management hacia la VLAN 10, y otra que permite el tráfico de salida general. Esto posibilita que el Admin PC y los playbooks de Ansible accedan a los nodos por SSH sin que VLAN 20 tenga acceso irrestricto al resto de la infraestructura.

Para la VLAN 40 se configuraron reglas con Firewall Schedules asociados. El servidor de backup solo puede acceder por SSH a las redes de datos durante la ventana nocturna definida. Fuera de ese horario, el tráfico desde VLAN 40 es bloqueado completamente, de modo que el servidor de backup no tiene conectividad con el resto del sistema durante el resto del día.

#### Tabla de enrutamiento resultante

Una vez aplicada toda la configuración, la tabla de enrutamiento refleja la estructura de red del sistema. La ruta por defecto sale por `em0` hacia el gateway de la escuela. Las rutas hacia 192.168.10.0/24, 192.168.20.0/24 y 192.168.40.0/24 son gestionadas localmente por las subinterfaces correspondientes de `ue0`. La VLAN 50 no aparece en la tabla de enrutamiento porque no tiene IP asignada.

#### Backup de configuración

OPNsense permite exportar toda su configuración como un fichero XML desde System > Backup & Restore. Este fichero contiene todas las interfaces, VLANs, reglas de firewall, NAT y schedules configurados. En caso de fallo del equipo o reinstalación, basta con restaurar ese fichero para recuperar el estado completo del sistema sin tener que reconfigurar nada manualmente.

***

### Especificaciones Técnicas

| Campo                  | Valor                                   |
| ---------------------- | --------------------------------------- |
| Modelo                 | HP 1810-24G                             |
| Puertos                | 24x Gigabit Ethernet (10/100/1000 Mbps) |
| Estándar               | IEEE 802.3 (Ethernet), 802.1Q (VLAN)    |
| VLANs Soportadas       | 1-4094                                  |
| Velocidad de Backplane | 48 Gbps                                 |
| Throughput Máximo      | 35.7 Mpps                               |
| Interfaz de Gestión    | Web UI (HTTP/HTTPS)                     |
| Dirección IP Gestión   | 192.168.2.10/24                         |

***

### Conceptos Fundamentales

#### Port vs VLAN vs Trunk

**PORT (Puerto):** interfaz física (P1-P24). Puede transportar múltiples VLANs. Por ejemplo, P11 transporta VLAN10, VLAN20 y VLAN50.

**VLAN (Virtual LAN):** red lógica aislada, definida por etiqueta 802.1Q. Por ejemplo, VLAN 10 es la red del cluster.

**TRUNK (en HP) = Aggregation:** corresponde a LACP, Link Aggregation. Combina múltiples puertos físicos en un enlace lógico. No es VLAN trunking y no se usa en Spotly.

#### Tagged vs Untagged vs Excluded

**Tagged (T):** el puerto transporta esa VLAN con etiqueta 802.1Q. Requiere que el dispositivo conectado entienda 802.1Q. Es el modo usado en los nodos, OPNsense y el servidor de backup.

**Untagged (U):** el puerto pertenece a esa VLAN sin etiqueta. El dispositivo conectado no necesita saber que existe una VLAN. Es el modo usado en el Admin PC, ya que Ubuntu Desktop no gestiona etiquetado 802.1Q de forma nativa.

**Excluded (E):** el puerto no participa en esa VLAN en absoluto.

***

### Configuración de Puertos

#### Tabla de participación de VLANs por puerto

| Puerto | Equipo         | VLAN 10 | VLAN 20 | VLAN 40 | VLAN 50 |
| ------ | -------------- | ------- | ------- | ------- | ------- |
| P4     | Admin PC       | E       | U       | E       | E       |
| P7     | nodo02         | T       | T       | E       | T       |
| P11    | nodo01         | T       | T       | E       | T       |
| P12    | Backup Server  | E       | E       | U       | E       |
| P19    | nodo03         | T       | T       | E       | T       |
| P23    | OPNsense (ue0) | T       | T       | T       | T       |

**Puerto P4 — Admin PC:** VLAN 20 en modo Untagged. Ubuntu Desktop no gestiona etiquetado 802.1Q nativamente, por lo que el puerto entrega el tráfico sin etiquetar y el equipo lo recibe como una red normal.

**Puertos P7, P11, P19 — Nodos del cluster:** VLANs 10, 20 y 50 en modo Tagged. La VLAN 10 gestiona el tráfico intra-cluster, la VLAN 20 el acceso SSH y administración, y la VLAN 50 sirve como uplink físico para MicroOVN. Netplan en cada nodo crea subinterfaces virtuales para cada VLAN.

**Puerto P12 — Backup Server:** VLAN 40 en modo Untagged. El servidor de backup está completamente aislado en su propia red y solo tiene acceso al resto de la infraestructura durante la ventana nocturna definida en OPNsense.

**Puerto P23 — OPNsense:** todas las VLANs en modo Tagged. Es el enlace troncal principal entre el switch y el firewall. OPNsense recibe las tramas etiquetadas y las distribuye a cada subinterfaz VLAN correspondiente.

***

### Configuración de VLANs

#### VLANs definidas

| VLAN ID | Nombre     | Subred          | Función                                  |
| ------- | ---------- | --------------- | ---------------------------------------- |
| 10      | cloud      | 192.168.10.0/24 | Tráfico intra-cluster MicroCloud         |
| 20      | management | 192.168.20.0/24 | Administración SSH, Ansible, LDAP, Wazuh |
| 40      | backup     | 192.168.40.0/24 | Copias de seguridad con acceso nocturno  |
| 50      | ovn-uplink | Sin IP          | Uplink físico para MicroOVN              |

La VLAN 50 merece una explicación específica. A diferencia del resto, no tiene dirección IP ni gateway en OPNsense. Su único propósito es proporcionar un canal físico de capa 2 para que MicroOVN pueda conectar las redes virtuales internas del cluster (ovn-app, ovn-dmz, ovn-infra) con el exterior. Durante el `microcloud init`, el sistema detectó la VLAN 50 como la interfaz sin IP disponible y la seleccionó automáticamente como uplink OVN. Todo el tráfico de las VMs que necesita salir del cluster viaja por esta VLAN hasta llegar a OPNsense, donde se aplica NAT y se enruta hacia internet.

***

### Pruebas de conectividad: proceso real y errores

La configuración de la red fue el proceso más complejo de esta fase del proyecto. Se realizaron tres pruebas de conectividad secuenciales, cada una añadiendo un componente adicional al circuito. A continuación se documenta el proceso tal como ocurrió, incluyendo los errores encontrados y las soluciones aplicadas.

#### Prueba 1: Internet a través del firewall hacia una computadora

El objetivo era verificar que OPNsense estaba correctamente instalado y configurado, que la interfaz WAN recibía conexión de internet desde la red de la escuela, y que la interfaz LAN distribuía correctamente esa conexión.

En el primer intento, la computadora no obtenía ninguna IP. Tras revisar las conexiones físicas, se identificó que el cable utilizado para la conexión WAN era defectuoso. Al sustituirlo, la computadora obtuvo la IP 192.168.1.149 por DHCP desde la LAN de OPNsense. La prueba se dio por superada.

#### Prueba 2: Internet a través del switch sin configuración VLAN

El objetivo era verificar que el switch HP 1810-24G pasaba tráfico correctamente en su estado por defecto. La computadora obtuvo la IP 192.168.109.49, que pertenecía directamente al rango de la red de la escuela, lo que indicaba que el tráfico no estaba pasando por el firewall.

La investigación reveló que la asignación de WAN y LAN en OPNsense estaba invertida. Al corregir la asignación de interfaces mediante la opción 1 de la consola de OPNsense (Assign Interfaces), el tráfico empezó a fluir correctamente a través del firewall.

#### Prueba 3: Circuito completo con configuración VLAN

La tercera prueba fue la más extensa y compleja. El objetivo era verificar que las VLANs configuradas tanto en OPNsense como en el switch funcionaban correctamente, permitiendo que los nodos se comunicaran con el gateway del firewall a través de sus respectivas VLANs.

**Error 1: VLANs configuradas sobre la interfaz WAN.** Al crear las interfaces VLAN en OPNsense, se seleccionó `em0`como interfaz padre en lugar de `ue0`. Esto significaba que el tráfico VLAN se intentaba enviar por la interfaz WAN, conectada a internet, en lugar de por la interfaz LAN conectada al switch. El error se corrigió editando cada subinterfaz para cambiar la interfaz padre de `em0` a `ue0`.

**Error 2: Configuración incorrecta de Trunk Configuration.** Durante la configuración del switch se accedió a la sección Trunk Configuration con la intención de habilitar el transporte de múltiples VLANs por el puerto 23. Tras varias horas de pruebas sin resultado, se identificó que Trunk Configuration en el HP 1810-24G corresponde a LACP, no a VLAN trunking. Toda la configuración de Trunk Configuration se eliminó y se configuró el tagging correcto por puerto desde VLANs > Participation/Tagging.

**Error 3: Reglas de firewall bloqueando el tráfico.** OPNsense aplica por defecto una política de deny all en las interfaces OPT. Aunque el routing estuviera correctamente configurado, los pings de los nodos al gateway eran descartados. Para verificar que el problema era la conectividad de red y no las reglas, se deshabilitaron temporalmente las reglas de firewall en las interfaces VLAN. Una vez confirmada la conectividad, las reglas se volvieron a habilitar y se configuraron correctamente.

#### Resultado final

Tras corregir los tres errores, se verificó conectividad completa entre los nodos y el gateway del firewall en la VLAN 10 y la VLAN 20. Los nodos podían hacer ping a 192.168.10.1 y 192.168.20.1, y el firewall respondía correctamente. La segmentación de red quedó operativa.
