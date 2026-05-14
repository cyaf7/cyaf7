# Switch HP 1810-24G - Configuración y Gestión

## OPNsense — Firewall y router central

### Qué es OPNsense y por qué se usa

OPNsense es una distribución de firewall y router de código abierto basada en FreeBSD, desarrollada por Deciso B.V. Utiliza el sistema de filtrado de paquetes `pf` de FreeBSD y ofrece una interfaz web completa para gestionar todas las funciones de red desde un único punto.

En el proyecto Spotly, OPNsense actúa como el núcleo de toda la red. Todo el tráfico que necesita cruzar de una VLAN a otra pasa obligatoriamente por él, que aplica las reglas de seguridad antes de permitir o denegar el paso. Sin OPNsense correctamente configurado, la segmentación de red no tiene ningún efecto real — los dispositivos podrían comunicarse libremente entre segmentos sin ningún control.

Además del enrutamiento entre VLANs, OPNsense gestiona el NAT de salida para que los dispositivos internos puedan acceder a internet, las reglas de firewall por interfaz, y los Firewall Schedules que controlan el acceso temporal de la VLAN de backup.

### Interfaces físicas

OPNsense dispone de dos interfaces físicas con roles distintos.

La interfaz `em0` es la NIC integrada del equipo y actúa como WAN. Está conectada directamente a la red de la escuela y recibe una IP dinámica por DHCP. Es la salida a internet del sistema y no se utiliza para comunicación interna entre dispositivos del proyecto.

La interfaz `ue0` es un adaptador USB Ethernet Realtek que actúa como LAN. Está conectada al switch HP 1810-24G y es el enlace principal entre OPNsense y toda la infraestructura interna. Sobre esta interfaz se crean todas las subinterfaces VLAN.

La necesidad de un adaptador externo no estaba planificada desde el inicio. El equipo destinado al firewall solo disponía de una NIC física, lo que impedía separar WAN y LAN correctamente. La adquisición del adaptador USB fue la solución que permitió continuar con la arquitectura prevista.

### Subinterfaces VLAN

Sobre la interfaz `ue0` se crean subinterfaces virtuales, una por cada VLAN activa en el sistema. Cada subinterfaz tiene asignada una dirección IP estática que actúa como gateway para los dispositivos de esa VLAN.

Cuando una trama Ethernet llega a `ue0` con una etiqueta VLAN 802.1Q, OPNsense la dirige a la subinterfaz correspondiente según el identificador de VLAN. Esto permite que un único cable físico transporte el tráfico de múltiples redes lógicas simultáneamente, con cada una completamente aislada de las demás a nivel de capa 2.

Las subinterfaces configuradas y sus gateways son los siguientes. La VLAN 10 (cloud) tiene gateway en 192.168.10.1/24 y gestiona el tráfico intra-cluster de MicroCloud. La VLAN 20 (management) tiene gateway en 192.168.20.1/24 y es la red de administración SSH y Ansible. La VLAN 40 (backup) tiene gateway en 192.168.40.1/24 y gestiona el tráfico de copias de seguridad con acceso temporalmente restringido.

### Distinción critica: Trunk Configuration vs VLAN Tagging

Durante el proceso de configuración se identificó una confusión que consumió una cantidad significativa de tiempo de trabajo: la sección Trunk Configuration del HP 1810-24G no hace referencia al concepto de VLAN trunk comúnmente utilizado en redes 802.1Q. En este switch específico, Trunk Configuration corresponde exclusivamente a LACP (Link Aggregation Control Protocol), que es la técnica de combinar múltiples cables físicos para formar un único enlace lógico más rápido y con redundancia.

El VLAN trunk en terminología 802.1Q, es decir, un único cable físico que transporta múltiples VLANs etiquetadas simultáneamente, se configura en este switch exclusivamente a través de VLANs > Participation/Tagging, asignando el modo Tagged (T) al puerto correspondiente para cada VLAN. Esta distinción no está claramente documentada en la interfaz del switch y fue la causa principal de varios intentos fallidos de configuración.

### NAT de salida

Para que los dispositivos de las VLANs internas puedan acceder a internet, OPNsense aplica NAT de salida, traduciendo las IPs privadas a la IP pública de la interfaz WAN antes de enviar el tráfico hacia el exterior.

Se configuró una regla manual explícita para la VLAN 10 desde Firewall > NAT > Outbound, especificando la red 192.168.10.0/24 como origen y la dirección de la interfaz `em0` como traducción. Esto garantiza que el tráfico del cluster salga correctamente a internet para descargas de imágenes, actualizaciones y comunicaciones externas.

