# Automatización con Ansible

### ¿Qué es Ansible?

Ansible es una herramienta de automatización de infraestructura que permite gestionar la configuración de múltiples servidores de forma simultánea y declarativa mediante archivos YAML llamados _playbooks_. A diferencia de otras herramientas como Puppet o Chef, Ansible no requiere la instalación de un agente en los servidores gestionados: se conecta por SSH a cada máquina y ejecuta los comandos necesarios de forma remota. Esto la convierte en una herramienta sumamente flexible para entornos pequeños, medianos y grandes.

La estructura fundamental de Ansible es la siguiente:

* **Inventario (inventory):** Archivo que define qué máquinas van a ser gestionadas, agrupadas por roles o funciones. En Spotly se encuentra en `/root/ansible/inventory/hosts.yml`.
* **Playbook:** Archivo YAML que describe las tareas que deben ejecutarse en un grupo de máquinas. Cada tarea es idempotente: ejecutarla una vez produce el mismo resultado que ejecutarla múltiples veces.
* **Módulos:** Componentes que ejecutan acciones específicas (instalar paquetes con `apt`, crear directorios, copiar archivos, etc.).
* **Variables:** Valores que cambian según el contexto, permitiendo reutilizar el mismo playbook en diferentes escenarios.

***

### Por qué Ansible fue primordial en Spotly

La infraestructura de Spotly consta de tres nodos físicos interconectados que deben mantener exactamente la misma configuración de red, los mismos usuarios, las mismas herramientas de seguridad y la misma versión de software. Sin automatización, cualquier cambio requeriría conectarse manualmente a cada nodo, ejecutar los mismos comandos tres veces, y documentar qué se cambió en cada uno. Esto introducía inconsistencia, consumía tiempo y era propenso a errores.

Ansible resolvió este problema por las siguientes razones:

#### 1. Consistencia garantizada

Cuando se ejecuta un playbook en los tres nodos, el resultado es idéntico. No existen diferencias accidentales entre máquinas debido a comandos olvidados o errores de digitación. El playbook es la fuente única de verdad sobre cómo debe estar configurada la infraestructura.

#### 2. Auditoría y trazabilidad

Cada playbook se guarda en control de versiones (Git). Si algo falla después de aplicar un playbook, se sabe exactamente qué cambió, cuándo se cambió y por qué. Los logs de ejecución permiten ver el resultado de cada tarea.

#### 3. Recuperación ante fallos

Si uno de los nodos falla durante la instalación de MicroCloud o la configuración de la red, no es necesario volver a comenzar desde cero. Se ejecuta el playbook nuevamente en el nodo afectado y se restaura al estado deseado.

#### 4. Repetibilidad del entorno

En Spotly, el cluster MicroCloud se construyó desde cero en el laboratorio. Si en el futuro fuera necesario trasladar la infraestructura a nuevos nodos, los playbooks permiten reproducir la configuración completa automáticamente sin necesidad de recordar pasos manuales.

#### 5. Integración con herramientas de orquestación

Semaphore, la interfaz web para Ansible, permite ejecutar playbooks sin acceso a la línea de comandos. Esto es especialmente útil para operaciones de mantenimiento programadas o para que otros miembros del equipo ejecuten tareas sin necesidad de ser expertos en Ansible o en línea de comandos.

#### 6. Seguridad mediante código

Las prácticas de seguridad (reglas de nftables, restricciones SSH, permisos de sudo) quedan documentadas y versionadas en los playbooks. Cualquier cambio en la política de seguridad se refleja en el código, revisado antes de aplicarse.

***

### Estructura del proyecto Ansible en Spotly

```
/root/ansible/
├── inventory/
│   └── hosts.yml                 # Definición de nodos: nodo01, nodo02, nodo03
├── playbooks/
│   ├── bootstrap.yml             # Inicialización: actualizaciones, SSH, paquetes base
│   ├── setup_team_users.yml      # Creación de usuarios del equipo localmente
│   ├── sssd_nodos.yml            # Configuración de LDAP/SSSD
│   ├── setup_ceph_volumes.yml    # Creación de volúmenes LVM para MicroCeph
│   ├── sync_and_list_snaps.yml   # Sincronización de versiones de snaps
│   ├── check_connectivity_vms.yml # Validación de conectividad en VMs
│   ├── nftables.yml              # Firewall host-level
│   ├── fail2ban.yml              # Protección contra ataques de fuerza bruta
│   ├── purge_local_users.yml     # Eliminación de usuarios locales del sistema
│   ├── backup_total.yml          # Configuración de backup con rsync
│   ├── deploy_wazuh_agents.yml   # Instalación de agentes en nodos
│   └── deploy_wazuh_agents_vms.yml # Instalación de agentes en VMs
└── group_vars/
    └── bare_metal.yml            # Variables que aplican a todos los nodos físicos
```

