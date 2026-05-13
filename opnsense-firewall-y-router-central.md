# OPNsense - Firewall y Router Central

### Introducción

OPNsense es el corazón de la red de Spotly. Todos los paquetes entre VLANs pasan por él. Sin OPNsense correctamente configurado, la segmentación VLAN es inútil.

### Acceso a la Web UI

#### Primera Conexión

1. Abrir navegador: https://192.168.109.x
2. Usuario: root
3. Contraseña: opnsense

**Advertencia**: IP dinámica (DHCP desde escuela)

* Anótala o accede vía SSH para encontrarla

#### Cambiar a IP Estática

System > Settings > General

```
Hostname: spotly-opnsense
Domain: spotly.local

Interface em0 (WAN):
  IPv4 Configuration Type: DHCP (déjalo así)
  
Interface VLAN:
  (Configuraremos después)
```

***

### Interfaz WAN (em0)

#### Descripción

* **Interfaz Física**: NIC integrada
* **Conexión**: Cable directo a router escuela
* **Configuración**: DHCP automático
* **IP**: 192.168.109.x (dinámico)
* **Gateway**: 192.168.109.1 (router escuela)
* **DNS**: 8.8.8.8, 8.8.4.4

#### Verificación

Interfaces > WAN (em0)

```
Status: ONLINE
IPv4 Address: 192.168.109.123 (dinámico)
Gateway: 192.168.109.1
MTU: 1500
```

#### Propósito

* Salida a internet (fuera de la escuela)
* Acceso a actualizaciones de OPNsense
* Backup cloud (futuro)
* **NO** para comunicación intra-cluster

***

### Interfaz LAN (ue0) - USB Realtek

#### Conexión Física

1. Conectar adaptador USB Realtek a OPNsense
2. Conectar cable Ethernet al switch HP (cualquier puerto)
3. Sistema detecta automáticamente como ue0

***

### Subinterfaces VLAN&#x20;

#### ¿Qué es una Subinterfaz VLAN?

Una subinterfaz VLAN es una interfaz virtual que etiqueta o desempaqueta tramas Ethernet con una ID VLAN específica.

```
Trama Ethernet sin etiqueta:
  [Destino | Origen | EtherType | Payload | CRC]

Trama Ethernet con VLAN (802.1Q):
  [Destino | Origen | VLAN Tag | EtherType | Payload | CRC]
```

Cuando una trama llega a ue0:

* Si tiene VLAN 10 → ve subinterfaz vlan0.10
* Si tiene VLAN 20 → ve subinterfaz vlan0.20
* Etc.

#### Crear Subinterfaces

**VLAN 10 (Intra-cluster)**

Interfaces > VLAN > + Add

```
Parent Interface: ue0
VLAN Tag: 10
VLAN Priority: 0
Description: Cloud cluster
```

Luego: Interfaces > LAN VLAN10

```
IPv4 Configuration Type: Static
IPv4 Address: 192.168.10.1
IPv4 Subnet Mask: 255.255.255.0

DHCP Server: ENABLED
DHCP Range: 192.168.10.50 - 192.168.10.100
```

**VLAN 20 (Management)**

Interfaces > VLAN > + Add

```
Parent Interface: ue0
VLAN Tag: 20
Description: Management SSH
```

Interfaces > LAN VLAN20

```
IPv4 Configuration Type: Static
IPv4 Address: 192.168.20.1
IPv4 Subnet Mask: 255.255.255.0

DHCP Server: ENABLED
DHCP Range: 192.168.20.50 - 192.168.20.100
Routes: Default gateway 192.168.20.1
DNS: 8.8.8.8
```

**VLAN 40 (Backup)**

```
Parent Interface: ue0
VLAN Tag: 40
Description: Backup window
```

```
IPv4 Address: 192.168.40.1
Subnet: 255.255.255.0

DHCP: DISABLED (IP estática en backup-pc)
```

***

### NAT Outbound Configuration

#### Propósito

Permitir que tráfico desde VLANs internas salga a internet (WAN) con IP pública escuela.

#### Configuración Automática

Firewall > NAT > Outbound (Auto)

OPNsense genera automáticamente:

```
Interface: em0 (WAN)
Source: any
Destination: any
Translation: Interface Address
```

Esto significa: cualquier tráfico desde cualquier interfaz interna que salga a WAN se convierte a la IP de em0.

#### Regla Manual para VLAN 10

Para mayor control sobre el cluster:

Firewall > NAT > Outbound (Manual)

```
Interface: em0 (WAN)
Protocol: any
Source Address: 192.168.10.0/24
Destination: any
Translation Address: Interface (em0)
Translation Port: (none)
Description: "VLAN10 cluster outbound"
```

