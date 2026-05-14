# Acceso Remoto — Tailscale (Temporal) y Futuro WireGuard

### Tailscale — Situación Actual

Tailscale es una VPN mesh basada en WireGuard que simplifica la conectividad P2P entre dispositivos. Actualmente está instalada en los tres nodos del cluster como solución temporal durante el desarrollo.

### Cómo Funciona Tailscale

Tailscale crea una red privada virtual donde cada dispositivo (nodo) recibe una IP en el rango 100.64.0.0/10. Los dispositivos se descubren automáticamente mediante servidores centrales de Tailscale en internet. Una vez descubiertos, intentan establecer conexión P2P directa; si fallan (por firewall restrictivo), el tráfico se enruta a través de servidores relay de Tailscale.

**Ventajas:**

* Configuración sencilla: instalar, iniciar sesión con cuenta, listo
* Descubrimiento automático
* Funciona a través de NAT/firewalls restrictivos
* No requiere mantener certificados ni infraestructura

**Desventajas:**

* El tráfico puede pasar por servidores Tailscale en USA (proveedor externo)
* No es privado: Tailscale ve metadatos del tráfico
* Bypassea el firewall OPNsense: las reglas de OPNsense no aplican a tráfico Tailscale
* No está bajo control de la institución educativa

### Instalación y Configuración

Tailscale se instala desde snap:

```
sudo snap install tailscale
```

Iniciar el daemon:

```
sudo tailscale up
```

El comando devuelve una URL para autenticarse con una cuenta Tailscale. Cada nodo se registra bajo la misma "red Tailscale" (tailnet).

Una vez autenticados, los nodos son alcanzables por Tailscale IP desde cualquier lugar. Por ejemplo, conectar a nodo01 desde laptop externa:

```
ssh ubuntu@100.64.XX.XX
```

Donde 100.64.XX.XX es la IP Tailscale de nodo01.

### Seguridad en Nodos — Restricción de SSH

Aunque Tailscale permite acceso desde cualquier lugar, las reglas de firewall local (nftables) en cada nodo lo restringen. Cada nodo tiene una regla:

```
tcp dport ssh allow from 192.168.20.0/24, 100.64.0.0/10
```

SSH solo se permite desde VLAN 20 (Admin PC) o desde el rango Tailscale. Cualquier otra fuente es descartada.

### Limitaciones Conocidas

**1. Tráfico Bypassea OPNsense:**

El firewall OPNsense no ve conexiones Tailscale. Logs de OPNsense muestran 0 tráfico SSH a través de Tailscale, incluso cuando alguien está conectado remotamente. Esto complica auditoría y control.

**2. No Está Bajo Control:**

Tailscale es servicio externo. Si los servidores de Tailscale se apagan, la red se desconecta. No hay control institucional.

**3. Escalabilidad Limitada:**

Plan Free permite solo una red Tailscale por cuenta. Para múltiples redes, se requiere plan pago.

### Monitoreo de Conexiones Tailscale

Verificar dispositivos conectados y estado:

```
sudo tailscale status
```

Muestra lista de nodos en la red Tailscale, sus IPs, y estado de conectividad (directa vs relay).

### Transitorio — Plan de Migración a WireGuard

WireGuard es un protocolo VPN moderno, minimalista y rápido. A diferencia de Tailscale (que es un servicio gestionado), WireGuard es un protocolo que se implementa localmente.

#### Implementación Prevista

**Hardware:** OPNsense firewall (máquina del laboratorio, bajo control)

**Configuración:**

1. Instalar plugin os-wireguard en OPNsense
2. Crear túnel WireGuard con direcciones internas (ej. 10.8.0.0/24)
3. Cada miembro del equipo genera par de claves ED25519
4. Admin agrega claves públicas como peers en OPNsense
5. Cada miembro configura cliente WireGuard en su laptop/desktop
6. Conexión a IP pública de OPNsense (em0, si está abierta) → entra a VLAN 20 privada

#### Ventajas de WireGuard

* Bajo control institucional: corre en OPNsense, máquina del laboratorio
* Tráfico pasa por firewall OPNsense: logs completos, auditoría
* Sin servidores externos: 100% privado
* Rendimiento superior: protocolo minimalista
* Configuración persistente en OPNsense: cambios sobreviven rebooteos

#### Proceso de Migración

1. **Implementar WireGuard en OPNsense** (requiere presencia física en laboratorio)
2. **Desplegar configuración a nodos** mediante playbook Ansible
3. **Agregar ruta** para tráfico WireGuard en Netplan de nodos
4. **Testing** con equipo de desarrollo
5. **Desactivar Tailscale** en nodos (mantener instalado pero deshabilitado)
6. **Documentar** configuración WireGuard en nuevo documento

#### Cambios en Infraestructura

Una vez WireGuard activo:

* Prometheus targets en nodos físicos (actualmente DOWN) serían alcanzables
* Acceso administrativo sería visible en OPNsense logs
* Tailscale se mantiene como fallback (no se desinstala inmediatamente)

### Consideraciones de Seguridad

**Actual (Tailscale):**

* Punto de entrada: Tailscale (servicio externo)
* Tráfico visible a: Tailscale Inc (USA)
* Firewall control: nftables en nodos
* Auditoría: logs locales en nodos

**Futuro (WireGuard):**

* Punto de entrada: OPNsense WAN (IP pública de firewall si está abierta, o solo intra-institución)
* Tráfico visible a: solo Spotly team (encriptado en tránsito)
* Firewall control: OPNsense + nftables en nodos
* Auditoría: logs OPNsense + logs locales en nodos

WireGuard ofrece control y privacidad superiores.

### Próximos Pasos Inmediatos

1. Mantener Tailscale operativo durante desarrollo
2. Documentar configuración WireGuard esperada (será próximo paso)
3. Evaluar necesidad de IP pública en em0 de OPNsense (para acceso WireGuard externo)
4. Reservar tiempo para implementación presencial (en laboratorio)

### Referencia Rápida

**Status Tailscale:**

```
sudo systemctl status tailscale
sudo tailscale status
```

**Desactivar Tailscale (sin desinstalar):**

```
sudo tailscale down
```

**Reactivar:**

```
sudo tailscale up
```

**Remover completamente (destructivo):**

```
sudo snap remove tailscale
```
