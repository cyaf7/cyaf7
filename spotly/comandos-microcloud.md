# Comandos MicroCloud

Este documento cubre los comandos principales de MicroCloud (lxc, microceph, microcloud, microovn) explicando para qué sirve cada uno.

### Comandos microcloud — Inicialización y Gestión del Cluster

**microcloud init** — Inicializa un nuevo cluster MicroCloud desde cero en el nodo donde se ejecuta. Este comando es interactivo: pregunta qué interfaz de red usar para comunicación intra-cluster, qué interfaz será el uplink de OVN, cuántos nodos participarán, rango de IPs para máquinas virtuales. Solo se ejecuta una vez en el primer nodo del cluster. Los otros nodos ejecutan `microcloud join` simultáneamente. Al completarse, el cluster tiene LXD, MicroCeph y MicroOVN instalados pero sin almacenamiento distribuido disponible (si los OSDs aún no existen).

**microcloud join** — Se ejecuta en nodos secundarios mientras `microcloud init` está en progreso en el nodo líder. Este comando hace que el nodo se descubra automáticamente mediante mDNS y se una al cluster. El nodo descarga la configuración de red del cluster líder y se sincroniza. Si se ejecuta después de que `microcloud init` terminó, falla porque el cluster ya está sellado.

**microcloud cluster list** — Desde cualquier nodo, muestra todos los nodos del cluster con sus direcciones IP, estado de participación en LXD, MicroCeph y MicroOVN. Útil para verificar que el cluster tiene el número esperado de nodos y que todos están activos.

**microcloud remove** — Elimina un nodo del cluster. Este comando es destructivo: purga el nodo de todas las estructuras de cluster. Se usa en mantenimiento o cuando se retira hardware permanentemente.

### Comandos lxc — Gestión de Máquinas Virtuales

#### Máquinas Virtuales Básicas

**lxc launch ubuntu:22.04 nombre --vm** — Crea e inicia una nueva máquina virtual. El flag `--vm` indica que será una máquina virtual completa (QEMU/KVM), no un contenedor. Ubuntu descarga la imagen base de internet (proceso lento, \~1 hora). La máquina arranca automáticamente una vez creada.

**lxc launch ubuntu:22.04 nombre --vm --storage remote** — Crea una máquina virtual con almacenamiento distribuido en MicroCeph. El disco de la máquina virtual se replicará automáticamente en 3 nodos. Sin este flag, la máquina usa almacenamiento local en el nodo donde se lanza, sin redundancia.

**lxc launch ubuntu:22.04 nombre --vm --storage remote --network ovn-app** — Crea una máquina virtual en una red OVN específica (en este caso ovn-app). Sin este flag, la máquina se conecta a la red por defecto que LXD creó durante microcloud init.

**lxc list** — Muestra todas las máquinas virtuales y contenedores del cluster con su estado (RUNNING, STOPPED), dirección IP, imagen base y nodo físico donde residen. Útil para inventario rápido.

**lxc info nombre** — Muestra información detallada de una máquina virtual específica: CPU, RAM, almacenamiento, interfaces de red, snapshots, estado de replicación en MicroCeph.

**lxc start nombre** — Inicia una máquina virtual detenida. El proceso de arranque incluye sincronización con MicroCeph si la máquina tiene discos allá.

**lxc stop nombre** — Detiene una máquina virtual en marcha. Es equivalente a un shutdown ordenado, no un apagado forzado. La máquina guarda su estado en MicroCeph.

**lxc stop nombre --force** — Detiene una máquina virtual de forma inmediata sin permitir que el sistema operativo ejecute su secuencia de shutdown. Debe evitarse porque deja el sistema de archivos potencialmente inconsistente.

**lxc delete nombre** — Elimina una máquina virtual completamente. Los datos almacenados en MicroCeph se liberan (el espacio vuelve a estar disponible para replicación en otros volúmenes). Esta operación es irreversible a menos que exista un snapshot anterior.

**lxc rename antiguo nuevo** — Renombra una máquina virtual. La máquina debe estar detenida. Útil si se comete error en el nombre durante creación.

#### Configuración de Máquinas Virtuales

**lxc config set nombre limits.memory 4GiB** — Modifica la memoria RAM asignada a una máquina virtual. Requiere que la máquina esté detenida. Útil cuando el software instalado necesita más recursos de los asignados por defecto.

**lxc config set nombre limits.cpu 4** — Modifica el número de CPUs asignadas. Requiere que la máquina esté detenida.