***

### Firewall Rules (Política DROP by Default)

#### Filosofía

**DEFAULT: DENY**

Todos los paquetes son bloqueados a menos que una regla explícitamente los permita.

#### Crear Firewall Rule Set para VLAN 10 (Cloud)

Firewall > Rules > VLAN10

**Regla 1: Allow Intra-VLAN10**

```
Action: Pass
Interface: VLAN10
Protocol: any
Source: VLAN10 network
Destination: VLAN10 network
```

**Explicación**: Todos los nodos en VLAN 10 pueden comunicarse entre sí sin restricción.

**Regla 2: Allow VLAN10 to WAN**

```
Action: Pass
Interface: VLAN10
Protocol: any
Source: VLAN10 network
Destination: any
```

**Explicación**: VLAN 10 puede salir a internet (tráfico outbound).

**Regla 3: Block Everything Else (Implicit)**

```
(No rule = DROP)
```

**Resultado**:

* VLAN10 ↔ VLAN20: BLOQUEADO
* VLAN10 → outbound: PERMITIDO

***

#### Firewall Rule Set para VLAN 20 (Management)

Firewall > Rules > VLAN20

**Regla 1: Allow SSH from VLAN20 to VLAN10:22**

```
Action: Pass
Interface: VLAN20
Protocol: TCP
Source: VLAN20 network
Destination: VLAN10 network / Port 22
```

**Explicación**: Admin PC (VLAN 20) puede SSH a nodos (VLAN 10).

**Regla 2: Allow VLAN20 Outbound**

```
Action: Pass
Interface: VLAN20
Protocol: any
Source: VLAN20 network
Destination: any
```

**Resultado**:

* VLAN20 → VLAN10:22: PERMITIDO (SSH)
* VLAN20 → VLAN10 (otros): BLOQUEADO
* VLAN20 → outbound: PERMITIDO



#### Firewall Rule Set para VLAN 40 (Backup)

**Regla 1: Allow VLAN40 to VLAN10:22 (CON SCHEDULE)**

```
Action: Pass
Interface: VLAN40
Protocol: TCP
Source: VLAN40 network
Destination: VLAN10 network / Port 22
Schedule: backup-window
```

**Regla 2: Allow VLAN40 to VLAN20:22 (CON SCHEDULE)**

```
Action: Pass
Interface: VLAN40
Protocol: TCP
Source: VLAN40 network
Destination: VLAN20 network / Port 22
Schedule: backup-window
```

**Explicación**: VLAN 40 SOLO tiene acceso SSH durante 02:30-04:00 (ventana de backup).

**Resultado**:

* 02:30-04:00: backup-pc puede rsync desde nodos y admin-pc
* Resto del día: BLOQUEADO completamente

***

### Firewall Schedules



***

### Verificación

#### Tabla de Enrutamiento

System > Diagnostics > Routes

```
Destination        Gateway         Interface
──────────────────────────────────────────────
0.0.0.0/0          192.168.109.1   em0 (WAN)
192.168.1.0/24     192.168.1.1     ue0
192.168.10.0/24    192.168.10.1    vlan0.10
192.168.20.0/24    192.168.20.1    vlan0.20
192.168.40.0/24    192.168.40.1    vlan0.40
```

### Backup y Restore de Configuración

#### Backup

System > Backup & Restore

```
Backup Filename: spotly-opnsense-config
Action: Backup
→ Descarga archivo: spotly-opnsense-config.xml
```

Guardar en Admin PC y disco externo.

#### Restore

Si OPNsense falla:

```
System > Backup & Restore
Action: Restore
Upload File: spotly-opnsense-config.xml
→ OPNsense reinicia con config anterior
```

### Troubleshooting

#### Caso 1: "VLAN10 no puede hacer ping a internet"

Verificar:

1. VLAN10 subinterfaz existe: `netstat -rn | grep 192.168.10`
2. NAT rule existe: Firewall > NAT > Outbound
3. Firewall rule permite outbound: Firewall > Rules > VLAN10

#### Caso 2: "VLAN10 puede acceder VLAN20 (security issue)"

Verificar:

1. Regla DROP existe al final de VLAN10: Firewall > Rules > VLAN10
2. No hay regla explicit Pass a VLAN20: Firewall > Rules > VLAN10

#### Caso 3: "Algunos paquetes se pierden en el cluster"

Verificar:

1. MTU en todas las subinterfaces = 1500
2. Cables Ethernet conectados correctamente
3. Switch HP tiene VLANs etiquetadas correctamente
