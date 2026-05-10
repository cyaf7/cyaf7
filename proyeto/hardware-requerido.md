# hardware requerido

## 1.1 Hardware Requerido - Especificaciones Completas

### Visión General

La infraestructura de Spotly se compone de tres capas de hardware:

1. **Capa de Cluster** (3 nodos MicroCloud)
2. **Capa de Red** (OPNsense firewall + Switch HP)
3. **Capa de Soporte** (PC administrativo, PC respaldo)

***

### Nodos de Cluster (3 máquinas idénticas)

#### Especificaciones Técnicas

| Aspecto               | Especificación                            |
| --------------------- | ----------------------------------------- |
| **Nombre**            | nodo1, nodo2, nodo3                       |
| **Sistema Operativo** | Ubuntu Server 22.04.5 LTS                 |
| **Kernel**            | Linux 5.15.x                              |
| **CPU**               | 8 cores @ 2.4 GHz (Intel Xeon o AMD EPYC) |
| **RAM**               | 16 GB DDR4 (mínimo 14 GB disponible)      |
| **Almacenamiento**    | 500 GB SSD (mínimo 350 GB libres post-OS) |
| **NIC**               | 1x Gigabit Ethernet (enp1s0)              |
| **Virtualización**    | CPU con soporte VT-x/AMD-V                |

#### Descripción Detallada

**CPU (8 cores)**

* Necesario para ejecutar LXD hypervisor
* Cada VM consume 2-4 cores según carga
* 8 cores permite \~3-4 VMs medianas simultáneamente
* El factor de replicación Ceph (3x) requiere CPU para replicación

**RAM (16 GB)**

* 2-3 GB: sistema operativo base
* 8-10 GB: MicroCloud (LXD, MicroCeph, MicroOVN)
* 3-4 GB: headroom para operaciones pico
* Si usas <14 GB libres, Ceph entra en HEALTH\_WARN

**Almacenamiento (500 GB SSD)**

* 20 GB: partición /root (Ubuntu)
* 350 GB mínimo: para MicroCeph OSDs
* 80 GB: espacio de trabajo (logs, temporal)
* SSD es crítico: Ceph necesita latencia <5ms

**NIC Única (enp1s0)**

* Enrutada a VLAN 10 (intra-cluster) via switch
* Enrutada a VLAN 20 (management) via switch
* Enrutada a VLAN 50 (OVN uplink) via switch
* Toda segmentación VLAN ocurre en el switch HP

#### Preparación del Nodo

Después de instalar Ubuntu 22.04.5:

```bash
# Verificar requisitos
lscpu | grep "CPU family\|Model name\|CPU(s)"
free -h | head -2
df -h / | tail -1
ethtool enp1s0

# Actualizar sistema
sudo apt update && sudo apt upgrade -y

# Instalar herramientas básicas
sudo apt install -y \
  curl wget git net-tools \
  snapd snapcraft \
  vim nano htop iotop \
  openssh-server openssh-client
```

***

### Firewall OPNsense

#### Especificaciones Técnicas

| Aspecto               | Especificación                     |
| --------------------- | ---------------------------------- |
| **Sistema Operativo** | OPNsense 26.1.2\_5 (FreeBSD 13.2)  |
| **CPU**               | 2 cores @ 2.0 GHz                  |
| **RAM**               | 4 GB DDR4                          |
| **Almacenamiento**    | 32 GB SSD                          |
| **Interfaz WAN**      | em0 (NIC integrada, 1 Gbps)        |
| **Interfaz LAN**      | ue0 (USB Realtek RTL8153, 1 Gbps)  |
| **Propósito**         | Router central, NAT, IDS, Firewall |

#### ¿Por qué OPNsense?

**Alternativas evaluadas:**

* **pfsense**: Similar, pero menos desarrollo reciente
* **Vyatta/EdgeRouter**: Menos user-friendly
* **MikroTik**: Propietario, menos documentación
* **iptables puro**: Demasiado manual, sin UI

**Elegido OPNsense porque:**

* Open-source basado en FreeBSD
* Web UI moderna (Restful API)
* Soporte completo para VLANs 802.1Q
* IDS/IPS con Suricata integrado
* VPN (WireGuard, OpenVPN)
* Firewall Schedules (para ventanas de backup)
* Actualización fácil vía web

#### Funciones Principales

**1. Router entre VLANs**

* Enrutamiento L3 entre VLAN 10, 20, 30, 40, 50
* Interfaz ue0 (USB) como uplink del switch