**lxc config device add nombre eth1 nic nictype=ovn network=ovn-app** — Añade una nueva interfaz de red a una máquina virtual conectándola a una red OVN específica. La máquina debe estar detenida. Útil para máquinas que necesitan estar en múltiples redes (por ejemplo backup-vm que necesita acceder a datos en ovn-app pero también reportar eventos en ovn-infra).

**lxc config device remove nombre eth0** — Elimina una interfaz de red de una máquina virtual. La máquina debe estar detenida. Es destructivo: la máquina pierde conectividad a esa red.

#### Acceso Interactivo y Ejecución Remota

**lxc exec nombre -- bash** — Abre una sesión interactiva de shell dentro de la máquina virtual con privilegios de root. No requiere SSH, claves o configuración de red previa. Es el método estándar para acceder a máquinas virtuales desde nodos físicos.

**lxc exec nombre -- bash -c "comando completo"** — Ejecuta un comando específico dentro de la máquina virtual sin abrir sesión interactiva. Retorna el output. Útil en scripts de automatización.

**lxc exec nombre -- systemctl status servicio** — Verifica el estado de un servicio dentro de la máquina virtual. Equivalente a ejecutar el comando localmente en la máquina virtual.

**lxc exec nombre -- journalctl -u servicio -f** — Muestra logs en tiempo real de un servicio específico. Útil para debugging cuando algo en la máquina virtual no funciona.

#### Transferencia de Ficheros

**lxc file push /ruta/local nombre/ruta/remota** — Transfiere un fichero desde el nodo físico a la máquina virtual. No requiere SSH ni puertos abiertos. Usa el socket interno del hipervisor LXD.