***

### Ejecución de playbooks: workflow típico

El flujo típico de trabajo con Ansible en Spotly es el siguiente:

1. **Escribir o modificar el playbook** en `/root/ansible/playbooks/`
2.  **Hacer un dry-run** para ver qué cambios se aplicarían sin modificar nada:

    ```bash
    ansible-playbook -i /root/ansible/inventory/hosts.yml \
      /root/ansible/playbooks/nombre_playbook.yml --check
    ```
3. **Revisar la salida** del dry-run para asegurar que los cambios esperados se van a ejecutar.
4.  **Aplicar el playbook** en uno o más nodos:

    ```bash
    ansible-playbook -i /root/ansible/inventory/hosts.yml \
      /root/ansible/playbooks/nombre_playbook.yml
    ```
5. **Verificar el resultado** conectándose a los nodos o inspeccionando los logs de ejecución.

***

### Principios aplicados en los playbooks de Spotly

Todos los playbooks del proyecto siguen estos principios:

* **Idempotencia:** Ejecutar el playbook múltiples veces produce el mismo resultado. No se crean duplicados ni se generan errores si la tarea ya está aplicada.
* **Explicititud:** Cada tarea incluye un mensaje descriptivo (`name:`) que explica qué se está haciendo.
* **Manejo de errores:** Se incluyen condiciones (`when:`) y validaciones para no fallar ante situaciones esperadas.
* **Documentación inline:** Los comentarios en YAML explican decisiones arquitectónicas o configuraciones no obvias.

***

## Inventarios de Ansible

Los inventarios son archivos YAML que definen cuáles máquinas serán gestionadas por Ansible y cómo se accede a ellas. En Spotly existen dos inventarios complementarios ubicados en `/root/ansible/inventory/`.

### hosts.yml (Nodos Bare Metal)

Este archivo define los tres nodos físicos (nodo01, nodo02, nodo03) agrupados bajo `bare_metal`. Cada nodo recibe su dirección IP en VLAN 20 (192.168.20.11, .12, .13), que es la red de management para SSH. Las variables globales `ansible_user: ansible` y `ansible_ssh_private_key_file: /root/.ssh/id_ed25519_ansible` garantizan que Ansible se conecta al usuario `ansible`con clave criptográfica sin contraseña en todos los nodos. Este inventario se usa para playbooks que ejecutan en infraestructura física: bootstrap, sssd, nftables, fail2ban, etc.

### hosts\_vms.yml (Máquinas Virtuales)

Este inventario define las VMs del cluster agrupadas por nodo físico donde corren: `vms_nodo01` (data, backup-vm) y `vms_nodo03` (app-core, dmz-receiver, monitoreo, vision). Ansible no accede directamente a las VMs: se conecta por SSH a cada nodo físico (192.168.20.11 o .13) y desde ahí ejecuta `lxc exec vm-name -- comando` como root. Cada VM recibe la variable `lxc_vm_name` que almacena su nombre LXD, usado en los playbooks para construir comandos dinámicos. El grupo unificado `vms:` permite ejecutar playbooks contra todas las VMs simultáneamente. Se usa en deploy\_wazuh\_agents\_vms.yml y check\_connectivity\_vms.yml.

### Ejecución con inventarios específicos

```bash
# Ejecutar en nodos físicos
ansible-playbook -i /root/ansible/inventory/hosts.yml \
  /root/ansible/playbooks/nftables.yml

# Ejecutar en VMs
ansible-playbook -i /root/ansible/inventory/hosts_vms.yml \
  /root/ansible/playbooks/deploy_wazuh_agents_vms.yml
```

***

## Catálogo de Playbooks

### 1. bootstrap.yml

**Propósito:** Preparación inicial de todos los nodos para integración con Ansible y MicroCloud.

Este playbook es la base de toda automatización: crea el usuario `ansible` en todos los nodos como cuenta de servicio dedicada para ejecutar las tareas remotas. Realiza: creación del grupo y usuario ansible, generación del directorio .ssh con permisos 0700, copia de la clave pública SSH del administrador (id\_ed25519\_ansible.pub) a authorized\_keys, configuración de sudoers para permitir que ansible ejecute comandos con sudo sin requerir contraseña (NOPASSWD), y bloqueo de login por contraseña (password\_lock) para forzar autenticación por clave SSH. Finaliza con una verificación: ejecuta `sudo -n whoami` sin contraseña para confirmar que el acceso privilegiado funciona correctamente. Sin este playbook, Ansible no podría conectarse ni ejecutar tareas.

***

### 2. setup\_team\_users.yml

**Propósito:** Creación de usuarios locales de la equipo en todos los nodos y VMs.