### Reglas de firewall

OPNsense aplica por defecto una política de denegación implícita en todas las interfaces OPT: cualquier tráfico que no coincida con una regla de paso explícita es descartado sin aviso. Este comportamiento fue responsable de varios fallos de conectividad durante la fase de configuración, cuando los pings no respondían pese a que el routing y el NAT estaban correctos. La causa era simplemente la ausencia de reglas de paso en las interfaces VLAN.

Las reglas se configuran por interfaz desde Firewall > Rules. Para la VLAN 10 se definieron dos reglas: una que permite la comunicación intra-VLAN entre los nodos del cluster, y otra que permite el tráfico de salida hacia internet. El acceso desde VLAN 10 hacia VLAN 20 queda bloqueado por la política implícita.

Para la VLAN 20 se definió una regla que permite SSH desde la red de management hacia la VLAN 10, y otra que permite el tráfico de salida general. Esto posibilita que el Admin PC y los playbooks de Ansible accedan a los nodos por SSH sin que VLAN 20 tenga acceso irrestricto al resto de la infraestructura.

Para la VLAN 40 se configuraron reglas con Firewall Schedules asociados. El servidor de backup solo puede acceder por SSH a las redes de datos durante la ventana nocturna definida. Fuera de ese horario, el tráfico desde VLAN 40 es bloqueado completamente, de modo que el servidor de backup no tiene conectividad con el resto del sistema durante el resto del día.

### Tabla de enrutamiento resultante

Una vez aplicada toda la configuración, la tabla de enrutamiento refleja la estructura de red del sistema. La ruta por defecto sale por `em0` hacia el gateway de la escuela. Las rutas hacia 192.168.10.0/24, 192.168.20.0/24 y 192.168.40.0/24 son gestionadas localmente por las subinterfaces correspondientes de `ue0`.

### Backup de configuración

OPNsense permite exportar toda su configuración como un fichero XML desde System > Backup & Restore. Este fichero contiene todas las interfaces, VLANs, reglas de firewall, NAT y schedules configurados. En caso de fallo del equipo o reinstalación, basta con restaurar ese fichero para recuperar el estado completo del sistema sin tener que reconfigurar nada manualmente.

### Especificaciones Técnicas

| Característica         | Valor                                   |
| ---------------------- | --------------------------------------- |
| Modelo                 | HP 1810-24G                             |
| Puertos                | 24x Gigabit Ethernet (10/100/1000 Mbps) |
| Estándar               | IEEE 802.3 (Ethernet), 802.1Q (VLAN)    |
| VLANs Soportadas       | 1-4094                                  |
| Velocidad de Backplane | 48 Gbps                                 |
| Throughput Máximo      | 35.7 Mpps (mega packets per second)     |
| Interfaz de Gestión    | Web UI (HTTP/HTTPS), SSH, Telnet        |
| Dirección IP Gestión   | 192.168.2.10/24                         |

### Acceso a la Web UI

#### Primera Configuración

1. Cable Ethernet desde laptop a cualquier puerto del switch
2. Configurar IP laptop: 192.168.2.x (mismo rango que switch por defecto)
3. Abrir navegador: https://192.168.2.10

Network > IP Configuration

```
Method: Static IP
IPv4 Address: 192.168.2.10
Subnet Mask: 255.255.255.0
Default Gateway: 192.168.2.1 (OPNsense)
DNS Servers: 8.8.8.8, 8.8.4.4
```

***

### Conceptos Fundamentales

#### Port vs VLAN vs Trunk

**PORT** (Puerto):

* Interfaz física (P1-P24)
* Puede transportar múltiples VLANs
* Ej: P11 transporta VLAN10, VLAN20, VLAN50

**VLAN** (Virtual LAN):

* Red lógica aislada
* Definida por etiqueta 802.1Q
* Ej: VLAN 10 = red de cluster

**TRUNK** (en HP) = Aggregation:

* LACP Link Aggregation (NO VLAN trunking)
* Combina múltiples puertos físicos en 1 enlace lógico
* **NO lo usamos en Spotly**

#### Tagged vs Untagged vs Excluded (CRÍTICO)

**Tagged (T)**:

