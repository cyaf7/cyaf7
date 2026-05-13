# Switch HP 1810-24G - Configuración y Gestión

### Introducción

El switch HP es el concentrador FÍSICO de todas las VLANs. Sin su configuración correcta, la segmentación de VLAN es imposible.

### Especificaciones Técnicas

| Característica         | Valor                                   |
| ---------------------- | --------------------------------------- |
| Modelo                 | HP 1810-24G (J9801A)                    |
| Puertos                | 24x Gigabit Ethernet (10/100/1000 Mbps) |
| Estándar               | IEEE 802.3 (Ethernet), 802.1Q (VLAN)    |
| VLANs Soportadas       | 1-4094                                  |
| Velocidad de Backplane | 48 Gbps                                 |
| Throughput Máximo      | 35.7 Mpps (mega packets per second)     |
| Interfaz de Gestión    | Web UI (HTTP/HTTPS), SSH, Telnet        |
| Dirección IP Gestión   | 192.168.2.10/24                         |
| Fuente de Alimentación | 100-240 VAC, 50/60 Hz                   |
| Consumo                | \~30W máximo                            |

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

***

### Troubleshooting

#### Caso 1: "Nodo no obtiene DHCP"

Se verifica:

1. Puerto P7/P11/P19 tiene VLAN 20 como T
2. DHCP está activo en OPNsense VLAN 20
3. Cable conectado correctamente

```bash
# En nodo:
sudo netplan apply
ip addr show vlan20
ip -4 addr show vlan20
```



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

***