**2. NAT Outbound**

* VLAN 10 (cluster) → internet vía WAN (em0)
* Necesario para que nodos accedan a internet

**3. Firewall (DROP by default)**

* OPT1 (VLAN 10): permite intra-cluster + outbound
* OPT2 (VLAN 20): permite SSH a nodos + outbound
* OPT3 (VLAN 30): permite dmz → app-core + outbound
* OPT4 (VLAN 40): permite backup SOLO en ventana (02:30-04:00)

**4. IDS con Suricata**

* Detecta ataques de red
* Bloquea patrones maliciosos

**5. VPN**

* WireGuard (en construcción, reemplazará Tailscale)
* OpenVPN (alternativa)

#### Interfaz WAN (em0)

```
Real Network (escuela)
         ↓
    Router escuela (DHCP)
         ↓
    Cable RJ45
         ↓
    OPNsense em0 (DHCP client)
         ↓
    IP dinámico: 192.168.109.x
```

**Características:**

* DHCP automático del router escuela
* Acceso a internet público
* Gateway a redes externas

#### Interfaz LAN (ue0)

```
USB Realtek RTL8153
         ↓
OPNsense ue0 (192.168.1.1/24)
         ↓
Subinterfaces VLAN:
  - vlan0.10 (VLAN 10 tag) → 192.168.10.1/24
  - vlan0.20 (VLAN 20 tag) → 192.168.20.1/24
  - vlan0.30 (VLAN 30 tag) → 192.168.30.1/24
  - vlan0.40 (VLAN 40 tag) → 192.168.40.1/24
         ↓
    Trama Ethernet etiquetada (802.1Q)
         ↓
    Cable RJ45
         ↓
    Switch HP 1810-24G
```

**¿Por qué USB?**

* Servidor no tiene segundo NIC físico
* Realtek USB 3.0 → Gigabit soporta VLAN trunking
* Suficiente ancho de banda para operaciones

#### Instalación de OPNsense

1. Descargar ISO desde opnsense.org
2. Quemar en USB con Balena Etcher
3. Conectar em0 a router escuela (DHCP)
4. Conectar ue0 al switch HP
5. Bootear desde USB
6. Configurar VLAN subinterfaces via web UI (192.168.1.1:443)

***

### PC Administrativo

#### Especificaciones Técnicas

| Aspecto               | Especificación                             |
| --------------------- | ------------------------------------------ |
| **Sistema Operativo** | Ubuntu Desktop 22.04 LTS                   |
| **CPU**               | 4 cores (suficiente para SSH, LDAP, Wazuh) |
| **RAM**               | 8 GB (mínimo 4 GB)                         |
| **VLAN**              | 20 (Management)                            |
| **IP Estática**       | 192.168.20.20                              |
| **Servicios**         | OpenLDAP (slapd), Ansible                  |

#### Rol en la Infraestructura

**1. Servidor LDAP Centralizado**

* Base DN: dc=spotly,dc=local
* Usuarios: cami, chris, ivan, prof
* SSSD en nodos se autentica contra este servidor
* SSH de nodos usa LDAP + public keys

**2. Host Ansible**

* Almacena playbooks
* Ejecuta configuración en nodos
* Conexión SSH hacia 192.168.20.11/12/13

**3. Punto de Control**

* SSH a nodos para troubleshooting
* Acceso a LXD UI remoto (via SSH tunnel)



### PC Respaldo

#### Especificaciones Técnicas

| Aspecto               | Especificación              |
| --------------------- | --------------------------- |
| **Sistema Operativo** | Ubuntu Server 22.04         |
| **CPU**               | 2 cores (solo rsync)        |
| **RAM**               | 4 GB                        |
| **Almacenamiento**    | 2-4 TB HDD (para respaldos) |
| **VLAN**              | 40 (Backup)                 |
| **IP Estática**       | 192.168.40.10               |
| **Servicio**          | rsync daemon                |

#### Rol

**Servidor de Respaldo Pasivo**

* Recibe rsync desde data (PostgreSQL) VM
* Recibe rsync desde app-core (configuración)
* Recibe rsync desde vision (detector configs)
* Activo SOLO durante ventana 02:30-04:00 (Firewall Schedule)

**Almacenamiento Secundario**

* Copia de PostgreSQL íntegra
* Copia de código de producción
* Punto de recuperación ante desastre

***

### Switch HP 1810-24G

