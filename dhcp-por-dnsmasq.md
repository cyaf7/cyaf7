---
description: en debian
---

# dhcp por dnsmasq

## Configuración de un servidor Debian como DHCP, router y NAT para una red interna

### 1. Escenario y objetivo

El sistema está compuesto por un servidor Debian 13 con dos interfaces de red y un cliente conectado únicamente a la red interna.

Red externa (salida a Internet):

* Red 172.20.10.0/24
* Interfaz en el servidor: enp0s3
* IP del servidor en esta red: 172.20.10.3
* Gateway real hacia Internet: 172.20.10.1

Red interna (LAN privada):

* Red 10.10.10.0/24
* Interfaz en el servidor: enp0s8
* IP del servidor en la LAN: 10.10.10.10

Objetivos:

1. Que el servidor actúe como servidor DHCP para la red interna.
2. Que el cliente obtenga IP automáticamente en la red 10.10.10.0/24.
3. Que el servidor actúe como router y permita al cliente salir a Internet.
4. Que la configuración sea persistente tras reiniciar.

***

### 2. Configuración de interfaces de red en Debian

La configuración se realizó con ifupdown mediante el archivo /etc/network/interfaces. Se utilizó allow-hotplug en lugar de auto porque en este entorno evitaba errores al levantar las interfaces.

Configuración aplicada:

```ini
allow-hotplug enp0s3
iface enp0s3 inet dhcp

allow-hotplug enp0s8
iface enp0s8 inet static
    address 10.10.10.10
    netmask 255.255.255.0
```

No se define gateway en la interfaz interna porque la ruta por defecto debe estar asociada a la interfaz externa. La interfaz interna solo define la red local.

Comprobación de estado de interfaces:

```bash
ip a
ip route
```

Salida esperada:

* enp0s3 con IP 172.20.10.3 y ruta default
* enp0s8 con IP 10.10.10.10/24

***

### 3. Configuración del servidor DHCP con dnsmasq

dnsmasq se utilizó como servidor DHCP exclusivo para la red interna.

Archivo de configuración editado: /etc/dnsmasq.conf

Configuración relevante:

```ini
interface=enp0s8
bind-interfaces

dhcp-range=10.10.10.50,10.10.10.100,255.255.255.0,12h

dhcp-option=3,10.10.10.10
```

Explicación de cada directiva:

interface=enp0s8\
Indica que dnsmasq solo escucha en la interfaz interna.

bind-interfaces\
Fuerza a dnsmasq a enlazarse únicamente a la interfaz indicada.

dhcp-range\
Define el rango de IPs dinámicas para los clientes.

dhcp-option=3\
La opción 3 en DHCP corresponde al router por defecto. Indica al cliente cuál es su gateway.

Se usa 10.10.10.10 porque el gateway del cliente debe estar en su misma red. El cliente no puede usar directamente una IP de la red 172.20.10.0/24 porque no pertenece a su segmento de red.

Reinicio y comprobación del servicio:

```bash
sudo systemctl restart dnsmasq
sudo systemctl status dnsmasq
```

Comprobación en el cliente:

```bash
ip a
```

Resultado esperado:

* IP 10.10.10.x
* Máscara /24

***

### 4. Activación del reenvío de paquetes (ip\_forward)

Activación temporal:

```bash
sudo sysctl -w net.ipv4.ip_forward=1
```

Comprobación:

```bash
cat /proc/sys/net/ipv4/ip_forward
```

Salida esperada:

```
1
```

Sin esta opción activa, los paquetes llegan al servidor pero no son reenviados a la interfaz externa.

***

### 5. Configuración del firewall y NAT con iptables

#### 5.1 Permitir el reenvío entre interfaces

La política por defecto de la cadena FORWARD era DROP, por lo que era necesario permitir explícitamente el tráfico.

Permitir tráfico desde la red interna hacia Internet:

```bash
sudo iptables -I FORWARD 1 -i enp0s8 -o enp0s3 -s 10.10.10.0/24 -j ACCEPT
```

Permitir tráfico de retorno (solo conexiones establecidas):

```bash
sudo iptables -I FORWARD 1 -i enp0s3 -o enp0s8 -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
```

Comprobación de política, reglas y contadores:

```bash
sudo iptables -L FORWARD -n -v
```

Se verificó que los contadores de paquetes aumentaban al hacer ping desde el cliente.

***

#### 5.2 Configuración de NAT

Regla aplicada:

```bash
sudo iptables -t nat -A POSTROUTING -s 10.10.10.0/24 -o enp0s3 -j MASQUERADE
```

Comprobación de reglas NAT:

```bash
sudo iptables -t nat -L -n -v
```

***

### 6. Comandos de diagnóstico utilizados durante la resolución de errores

**Pra subir o bajar una interfaz sin dhclient:**

```bash
sudo ip link set enp0s8 up
```

Activa la interfaz de red a nivel del sistema.



```bash
sudo ip link set enp0s8 down
```

Desactiva la interfaz de red.



```bash
ip a show enp0s8
```

Muestra el estado actual de la interfaz.

Nota: hacer down y luego up fuerza una nueva negociación de red cuando no existe dhclient.

\
**dnsmasq guarda leases en:**

```bash
/var/lib/misc/dnsmasq.leases
```

Ver rutas desde el punto de vista del kernel:

```bash
ip route
ip route get 8.8.8.8 from 10.10.10.23
```

Captura de tráfico ICMP para ver hasta dónde llegaban los paquetes:

en el server:

```bash
sudo tcpdump -i enp0s8 icmp
sudo tcpdump -i enp0s3 icmp
```

en la client hacer un ping -c 3 8.8.8.8, y luego mirar que sale de respuesta en el server.

Ver reglas y contadores de iptables:

```bash
sudo iptables -L FORWARD -n -v
sudo iptables -t nat -L -n -v
```

Instalar y usar conntrack para limpiar estados (si es necesario):

```bash
sudo apt update
sudo apt install -y conntrack
sudo conntrack -F
```

Comprobación de conectividad desde el cliente:

```bash
ping 10.10.10.10
ping -c 3 8.8.8.8
```

***

### 7. Persistencia tras reinicio

#### 7.1 Persistencia de sysctl en Debian 13

Archivo creado:

/etc/sysctl.d/99-router.conf

Contenido:

```ini
net.ipv4.ip_forward=1

net.ipv4.conf.all.rp_filter=0
net.ipv4.conf.default.rp_filter=0
net.ipv4.conf.enp0s3.rp_filter=0
net.ipv4.conf.enp0s8.rp_filter=0
```

Aplicación inmediata:

```bash
sudo sysctl --system
```

***

#### 7.2 Persistencia de iptables

Instalación:

```bash
sudo apt install -y iptables-persistent
```

Durante la instalación se guardan las reglas actuales para que se restauren automáticamente en cada arranque.

Comprobación del servicio de persistencia:

```bash
systemctl status netfilter-persistent
```

***

### 8. Flujo final de tráfico

1. El cliente obtiene IP por DHCP en la red 10.10.10.0/24.
2. El cliente envía tráfico externo a su gateway 10.10.10.10.
3. El servidor reenvía el tráfico desde enp0s8 a enp0s3.
4. El servidor aplica NAT y envía el paquete a Internet.
5. La respuesta vuelve al servidor, se deshace el NAT y se entrega al cliente.