Este playbook crea los usuarios locales de la equipo (mortari, aoki, figueroa) en todos los nodos y VMs del cluster. Para cada usuario: crea la cuenta con shell /bin/bash, la añade al grupo sudo para permitir comandos administrativos, crea el directorio .ssh con permisos restrictivos (0700), copia la clave pública SSH del administrador Ansible (id\_ed25519\_ansible.pub) al archivo authorized\_keys de cada usuario, y finalmente bloquea el login por contraseña mediante password\_lock para forzar autenticación exclusivamente mediante pares de claves criptográficas ED25519. Al finalizar, los usuarios pueden acceder por SSH pero no pueden usar contraseña, garantizando seguridad.

***

### 3. sssd\_nodos.yml

**Propósito:** Configuración de SSSD para autenticación centralizada contra LDAP.

Este playbook configura SSSD (System Security Services Daemon) en los tres nodos físicos para autenticar usuarios contra el servidor LDAP centralizado en el Admin PC (192.168.20.20). Instala los paquetes necesarios (sssd, libnss-sss, libpam-sss, ldap-utils), copia el certificado CA autofirmado del servidor LDAP, y escribe la configuración completa de SSSD incluyendo búsqueda de usuarios, grupos y claves SSH públicas. Configura NSS y PAM para resolver usuarios desde LDAP en lugar de archivos locales, habilita `AuthorizedKeysCommand` en SSH para obtener claves públicas directamente desde LDAP, y finaliza con verificaciones: valida que puede obtener la clave SSH de camilly desde LDAP y que el grupo spotly-admin se resuelve correctamente.

***

### 4. setup\_ceph\_volumes.yml

**Propósito:** Creación de volúmenes lógicos LVM para almacenamiento de MicroCeph.

Este playbook crea un volumen lógico (LV) de 100 GB denominado microcloud-storage dentro del Volume Group ubuntu-vg existente en todos los nodos. Usa el módulo community.general.lvol para crear el LV de forma idempotente. Sin embargo, este enfoque fue descartado porque MicroCloud no detecta volúmenes lógicos que pertenecen al VG del sistema operativo por razones de seguridad (rechaza discos que comparten el mismo grupo que la raíz del SO). **Nota:**&#x45;l proyecto evolucionó hacia loop files (/mnt/ceph-disks/ceph-osd.img) como solución definitiva, que es el método oficial recomendado por Canonical para entornos de desarrollo y laboratorio.

***

### 5. sync\_and\_list\_snaps.yml

**Propósito:** Sincronización y verificación de versiones de snaps en todos los nodos.

Este playbook sincroniza las versiones de los cuatro snaps críticos de MicroCloud en todos los nodos usando el parámetro `--cohort="+"`, que es fundamental para evitar incompatibilidades. El cohort vincula todos los nodos al mismo canal de actualización: cuando un nodo se actualiza, todos los demás reciben exactamente la misma versión. Esto previene el error "Rejecting peer due to invalid MicroCeph version" durante microcloud init, que ocurre si los nodos tienen versiones diferentes. Tras aplicar, lista las versiones actuales de lxd, microceph, microovn y microcloud en cada nodo para auditoría. Es el playbook que se ejecuta antes de cualquier operación de cluster para garantizar consistencia.

***

### 6. nftables.yml

**Propósito:** Implementación de firewall host-level mediante nftables.

Este playbook configura nftables (firewall host-level) en los tres nodos bare\_metal con política de DROP implícita en tráfico entrante. Define la tabla inet filter con tres chains: input (DROP por defecto, permite loopback, conexiones establecidas, ICMP, SSH desde VLAN 20 management + red Tailscale 100.64.0.0/10, y todo tráfico desde VLAN 10 intra-cluster), forward (DROP), output (ACCEPT sin restricción). Crítico: incluye validación `wait_for` en puerto 22 después de aplicar para garantizar que SSH sigue accesible y no se bloquea el playbook. Finaliza listando todas las reglas activas con `nft list ruleset` para auditoría. El resultado: los nodos rechazan todo tráfico de entrada excepto SSH desde redes autorizadas y comunicación intra-cluster.

***

### 7. fail2ban.yml

**Propósito:** Instalación y configuración de fail2ban para protección contra ataques de fuerza bruta.

Este playbook instala y configura fail2ban en todos los nodos (bare\_metal y VMs) para proteger contra ataques de fuerza bruta en SSH. Crea la configuración jail.local definiendo política global: bantime 3600 segundos (1 hora), findtime 600 segundos (ventana de 10 minutos), maxretry 3 intentos fallidos antes de banear, y usa backend systemd con nftables-multiport para integración con el firewall del host. La sección \[sshd] especifica bantime más agresivo de 7200 segundos (2 horas) para SSH específicamente. Tras aplicar, cualquier IP que intente 3 logins SSH fallidos en 10 minutos es automáticamente bloqueada por nftables durante 2 horas. Finaliza verificando el estado con `fail2ban-client status sshd`.

