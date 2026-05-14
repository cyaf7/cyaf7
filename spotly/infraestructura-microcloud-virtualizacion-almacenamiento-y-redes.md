# Infraestructura MicroCloud: Virtualización, Almacenamiento y Redes

### Introducción

El clúster MicroCloud del proyecto Spotly integra tres componentes desarrollados por Canonical que trabajan de forma conjunta sobre tres nodos físicos Dell: **LXD** para virtualización, **MicroCeph** para almacenamiento distribuido y **MicroOVN** para redes virtuales. Este documento describe el funcionamiento de cada componente, las decisiones de diseño adoptadas y las soluciones aplicadas a las limitaciones del hardware disponible en el laboratorio.

***

### 1. LXD — Virtualización

#### Qué es LXD

LXD es el hipervisor que gestiona el ciclo de vida de las máquinas virtuales del clúster. Utiliza QEMU/KVM como backend de virtualización, lo que permite crear VMs con aislamiento completo de kernel, memoria y dispositivos. A diferencia de los contenedores del sistema (LXC), las VMs ejecutadas con el flag `--vm` disponen de su propio kernel, firmware UEFI y drivers VirtIO.

El equipo eligió VMs en lugar de contenedores para garantizar el aislamiento total entre servicios: si un servicio se ve comprometido, no tiene acceso al sistema operativo del nodo físico ni a otros servicios.

#### Gestión de VMs en el clúster

LXD opera en modo clúster sobre los tres nodos físicos. Esto significa que cualquier nodo puede crear, iniciar o migrar VMs, y que el estado del clúster (qué VMs existen, en qué nodo corren, cuál es su configuración) se comparte entre los tres nodos mediante una base de datos distribuida.

Las VMs activas y su ubicación son las siguientes:

| VM           | Nodo   | IP principal | Servicio                    |
| ------------ | ------ | ------------ | --------------------------- |
| app-core     | nodo03 | 10.10.20.3   | FastAPI · WebSocket         |
| data         | nodo01 | 10.10.20.2   | PostgreSQL 15               |
| vision       | nodo01 | 10.10.20.4   | YOLO · OpenCV               |
| dmz-receiver | nodo03 | 10.10.30.2   | cloudflared · Traefik       |
| monitoreo    | nodo03 | 10.10.10.3   | Prometheus · Grafana · Loki |

#### Acceso a las VMs

El acceso a las VMs no requiere SSH configurado. LXD proporciona un canal interno de comunicación con cada VM a través del nodo físico que la aloja:

```bash
lxc exec app-core -- bash
```

Este comando abre una sesión de shell dentro de la VM con privilegios de root, equivalente a acceso directo por consola.

Para Ansible, el equipo utiliza el método LXC Proxy: Ansible se conecta al nodo físico por SSH (usando la llave Ed25519 del usuario `ansible`) y desde ahí ejecuta comandos dentro de la VM a través de `lxc exec`. Esto permite automatizar la configuración de las VMs sin necesidad de que tengan IP accesible desde la red de gestión.

#### Almacenamiento de las VMs

Los discos de todas las VMs se almacenan en el pool `remote` de tipo Ceph, gestionado por MicroCeph. Esto significa que el disco de cada VM está replicado en los tres nodos simultáneamente. Si el nodo físico que aloja una VM cae, LXD puede arrancarla en cualquier otro nodo sin pérdida de datos, ya que el disco sigue disponible a través de Ceph.

***

### 2. MicroCeph — Almacenamiento Distribuido

#### Qué es MicroCeph

MicroCeph implementa Ceph, el sistema de almacenamiento distribuido de código abierto, en un formato simplificado diseñado para clústeres de tres o más nodos. Ceph divide los datos en bloques y los replica automáticamente entre todos los nodos del clúster. Con el factor de replicación por defecto de 3 y tres nodos, cada bloque de datos existe simultáneamente en los tres nodos.

La diferencia fundamental respecto a RAID tradicional es que la replicación se distribuye entre **máquinas físicas distintas**. Si un nodo cae completamente (fallo de hardware, corte de luz), los datos siguen disponibles en los otros dos nodos sin intervención manual.

#### El problema del hardware disponible

MicroCeph requiere que cada nodo aporte al menos un disco dedicado exclusivamente al almacenamiento. Los ordenadores del laboratorio disponen de un único disco físico por máquina, completamente ocupado por el sistema operativo Ubuntu Server.