**lxc file push -r /ruta/local/ nombre/ruta/remota/** — Transfiere un directorio completo (recursivamente) desde el nodo a la máquina virtual. El flag `-r` es obligatorio para directorios.

**lxc file pull nombre/ruta/remota /ruta/local** — Transfiere un fichero desde la máquina virtual al nodo físico.

**lxc file pull -r nombre/ruta/remota/ /ruta/local/** — Transfiere un directorio completo desde la máquina virtual al nodo físico.

#### Snapshots y Recuperación

**lxc snapshot nombre nombre-snapshot** — Crea un snapshot (copia de punto-en-tiempo) de una máquina virtual. El snapshot captura el estado completo del disco en ese instante y se almacena en MicroCeph con deduplicación automática. Permite revertir a ese estado si algo falla posteriormente.

**lxc snapshot list nombre** — Lista todos los snapshots de una máquina virtual específica mostrando su nombre y fecha de creación.

**lxc restore nombre nombre-snapshot** — Revierte una máquina virtual a un snapshot anterior. La máquina debe estar detenida. Todos los cambios posteriores al snapshot se pierden.

**lxc delete nombre/nombre-snapshot** — Elimina un snapshot específico liberando espacio en MicroCeph.

#### Export e Import

**lxc export nombre /ruta/fichero.tar.gz** — Exporta una máquina virtual completa (sistema de archivos + configuración) a un archivo tar.gz. Este archivo puede transferirse a otro cluster o almacenarse como backup offline. Diferente de snapshots: el snapshot permanece dentro de MicroCeph, el export es un fichero independiente.

**lxc import /ruta/fichero.tar.gz nombre** — Importa una máquina virtual desde un archive export anterior. Útil para recuperación de desastres o replicación a otro cluster.

### Comandos microceph — Almacenamiento Distribuido

**microceph status** — Muestra el estado general del cluster Ceph: número de OSDs activos, factor de replicación, estado de salud (HEALTH\_OK, HEALTH\_WARN, HEALTH\_ERR), porcentaje de espacio utilizado, estado de rebalanceo de datos. Este es el comando más importante para monitoreo operacional.

**microceph disk list** — Lista todos los OSDs del cluster con su estado (UP/DOWN), peso de replicación (cuántos datos se asignan a cada OSD), y dispositivo correspondiente (/dev/loopX, /dev/sdX, etc.).

**microceph disk add /dev/loopX --wipe** — Agrega un nuevo disco (OSD) al cluster. El flag `--wipe` borra completamente el contenido previo del dispositivo usando ceph-bluestore-tool zap-device. El OSD se inicializa, se une al cluster y comienza a recibir datos replicados. Este proceso tarda minutos dependiendo del tamaño del disco.

**microceph disk remove /dev/loopX** — Marca un OSD como fuera de servicio y lo prepara para eliminación. Ceph automáticamente replica los datos de ese OSD a otros nodos. Una vez que la replicación completa, el OSD puede removerse físicamente.

**sudo ceph osd tree** — Muestra la estructura jerárquica del cluster: qué nodos contienen qué OSDs, pesos de replicación, estado UP/DOWN. Comando fundamental para entender la topología del cluster.

**sudo ceph osd out N** — Marca el OSD número N como fuera del cluster sin borrarlo. Ceph comienza a migrar sus datos a otros OSDs. Útil antes de una purga.

**sudo ceph osd purge N --yes-i-really-mean-it** — Elimina completamente un OSD del cluster eliminando todas sus referencias. El OSD deja de existir en la configuración de Ceph. Comando destructivo que requiere confirmación explícita.

**sudo ceph health detail** — Muestra estado de salud detallado del cluster listando cualquier problema, aviso o degradación. Si no hay problemas, reporta HEALTH\_OK.

**sudo ceph pg stat** — Muestra estadísticas de los placement groups (unidades de replicación de datos en Ceph). Útil para diagnóstico de rebalanceo lento.

### Comandos microovn — Redes Virtuales

**lxc network list** — Lista todas las redes disponibles en el cluster: redes OVN creadas (ovn-infra, ovn-app, ovn-dmz), red física (UPLINK), redes por defecto.

**lxc network create ovn-app --type=ovn network=UPLINK ipv4.address=10.10.20.1/24 ipv6.address=none ipv4.nat=true** — Crea una nueva red OVN con nombre ovn-app. El parámetro `network=UPLINK` la conecta a la red física (VLAN). `ipv4.address=10.10.20.1/24` configura el gateway OVN. `ipv4.nat=true` permite NAT automático para que máquinas virtuales en esta red accedan a internet sin IPs públicas propias. `ipv6.address=none` deshabilita IPv6 (no necesario en Spotly).

**lxc network delete nombre** — Elimina una red OVN. Las máquinas virtuales conectadas a esta red pierden conectividad. El comando falla si hay máquinas virtuales activas en la red.

**lxc network info nombre** — Muestra información detallada de una red OVN: direcciones asignadas, máquinas virtuales conectadas, ACLs activas, tipo de red.

**lxc network acl list** — Lista todas las listas de control de acceso (ACLs) definidas en el cluster.

**lxc network acl create spotly-app-policy** — Crea una nueva ACL llamada spotly-app-policy. Las ACLs definen qué tráfico está permitido entre máquinas virtuales en redes OVN.

**lxc network acl rule add spotly-app-policy ingress allow source 10.10.20.3 destination 10.10.20.2 protocol tcp destination-port 5432** — Añade una regla a una ACL permitiendo que el tráfico TCP del puerto 5432 desde app-core (10.10.20.3) llegue a data (10.10.20.2). Sin esta regla, app-core no podría conectar a PostgreSQL aunque estén en la misma red OVN.

**lxc network acl rule remove spotly-app-policy "regla-anterior"** — Elimina una regla existente de una ACL.

**lxc network acl assign ovn-app spotly-app-policy** — Aplica una ACL a una red OVN. A partir de este momento, toda comunicación en ovn-app se evalúa contra las reglas de spotly-app-policy.

**lxc network acl unassign ovn-app** — Desasocia una ACL de una red OVN. Sin ACL, el comportamiento predeterminado es permitir todo tráfico (peligroso en producción).

### Comandos de Diagnóstico

**lxc cluster list** — Muestra todos los nodos del cluster LXD con su estado, rol (líder/miembro) y participación en MicroCeph/MicroOVN.

**lxc storage list** — Lista los storage pools disponibles: remote (MicroCeph distribuido), local (almacenamiento local en cada nodo).

**lxc storage show remote** — Muestra detalles del storage pool remote: tamaño total, espacio utilizado, máquinas virtuales usando este pool, configuración de replicación.

**lxc version** — Muestra la versión instalada de LXD, la versión del protocolo, y dependencias.

**snap list | grep -E 'lxd|microceph|microovn|microcloud'** — Desde Ubuntu, lista las versiones instaladas de todos los componentes de MicroCloud. Útil para verificar que el cohort está correctamente sincronizado entre nodos.

**snap refresh --cohort="+"** — Sincroniza todos los snaps al mismo cohort de actualización. Si algunos nodos tienen versiones diferentes de microceph, el cluster reportará errores. Este comando asegura que todos los nodos avanzan juntos en actualizaciones.

### Comandos systemctl — Gestión de Servicios en Máquinas Virtuales

Los servicios dentro de las máquinas virtuales se gestionan usando systemctl. Estos comandos se ejecutan dentro de la máquina virtual, generalmente vía `lxc exec nombre -- bash -c "comando"`.

**systemctl status servicio** — Muestra el estado actual de un servicio: ACTIVE/INACTIVE, si está habilitado al arranque, PID del proceso, logs recientes. Dentro de una VM se ejecuta como `lxc exec nombre -- systemctl status postgresql`para ver PostgreSQL.

**systemctl start servicio** — Inicia un servicio detenido. Ejemplo: `lxc exec data -- systemctl start postgresql` inicia el servidor PostgreSQL dentro de data si estaba parado.

**systemctl stop servicio** — Detiene un servicio en marcha. Ejemplo: `lxc exec dmz-receiver -- systemctl stop cloudflared` detiene el túnel Cloudflare.

**systemctl restart servicio** — Detiene y reinicia un servicio. Ejemplo: `lxc exec app-core -- systemctl restart spotly-api` reinicia el backend FastAPI para aplicar cambios de configuración.

**systemctl reload servicio** — Recarga la configuración de un servicio sin detenerse completamente. No todos los servicios soportan reload. Ejemplo: `lxc exec dmz-receiver -- systemctl reload nginx` recarga la configuración de Nginx sin interrumpir conexiones activas.

**systemctl enable servicio** — Configura un servicio para iniciar automáticamente en el arranque de la máquina virtual. Ejemplo: `lxc exec data -- systemctl enable postgresql` hace que PostgreSQL inicie automáticamente cuando data arranca.

**systemctl disable servicio** — Desactiva el inicio automático de un servicio. Ejemplo: `lxc exec monitoreo -- systemctl disable prometheus` evita que Prometheus inicie al arrancar la máquina.

**systemctl daemon-reload** — Recarga la configuración del demonio systemd después de crear o modificar ficheros de servicio. Obligatorio después de editar `/etc/systemd/system/nombre.service`. Ejemplo: `lxc exec app-core -- systemctl daemon-reload`después de modificar spotly-api.service.

**journalctl -u servicio -f** — Muestra logs en tiempo real de un servicio específico. El flag `-f` mantiene la salida actualizándose continuamente (tail -f). Ejemplo: `lxc exec app-core -- journalctl -u spotly-api -f` muestra logs del backend FastAPI mientras se ejecuta.

**journalctl -u servicio --since "5 min ago"** — Muestra logs de los últimos 5 minutos. Útil para debugging reciente. Ejemplo: `lxc exec dmz-receiver -- journalctl -u cloudflared --since "5 min ago"` muestra intentos de conexión recientes del túnel.

**journalctl -u servicio -n 50** — Muestra las últimas 50 líneas de logs. Ejemplo: `lxc exec monitoreo -- journalctl -u prometheus -n 50` muestra los últimos errores de Prometheus sin cargar todo el historial.

### Verificación de Servicios en Cada Máquina Virtual

La siguiente tabla muestra qué servicios correr en cada máquina virtual y cómo verificarlos:

| Máquina Virtual | Servicios Principales        | Comando de Verificación                                          |
| --------------- | ---------------------------- | ---------------------------------------------------------------- |
| data            | postgresql                   | `lxc exec data -- systemctl status postgresql`                   |
| data            | prometheus-node-exporter     | `lxc exec data -- systemctl status prometheus-node-exporter`     |
| data            | wazuh-agent                  | `lxc exec data -- systemctl status wazuh-agent`                  |
| app-core        | spotly-api (FastAPI systemd) | `lxc exec app-core -- systemctl status spotly-api`               |
| app-core        | prometheus-node-exporter     | `lxc exec app-core -- systemctl status prometheus-node-exporter` |
| app-core        | wazuh-agent                  | `lxc exec app-core -- systemctl status wazuh-agent`              |
| dmz-receiver    | cloudflared                  | `lxc exec dmz-receiver -- systemctl status cloudflared`          |
| dmz-receiver    | traefik                      | `lxc exec dmz-receiver -- systemctl status traefik`              |
| dmz-receiver    | nginx                        | `lxc exec dmz-receiver -- systemctl status nginx`                |
| dmz-receiver    | wazuh-agent                  | `lxc exec dmz-receiver -- systemctl status wazuh-agent`          |
| monitoreo       | prometheus                   | `lxc exec monitoreo -- systemctl status prometheus`              |
| monitoreo       | loki                         | `lxc exec monitoreo -- systemctl status loki`                    |
| monitoreo       | grafana-server               | `lxc exec monitoreo -- systemctl status grafana-server`          |
| monitoreo       | wazuh-agent                  | `lxc exec monitoreo -- systemctl status wazuh-agent`             |

### Ejemplos de Ejecución Real

#### Ejemplo 1: Transferir Código del Backend a app-core

Asumir que el código del backend está en `/opt/spotly/backend/` en nodo03 y necesita actualizarse en la máquina virtual app-core.

```
# Paso 1: Copiar el código actualizado al nodo físico desde el PC de desarrollo
scp -r ~/spotly-src/backend/ camilly@192.168.20.11:/tmp/spotly-backend-new/

# Paso 2: Verificar que el directorio existe en el nodo
ssh camilly@192.168.20.11 ls -la /tmp/spotly-backend-new/

# Paso 3: Inyectar el código en la máquina virtual app-core
sudo lxc file push -r /tmp/spotly-backend-new/ app-core/opt/spotly/backend/

# Paso 4: Instalar nuevas dependencias si las hay
sudo lxc exec app-core -- bash -c "source /opt/spotly/venv/bin/activate && pip install -r /opt/spotly/backend/requirements.txt"

# Paso 5: Reiniciar el servicio para aplicar cambios
sudo lxc exec app-core -- systemctl restart spotly-api

# Paso 6: Verificar que el servicio está ejecutándose correctamente
sudo lxc exec app-core -- systemctl status spotly-api

# Paso 7: Revisar logs para detectar cualquier error de arranque
sudo lxc exec app-core -- journalctl -u spotly-api -n 20
```

#### Ejemplo 2: Ver Logs en Tiempo Real del Túnel Cloudflare

Para diagnosticar problemas de conectividad del túnel Cloudflare en dmz-receiver:

```
# Abrir sesión interactiva en dmz-receiver y ver logs en vivo
sudo lxc exec dmz-receiver -- journalctl -u cloudflared -f

# En otra terminal, revisar el estado del túnel
sudo lxc exec dmz-receiver -- systemctl status cloudflared

# Verificar que el servicio está escuchando en puerto 80
sudo lxc exec dmz-receiver -- ss -tlnp | grep -E '80|443|7844'

# Probar conectividad al dominio desde fuera del cluster
curl https://app.spotly.cat/api/health
```

#### Ejemplo 3: Transferir Base de Datos SQL a data e Importarla

Para aplicar actualizaciones del esquema de la base de datos:

```
# Paso 1: El fichero schema.sql está en /opt/spotly/schema.sql en nodo01
# Transferirlo a la máquina virtual data
sudo lxc file push /opt/spotly/schema.sql data/tmp/schema.sql

# Paso 2: Aplicar el esquema a PostgreSQL
sudo lxc exec data -- bash -c "psql -h 127.0.0.1 -U spotlyuser -d spotlydb -f /tmp/schema.sql"

# Paso 3: Verificar que las tablas fueron creadas
sudo lxc exec data -- psql -h 127.0.0.1 -U spotlyuser -d spotlydb -c '\dt'

# Paso 4: Limpiar el fichero temporal
sudo lxc exec data -- rm /tmp/schema.sql
```

#### Ejemplo 4: Monitorear Prometheus en monitoreo

Para verificar que Prometheus está scrapeando los targets correctamente:

```
# Acceder a monitoreo y ver logs de Prometheus
sudo lxc exec monitoreo -- journalctl -u prometheus -f

# Revisar el fichero de configuración
sudo lxc exec monitoreo -- cat /etc/prometheus/prometheus.yml

# Desde el nodo físico, acceder a la API de Prometheus
# (requiere estar en VLAN 20 o tener ruta a ovn-infra)
curl http://10.10.10.3:9090/api/v1/targets

# Para ver targets UP vs DOWN
curl http://10.10.10.3:9090/api/v1/targets?state=active

# Ver una métrica específica (CPU usage en app-core)
curl 'http://10.10.10.3:9090/api/v1/query?query=node_cpu_seconds_total{instance="10.10.20.3:9100"}'
```

#### Ejemplo 5: Reiniciar Servicios Después de Cambios de Configuración

Después de actualizar un fichero de configuración (por ejemplo `/etc/postgresql/15/main/postgresql.conf`):

```
# Editar la configuración dentro de la máquina virtual
sudo lxc exec data -- nano /etc/postgresql/15/main/postgresql.conf

# Reiniciar el servicio para aplicar cambios
sudo lxc exec data -- systemctl restart postgresql

# Verificar que el servicio está activo
sudo lxc exec data -- systemctl status postgresql

# Ver los últimos logs para detectar errores de configuración
sudo lxc exec data -- journalctl -u postgresql -n 30
```

#### Ejemplo 6: Crear un Snapshot de una Máquina Virtual

Para crear un punto de recuperación antes de cambios significativos:

```
# Crear un snapshot llamado pre-upgrade
sudo lxc snapshot app-core pre-upgrade

# Listar los snapshots existentes
sudo lxc snapshot list app-core

# Si algo falla, revertir al snapshot
sudo lxc stop app-core
sudo lxc restore app-core pre-upgrade
sudo lxc start app-core

# Verificar que la máquina arrancó correctamente
sudo lxc list | grep app-core
```

#### Ejemplo 7: Acceder a Node Exporter Metrics Manualmente

Para verificar que Node Exporter está funcionando en una máquina virtual:

```
# Desde el nodo físico que tiene acceso a la red OVN
# (típicamente desde el nodo donde reside la máquina virtual)
curl http://10.10.20.2:9100/metrics | head -20

# Ver métricas específicas de CPU (primeras 5 líneas)
curl http://10.10.20.3:9100/metrics | grep "node_cpu_seconds_total" | head -5

# Ver métricas de memoria
curl http://10.10.20.2:9100/metrics | grep "node_memory_"

# Verificar que el puerto está escuchando
sudo lxc exec data -- ss -tlnp | grep 9100
```

#### Ejemplo 8: Exportar una Máquina Virtual para Backup

Para crear un backup portátil de una máquina virtual que pueda restaurarse en otro cluster:

```
# Exportar la máquina virtual completa a un archivo
sudo lxc export data /mnt/backups/data-backup-2026-05-10.tar.gz

# Verificar que el archivo se creó y verificar su tamaño
ls -lh /mnt/backups/data-backup-2026-05-10.tar.gz

# Copiar el backup a un servidor remoto (backup PC)
scp /mnt/backups/data-backup-2026-05-10.tar.gz ansible@192.168.40.10:/home/ansible/backups_vms/vms/

# Para restaurar (en otro cluster):
sudo lxc import /home/ansible/backups_vms/vms/data-backup-2026-05-10.tar.gz data-restored
```

#### Ejemplo 9: Ver Errores de Wazuh Agent

Si un agente Wazuh no está reportando eventos correctamente:

```
# Revisar el estado del agente
sudo lxc exec app-core -- systemctl status wazuh-agent

# Ver los últimos errores del agente
sudo lxc exec app-core -- journalctl -u wazuh-agent -n 50

# Verificar la configuración del manager en ossec.conf
sudo lxc exec app-core -- grep -A 5 "manager-ip" /var/ossec/etc/ossec.conf

# Manualmente corregir si es necesario (variable no sustituida)
sudo lxc exec app-core -- bash -c "sed -i 's|MANAGER_IP|192.168.20.20|g' /var/ossec/etc/ossec.conf && systemctl restart wazuh-agent"

# Desde el manager (Admin PC), verificar que el agente está registrado
sudo docker exec single-node-wazuh.manager-1 /var/ossec/bin/agent_control -l
```

### Flujo Típico de Operación

Un flujo típico de operación en Spotly sería: (1) **microcloud init** en nodo1 respondiendo preguntas sobre red y uplink, (2) **microcloud join** simultáneamente en nodo2 y nodo3, (3) **microceph disk add** para los tres OSDs una vez que los loop devices existen, (4) **lxc launch ubuntu:22.04 data --vm --storage remote --network ovn-app** para crear máquinas virtuales en redes específicas, (5) **lxc file push** para transferir código y esquemas SQL, (6) **lxc exec nombre -- bash** para acceder y configurar servicios con systemctl, (7) **lxc snapshot** para crear puntos de recuperación, (8) **lxc exec nombre -- journalctl -f** para monitoreo de logs en tiempo real, (9) **microceph status** y **lxc list** para monitoreo continuo de infraestructura.