***

### 8. check\_connectivity\_vms.yml

**Propósito:** Validación de acceso a VMs y conectividad con Wazuh Manager.

Este playbook valida que las VMs del cluster son accesibles vía `lxc exec` y que tienen conectividad de red hacia el Wazuh Manager (192.168.20.20:1514) antes de desplegar agentes de monitorización. Ejecuta tres verificaciones en cada VM: comprueba que lxc exec funciona ejecutando `whoami`, obtiene la versión del SO mediante `lsb_release -d`, y realiza un test de conectividad TCP a través de /dev/tcp hacia el Manager en puerto 1514 con timeout de 3 segundos. Finaliza con un resumen legible para cada VM mostrando usuario (root), SO y estado de acceso al Manager (ACCESIBLE o BLOQUEADO). Es una verificación de diagnóstico antes de ejecutar el playbook de despliegue de agentes.

***

### 9. deploy\_wazuh\_agents\_vms.yml

**Propósito:** Instalación de Wazuh Agent en todas las VMs del cluster.

Este playbook instala el Wazuh Agent (versión 4.7.2-1) en todas las VMs del cluster para centralizar monitorización de logs y seguridad. Ejecuta dentro de cada VM via `lxc exec`: limpia locks de dpkg de instalaciones anteriores, descarga e instala la clave GPG del repositorio Wazuh, configura el repositorio apt oficial, instala el paquete wazuh-agent con variables de entorno (WAZUH\_MANAGER=192.168.20.20, WAZUH\_AGENT\_NAME=nombre\_vm), y finalmente habilita e inicia el servicio. Tras aplicar, cada VM conecta automáticamente al Wazuh Manager en puerto 1514 para enviar telemetría de sistema y eventos de seguridad, permitiendo auditoría y detección de anomalías centralizada.

***

### 10. deploy\_wazuh\_agents.yml

**Propósito:** Instalación de Wazuh Agent en los nodos físicos bare\_metal.

Este playbook instala Wazuh Agent (versión 4.9.2-1) en los nodos físicos bare\_metal del cluster para centralizar monitorización de seguridad y logs a nivel de infraestructura. Descarga e instala la chave GPG del repositorio Wazuh, añade el repositorio apt oficial, instala el paquete wazuh-agent pasando como variables de entorno la IP del Manager (192.168.20.20) y el nombre del nodo, garantiza que la configuración de ossec.conf apunta al Manager correcto editando la etiqueta `<address>`, habilita e inicia el servicio systemd. Finaliza verificando el estado con `wazuh-control status`para confirmar conexión exitosa. Diferencia clave con el playbook anterior: este actúa directamente en nodos físicos, mientras que deploy\_wazuh\_agents\_vms.yml usa `lxc exec` para desplegar dentro de VMs.

***

### 11. purge\_local\_users.yml

**Propósito:** Eliminación de usuarios locales en preparación para migración a LDAP.

Este playbook purga completamente todos los usuarios locales (mortari, aoki, figueroa, prof, camilly, ivan) de todos los nodos en preparación para migrar a autenticación centralizada LDAP. Realiza: termina todos los procesos activos de cada usuario con `pkill -u`, elimina las cuentas de usuario y sus directorios home completos con `userdel -r` (remove: yes, force: yes), borra los archivos de sudoers individuales en `/etc/sudoers.d/`, elimina el archivo de grupo team\_users. Finaliza verificando que `/home` está limpia listando contenido. Después de aplicar este playbook, los usuarios solo existen en LDAP y acceden via SSSD, no como cuentas locales del sistema.

***

### 12. backup\_total.yml

**Propósito:** Automatización de backup total del cluster con exportación de VMs y configs de nodos.

Este playbook ejecuta un respaldo total en dos bloques secuenciales. **Bloque 1** (en los nodos bare\_metal): compacta configuraciones críticas (/etc/netplan, /etc/hosts, /var/lib/ovn, bases de datos LXD) en archivos tar.gz, y exporta todas las VMs del cluster (app-core, data, dmz-receiver, monitoreo, vision) mediante `lxc export --optimized-storage` con fecha en el nombre. **Bloque 2** (en localhost): coordina centralmente la transferencia de todos los archivos desde los nodos al servidor de backup físico en 192.168.40.10 via scp con clave SSH sin contraseña (BatchMode), implementa retención de 2 días eliminando backups antiguos en el servidor destino, y limpia archivos temporales en los nodos tras envío exitoso. Permite recuperación total del cluster si algún nodo falla.

***

_Fuente: Ansible Official Documentation — https://docs.ansible.com/_