El equipo intentó dos soluciones antes de encontrar la definitiva:

**Intento 1: Volúmenes LVM dentro de ubuntu-vg.** Se crearon volúmenes lógicos de 200 GB dentro del Volume Group existente del sistema operativo. MicroCeph los rechazó porque requiere dispositivos de bloque completamente independientes del VG del sistema.

**Solución adoptada: Loop files como OSDs.** Un loop file es un fichero regular almacenado en el disco que el kernel presenta al sistema como si fuera un dispositivo de bloque físico. MicroCeph acepta estos dispositivos como OSDs sin distinción. Esta es la solución oficial recomendada por Canonical para entornos de laboratorio.

#### Implementación de los loop files

En cada nodo se creó un fichero de imagen de 200 GB:

```bash
truncate -s 200G /mnt/ceph-disks/ceph-osd.img
losetup -fP /mnt/ceph-disks/ceph-osd.img
microceph disk add /dev/loopX --wipe
```

Para que los loop devices persistan tras un reinicio, se añadió en `/etc/rc.local` la lógica de montaje con un guard de idempotencia:

```bash
losetup -j /mnt/ceph-disks/ceph-osd.img | grep -q loop || \
  losetup -fP /mnt/ceph-disks/ceph-osd.img
```

El guard evita crear dispositivos duplicados si el fichero ya estaba montado, que fue la causa raíz del fallo de OSD en nodo02.

#### Alta disponibilidad: arquitectura ideal vs entorno de laboratorio

MicroCloud proporciona alta disponibilidad a nivel de datos mediante MicroCeph: con el factor de replicación por defecto de 3, cada bloque de datos existe simultáneamente en los tres nodos del clúster. Si un nodo falla, los datos siguen siendo accesibles desde los otros dos. LXD también permite la migración automática de máquinas virtuales entre nodos cuando uno de ellos queda fuera de servicio.

Sin embargo, existe una distinción importante entre alta disponibilidad a nivel de software y alta disponibilidad a nivel de hardware físico. En un entorno de producción real, cada nodo del clúster debería tener una máquina física idéntica como réplica de respaldo. Si el hardware de un nodo falla completamente, por ejemplo por un fallo de la placa base o de la fuente de alimentación, ese nodo queda fuera del cluster hasta ser reparado o sustituido. Durante ese tiempo, el cluster opera con dos nodos en lugar de tres, lo que reduce la tolerancia a fallos adicionales.

La arquitectura ideal para un despliegue de producción sería la siguiente: por cada nodo del cluster existiría una máquina física idéntica en standby, con el mismo hardware, la misma configuración de red y el mismo software instalado. Si el nodo principal falla, la máquina de standby entra en el cluster automáticamente sin intervención manual, manteniendo el clúster en tres nodos activos en todo momento. Este modelo se denomina N+1 redundancy, donde N es el número de nodos necesarios para operar y el 1 adicional garantiza continuidad ante un fallo.

En este proyecto no fue posible implementar esta arquitectura debido a la disponibilidad limitada de hardware en el laboratorio. Los seis ordenadores Dell disponibles se distribuyeron entre los roles esenciales del sistema: tres nodos de MicroCloud, un firewall, un equipo de administración y uno destinado a backup. No quedaban equipos disponibles para actuar como réplicas físicas de los nodos del clúster. Esta limitación queda documentada como una diferencia conocida respecto a un despliegue de producción real, y no afecta a la validez funcional del sistema en el entorno de laboratorio.

***

### 3. MicroOVN — Redes Virtuales

#### Qué es OVN

OVN (Open Virtual Network) es el sistema de redes virtuales de la comunidad Open vSwitch. MicroOVN integra OVN en el clúster MicroCloud y permite crear redes lógicas completamente aisladas entre máquinas virtuales, independientemente de la topología de red física subyacente.

#### Cómo funciona: switches y routers lógicos

Para cada red OVN que se crea, OVN instancia automáticamente dos componentes lógicos:

**Switch lógico:** conecta todas las VMs dentro de la misma red. Cuando `app-core` envía un paquete a `data`, el switch lógico de `ovn-app` lo entrega directamente sin que el paquete salga del mundo OVN ni pase por OPNsense. El switch lógico funciona en capa 2, igual que un switch físico, pero implementado en software distribuido sobre los tres nodos.