```
Switch → Nodo:
  Trama: [...| VLAN 10 Tag | Payload |...]
                    |
         Nodo ve: vlan10@enp1s0
         
Nodo → Switch:
  Si envía en vlan10: trama va CON tag VLAN10
  Switch ve etiqueta 10: ¿puerto P7 tiene VLAN10 T? SÍ → PASS
```

Requiere: dispositivo entienda 802.1Q

**Untagged (U)**:

```
Switch → Dispositivo:
  Trama: [...| SIN etiqueta | Payload |...]
               |
         Dispositivo ve: acceso directo a esa VLAN
         Transparente: no sabe que está en VLAN
         
Ejemplo: Windows PC en P4 (U de VLAN 20)
  → Obtiene IP 192.168.20.x
  → Cree que está en "la red normal"
  → En realidad está en VLAN 20
```

Requiere: dispositivo NO entienda 802.1Q

**Excluded (E)**:

```
Switch -> Dispositivo:
  Trama con VLAN 30 etiqueta:
    Puerto tiene VLAN30=E
     Trama es DESCARTADA
     Conexión NO existe
```

***

### Configuración de Puertos

#### Port Configuration

Interfaces > Port Settings

```
Para CADA puerto:
  State: Enabled
  Speed/Duplex: Auto (recommended)
  Flow Control: Enabled
  BPDU Guard: Disabled (no Spanning Tree)
  Port Rate Limit: Unlimited
```

#### Puerto P1-P5, P8-P9, P12-P18, P20-P22, P24 (NO USADOS)

```
State: Disabled
(Desactivar puertos no usados reduce consumo y surface de ataque)
```

#### Puerto P4 - Windows PC (Temporal)

```
VLAN Participation:
  VLAN 1: Excluded
  VLAN 10: Excluded
  VLAN 20: Untagged <- Acceso directo a VLAN 20
  VLAN 40: Excluded
```

**Resultado**: Windows obtiene IP 192.168.20.x automáticamente (DHCP desde OPNsense)

#### Puerto P6 - Admin PC

```
VLAN Participation:
  VLAN 10: Excluded
  VLAN 20: Tagged <- Admin PC entiende 802.1Q
  VLAN 40: Excluded
```

**Resultado**: Admin PC puede configurar subinterfaz vlan20 para acceder VLAN 20

#### Puerto P7 - Node2

```
VLAN Participation:
  VLAN 10: Tagged <- Nodo entiende 802.1Q
  VLAN 20: Tagged
  VLAN 40: Excluded
```

**Resultado**: Nodo recibe 2 etiquetas: 10, 20. Netplan crea subinterfaces.

#### Puerto P11 - Node1

Igual que P7 (idéntico)

#### Puerto P19 - Node3

Igual que P7 (idéntico)

#### Puerto P10 - Laptop (Temporal)

```
VLAN Participation:
  VLAN 20: Tagged
  Resto: Excluded
```

#### Puerto P23 - OPNsense (TRUNK VLAN)

**Este es el más importante**: OPNsense es el router central, debe transportar TODAS las VLANs

```
VLAN Participation:
  VLAN 10: Tagged <- Cluster
  VLAN 20: Tagged <- Management
  VLAN 40: Tagged <- Backup

Port Settings:
  State: Enabled
  Speed: Auto
  Flow Control: Enabled
```

**Resultado**: OPNsense ve todas las VLANs etiquetadas, puede enrutarlas.

***

### Tabla de Participación de Puertos (Resumen)

```
Puerto  Dispositivo    VLAN10  VLAN20   VLAN40  
────────────────────────────────────────────────
P4      Windows        E       U         E       
P6      Admin PC       E       T         T       
P7      Node2          T       T         E       
P10     Laptop         E       T         E       
P11     Node1          T       T         E       
P19     Node3          T       T         E       
P23     OPNsense       T       T         T       

Leyenda:
T = Tagged (etiquetado, dispositivo entiende VLAN)
U = Untagged (sin etiquetar, acceso directo)
E = Excluded (excluido, sin acceso)
```

***

### Configuración de VLANs

#### VLAN Management

System > VLAN > VLAN Configuration

**VLAN 10 (Cluster)**

```
VLAN ID: 10
VLAN Name: cluster
VLAN Status: Active
Port Tagging:
  P7: T, P11: T, P19: T, P23: T
```

**VLAN 20 (Management)**

```
VLAN ID: 20
VLAN Name: management
VLAN Status: Active
Port Tagging:
  P4: U, P6: T, P7: T, P11: T, P19: T, P10: T, P23: T
```