#### Especificaciones Técnicas

| Aspecto                 | Especificación       |
| ----------------------- | -------------------- |
| **Modelo**              | HP 1810-24G (J9801A) |
| **Puertos**             | 24x Gigabit Ethernet |
| **Estándar**            | IEEE 802.1Q (VLAN)   |
| **VLANs Soportadas**    | 1-4094               |
| **Velocidad Backplane** | 48 Gbps              |
| **Interfaz Gestión**    | Web UI, SSH          |
| **Dirección IP**        | 192.168.2.10         |

#### Distribución de Puertos

```
Puerto 1-5, 8-9, 12-18, 20-22, 24: Disponibles (sin usar)
Puerto 4: Ubuntu desktop (VLAN 20, Untagged, para configurar el switch )
Puerto 6: Admin PC (VLAN 20, Tagged)
Puerto 7: Node2 (VLAN 10,20, Tagged)
Puerto 10: Laptop (para configuraciones, no hace parte de la infraestrutura) (VLAN 20, Tagged)
Puerto 11: Node1 (VLAN 10,20, Tagged)
Puerto 19: Node3 (VLAN 10,20, Tagged)
Puerto 23: OPNsense ue0 (VLAN 10,20,30,40, Tagged)
```

#### VLAN Tagging Explicado

**Tagged (T)**:

* Puerto transporta tramas CON etiqueta 802.1Q
* El dispositivo entiende VLANs (requiere configuración de subinterfaces)
* Ejemplo: enp1s0.10 en nodo extrae VLAN 10 automáticamente

**Untagged (U)**:

* Puerto transporta tramas SIN etiqueta
* Acceso directo a una VLAN (para devices sin soporte VLAN)
* Ejemplo: Windows en P4 obtiene VLAN 20 transparentemente

**Excluded (E)**:

* Puerto NO pertenece a esa VLAN
* Tráfico bloqueado en switch level

***

### Adaptador USB Realtek RTL8153

#### Especificaciones Técnicas

| Aspecto       | Especificación               |
| ------------- | ---------------------------- |
| **Interfaz**  | USB 3.0                      |
| **Salida**    | RJ45 Gigabit Ethernet        |
| **Velocidad** | 1000 Mbps full duplex        |
| **Propósito** | LAN interface OPNsense (ue0) |

#### ¿Por qué USB?

**Contexto:**

* Servidor OPNsense no tiene segundo NIC físico
* Necesita MÍNIMO 2 interfaces (WAN + LAN)
* Solución: em0 (integrado) para WAN, USB para LAN

**Ventajas:**

* Gigabit full-duplex suficiente
* Soporte para VLAN trunking en Linux/FreeBSD
* Bajo costo
* Conecta al switch HP

**Latencia:**

* USB 3.0 → RJ45 introduce <1ms latencia
* Despreciable para redes internas

***

### Cámara USB&#x20;

#### Especificaciones Técnicas

| Aspecto               | Especificación                  |
| --------------------- | ------------------------------- |
| **Modelo**            | ELP-USBFHD01M-BL170 (ELP brand) |
| **Resolución Nativa** | 1920x1080p (Full HD)            |
| **Ángulo**            | 170° fisheye                    |
| **FPS Nativo**        | 30 FPS                          |
| **Interfaz**          | USB 2.0 Hi-Speed                |
| **Ubicación**         | MacBook de Ivan (en campo, 4G)  |

#### Rol

**Captura de Imágenes**

* Toma fotos del aparcamiento en tiempo real
* Envía via script spotly\_cam.py (30 FPS)
* Resolución reducida a 960x540 para ancho de banda
* JPEG quality 80 (balance calidad/tamaño)

#### Procesamiento

```
Cámara USB (1920x1080 30 FPS)
         ↓
spotly_cam.py (MacBook):
  - Captura frame
  - Redimensiona a 960x540
  - Comprime JPEG (quality 80)
  - ~50 KB por frame
         ↓
POST https://app.spotly.cat/stream/push
         ↓
App-core guarda en frame_buffer
         ↓
vision VM ejecuta YOLO
```

***

### Periféricos Adicionales

#### Disco Externo SanDisk 1 TB

* Propósito: Backup externo (segunda capa de respaldo)
* Conexión: USB 3.0
* Ubicación: Almacenado en laboratorio
* Procedimiento: Conexión manual periódica

#### Pendrives USB

* Instalación de sistemas operativos&#x20;