**Router lógico:** conecta las redes OVN entre sí y con el exterior. Tiene una interfaz en cada red y una interfaz en el UPLINK (vlan50) que conecta con OPNsense para el tráfico hacia internet. Cuando una VM en `ovn-app` necesita salir a internet, el paquete va al router lógico, que lo reenvía por el UPLINK hacia OPNsense, que aplica NAT y lo envía a la WAN.

Las ACLs de OVN actúan sobre el switch lógico de cada red, antes de que el paquete llegue al sistema operativo de la VM destino. Esto hace el filtrado más eficiente y más seguro que iptables o nftables dentro de la VM.

#### Redes configuradas en el proyecto

El clúster dispone de tres redes OVN activas, cada una con un propósito funcional diferente:

**ovn-app — 10.10.20.0/24**

Red de la capa de aplicación. Contiene todos los servicios internos de Spotly.

| VM                  | IP         | Servicio                    |
| ------------------- | ---------- | --------------------------- |
| data                | 10.10.20.2 | PostgreSQL 15               |
| app-core            | 10.10.20.3 | FastAPI · WebSocket         |
| vision              | 10.10.20.4 | YOLO · OpenCV               |
| dmz-receiver (eth1) | 10.10.20.6 | Interfaz interna de Traefik |

La particularidad de `dmz-receiver` es que tiene dos interfaces de red: `eth0` en `ovn-dmz` (cara pública) y `eth1` en `ovn-app` (cara interna). Gracias a `eth1`, Traefik puede reenviar peticiones directamente a `app-core` sin salir al exterior ni pasar por OPNsense.

**ovn-dmz — 10.10.30.0/24**

Zona desmilitarizada lógica. Solo contiene la VM `dmz-receiver`.

| VM                  | IP         | Servicio                      |
| ------------------- | ---------- | ----------------------------- |
| dmz-receiver (eth0) | 10.10.30.2 | cloudflared · Traefik · Nginx |

Esta red es el único punto de entrada del tráfico público al clúster. Si `dmz-receiver` fuera comprometido, el atacante quedaría atrapado en `ovn-dmz`, sin acceso directo a la base de datos ni a los nodos físicos. El tráfico entre `ovn-dmz` y `ovn-app` solo está permitido desde `dmz-receiver eth1` hacia `app-core`, nunca en sentido inverso.

**ovn-infra — 10.10.10.0/24**

Red de infraestructura interna, destinada a servicios de monitoreo que nunca deben ser accesibles desde internet.

| VM        | IP         | Servicio                    |
| --------- | ---------- | --------------------------- |
| monitoreo | 10.10.10.3 | Prometheus · Grafana · Loki |

#### Verificación del estado de las redes

```bash
sudo lxc network list
```

```
| ovn-app   | ovn | YES | 10.10.20.1/24 | 5 | CREATED |
| ovn-dmz   | ovn | YES | 10.10.30.1/24 | 1 | CREATED |
| ovn-infra | ovn | YES | 10.10.10.1/24 | 1 | CREATED |
```

***

### 4. Relación entre los tres componentes

Los tres componentes operan en capas independientes pero complementarias:

**LXD** gestiona el ciclo de vida de las VMs (creación, arranque, parada, migración) y utiliza MicroCeph como backend de almacenamiento para los discos de las VMs, y MicroOVN como backend de red para las interfaces de las VMs.

**MicroCeph** proporciona el pool de almacenamiento distribuido `remote` sobre el que LXD almacena los discos de las VMs. La replicación x3 garantiza que si un nodo físico falla, LXD puede arrancar la VM en otro nodo sin pérdida de datos.

**MicroOVN** proporciona las redes virtuales aisladas sobre las que corren las VMs. El UPLINK (vlan50) es la interfaz de salida que conecta las redes OVN con OPNsense y de ahí a internet.

```
Internet
    │
OPNsense (vlan50 UPLINK)
    │
Router lógico OVN
    ├── ovn-app   (switch lógico → data, app-core, vision, dmz-receiver eth1)
    ├── ovn-dmz   (switch lógico → dmz-receiver eth0)
    └── ovn-infra (switch lógico → monitoreo)
```

El tráfico entre VMs dentro de la misma red OVN nunca abandona el software de OVN. El tráfico entre redes distintas pasa por el router lógico. Solo el tráfico con destino a internet pasa por OPNsense.