**VLAN 40 (Backup)**

```
VLAN ID: 40
VLAN Name: backup
VLAN Status: Active
Port Tagging:
  P23: T
  (Todos los demás: E)
```

***

### Spanning Tree (Desactivado)

Sistema > Spanning Tree > Global Configuration

```
Spanning Tree: Disabled
(No necesitamos redundancia de switch, solo 1)
```

***

### Port Mirror / Port Monitoring (Opcional)

Para debugging de tráfico:

Monitoring > Port Monitor

```
Monitor Port: P10 (laptop)
Monitored Port: P7 (Node2)
Direction: Both

(Laptop verá TODAS las tramas de Node2)
```

#### Test de Conectividad Física

Desde Admin PC:

```bash
# Verifica que los nodos están en diferentes VLANs
ping 192.168.10.11  # Node1 (VLAN 10)
ping 192.168.10.12  # Node2 (VLAN 10)
ping 192.168.10.13  # Node3 (VLAN 10)

# Todos deberían responder (mismo broadcast domain)

# Verifica SSH desde VLAN 20
ssh ubuntu@192.168.20.11  # Node1 (VLAN 20)
ssh ubuntu@192.168.20.12  # Node2 (VLAN 20)
ssh ubuntu@192.168.20.13  # Node3 (VLAN 20)

```

***

### Operación Diaria

#### Agregar Nuevo Dispositivo

1. Identificar puerto disponible (P1, P2, etc)
2. Determinar VLAN (10=cluster, 20=management, etc)
3. Si dispositivo entiende VLAN: Tagged (T)
4. Si no entiende VLAN: Untagged (U) a UNA sola VLAN
5. Guardar configuración
6. Conectar cable

#### Remover Dispositivo

1. Desconectar cable
2. En switch: marcar puerto como Excluded de todas las VLANs
3. Guardar configuración

### Pruebas de conectividad: proceso real y errores

La configuración de la red fue el proceso más complejo de esta fase del proyecto. Se realizaron tres pruebas de conectividad secuenciales, cada una añadiendo un componente adicional al circuito. A continuación se documenta el proceso tal como ocurrió, incluyendo los errores encontrados, las hipótesis manejadas y las soluciones aplicadas.

#### Prueba 1: Internet a través del firewall hacia una computadora

El objetivo de esta primera prueba era verificar que OPENsense estaba correctamente instalado y configurado, que la interfaz WAN recibió conexion de internet desde la red de la escuela, y que la interfaz LAN distribuye correctamente esa conexión a los dispositivos conectados.

La prueba consistió en conectar un cable desde el puerto de red de la escuela a la interfaz WAN del firewall (em0, NIC integrada), y otro cable desde la interfaz LAN (ue0, adaptador USB Realtek) directamente a una computadora, sin switch de por medio.

En el primer intento, la computadora no obtenía ninguna IP y no había conectividad. Tras revisar las conexiones físicas, se identificó que el cable utilizado para la conexión WAN era defectuoso. Al sustituirlo por un cable en buen estado, la computadora obtuvo la IP 192.168.1.149 por DHCP desde la LAN de OPNsense, y el firewall recibio correctamente una IP del rango de la escuela (192.168.109.x) en su interfaz WAN. La prueba se dio por superada.

#### Prueba 2: Internet a través del switch sin configuración VLAN

El objetivo de la segunda prueba era verificar que el switch HP 1810-24G pasaba trafico correctamente en su estado por defecto, antes de aplicar ninguna configuración de VLANs. El circuito probado fue: red de la escuela > firewall > switch > computadora.

Al conectar la computadora al switch, se esperaba que obtuviera una IP del rango de la LAN de OPNsense (192.168.1.x). Sin embargo, la computadora obtuvo la IP 192.168.109.49, que pertenecía directamente al rango de la red de la escuela. Esto indicaba que el trafico no estaba pasando por el firewall sino que llegaba directamente desde la red de la escuela.

La investigación reveló que el problema no era el switch, que funcionaba correctamente en modo no gestionado, sino la configuración de interfaces en OPNsense: la asignación de WAN y LAN estaba invertida. La interfaz que debía ser WAN (conectada a la escuela) estaba configurada como LAN, y viceversa. Al corregir la asignación de interfaces en la consola de OPNsense mediante la opción 1 (Assign Interfaces), el tráfico empezó a fluir correctamente a través del firewall.<br>

#### Prueba 3: Circuito completo con configuracion VLAN

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





***

