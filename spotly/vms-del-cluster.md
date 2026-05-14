# vms del cluster

### 3.2 VM: data — Base de Datos PostgreSQL

La máquina virtual data es la capa de persistencia del sistema. Aloja PostgreSQL 15, que almacena el estado de todas las plazas de aparcamiento, historial de cambios, información de usuarios y toda la información estructurada del dominio de negocio. Es la única máquina virtual autorizada a contener datos permanentes.

PostgreSQL escucha en 0.0.0.0:5432 pero solo acepta conexiones autenticadas desde la subred ovn-app (10.10.20.0/24). Las reglas de firewall OVN limitan el acceso únicamente desde app-core (10.10.20.3). El esquema contiene 19 tablas, siendo la central parking\_spots que almacena para cada plaza: identificador, número de celda (A1, A2, etcétera), estado (libre u ocupado), coordenadas ROI en JSON para detección YOLO, indicador de plaza activa, y timestamp de última modificación gestionado por triggers que disparan pg\_notify.

Node Exporter expone métricas en puerto 9100. El servicio monitoreo (10.10.20.7 en ovn-app) hace scraping cada 15 segundos. El agente Wazuh envía eventos de seguridad al Admin PC (192.168.20.20:1514) y fue instalado mediante Ansible configurando automáticamente la dirección del manager.

| Parámetro         | Valor                   |
| ----------------- | ----------------------- |
| IP OVN            | 10.10.20.2              |
| Red OVN           | ovn-app                 |
| Nodo físico       | nodo01                  |
| Sistema operativo | Ubuntu Server 22.04 LTS |
| Almacenamiento    | Pool remote — MicroCeph |
| Estado            | RUNNING                 |

### 3.3 VM: app-core — Backend FastAPI

La máquina virtual app-core es el núcleo de la lógica de negocio. Aloja el API REST desarrollado con FastAPI que recibe eventos de cambio de estado de plazas desde el pipeline de cámara, los persiste en PostgreSQL, y notifica a clientes mediante WebSocket. Es el único intermediario entre la capa pública (dmz-receiver) y la capa de datos (data).

FastAPI ejecuta como servicio systemd llamado spotly-api en el puerto 8000, escuchando en 0.0.0.0. El código fue transferido desde la máquina de desarrollo al nodo físico mediante SCP, y del nodo a la máquina virtual mediante lxc file push. Las dependencias Python se instalan en un entorno virtual ubicado en /opt/spotly/venv/.

El servicio mantiene un pool de conexiones a PostgreSQL (10.10.20.2:5432) que verifica como operativo en los logs de arranque. WebSocket gestiona conexiones abiertas con clientes móviles en la misma instancia Uvicorn. Cuando PostgreSQL detecta un cambio mediante trigger, dispara pg\_notify en el canal spot\_change, el proceso LISTEN de FastAPI recibe la notificación, y ws\_manager hace broadcast del evento a todos los clientes conectados, actualizando mapas en tiempo real sin polling.

Node Exporter expone métricas en puerto 9100 y es scrapeado por monitoreo (10.10.20.7). El agente Wazuh fue instalado vía Ansible y envía eventos al Admin PC:1514. Las reglas de firewall OVN permiten únicamente tráfico desde dmz-receiver (10.10.20.6 en ovn-app) hacia los puertos 8000 y 8001, bloqueando acceso desde vision a la base de datos (vision solo puede comunicarse con app-core, no directamente con data).

| Parámetro         | Valor                   |
| ----------------- | ----------------------- |
| IP OVN            | 10.10.20.3              |
| Red OVN           | ovn-app                 |
| Nodo físico       | nodo03                  |
| Sistema operativo | Ubuntu Server 22.04 LTS |
| Puerto API        | 8000 TCP                |
| Almacenamiento    | Pool remote — MicroCeph |
| Estado            | RUNNING                 |
| Servicio systemd  | spotly-api              |

### 3.4 VM: dmz-receiver — Punto de Entrada Público

La máquina virtual dmz-receiver es el único punto de contacto entre internet y la infraestructura interna. Reside en la red OVN de demilitarización (ovn-dmz) y concentra todo el tráfico público entrante. Ejecuta tres servicios: cloudflared mantiene el túnel permanente hacia Cloudflare, Traefik actúa como reverse proxy distribuidor de tráfico según rutas HTTP, Nginx sirve los ficheros estáticos de los dos frontends.

cloudflared establece una conexión saliente persistente hacia Cloudflare usando QUIC (UDP 7844) con fallback automático a HTTPS (TCP 443) si UDP no está disponible. El tráfico externo entra por Cloudflare y llega a la máquina virtual a través de ese túnel. El firewall OPNsense no requiere ninguna regla de entrada adicional. El dominio spotly.cat está registrado en Nominalia con nameservers delegados a Cloudflare. El túnel spotly-tunnel está activo en Zero Trust (cuenta spotly.cci@gmail.com, equipo spottly, plan Free) y en estado Healthy.

Traefik distribuye rutas: /api/\* → app-core:8000 (API REST), /ws/\* → app-core:8000 (WebSocket), /admin\* → localhost:8081 (frontend admin), /\* → localhost:3000 (frontend usuario). Nginx sirve las SPAs React compiladas en dos puertos separados. El acceso al panel admin es solo temporal durante desarrollo y debe ser accesible únicamente vía WireGuard VPN en producción.

La máquina tiene dos NICs: eth0 en ovn-dmz (10.10.30.2) recibe tráfico de Cloudflare, eth1 en ovn-app (10.10.20.6) permite a Traefik reenviar peticiones al backend en app-core. La segunda NIC fue añadida con la máquina detenida porque LXD no soporta hotplug de interfaces OVN. El agente Wazuh fue instalado mediante Ansible y fue transferido desde la caché de apt de app-core porque dmz-receiver no tenía acceso a repositorios en ese momento.

| Parámetro               | Valor                          |
| ----------------------- | ------------------------------ |
| IP principal (ovn-dmz)  | 10.10.30.2                     |
| IP secundaria (ovn-app) | 10.10.20.6                     |
| Nodo físico             | nodo02                         |
| Sistema operativo       | Ubuntu Server 22.04 LTS        |
| Servicios               | cloudflared, Traefik v3, Nginx |
| Dominio público         | app.spotly.cat                 |
| Estado del túnel        | Healthy                        |
| Almacenamiento          | Pool remote — MicroCeph        |
| Estado                  | RUNNING                        |

### 3.5 VM: vision — Procesamiento de Imágenes (Reservada)

La máquina virtual vision está reservada para alojar el pipeline de visión artificial cuando se decida migrar el procesamiento desde la máquina del equipo de visión al cluster. Actualmente la máquina está en estado RUNNING pero sin ningún software de aplicación instalado.

El pipeline de detección de plazas (YOLOv8 más OpenCV) se ejecuta actualmente en el MacBook del equipo de visión artificial, que procesa frames de la cámara USB ELP-USBFHD01M-BL170 (170 grados fisheye, 1080p) localmente. El script detector.py calcula varianza de píxeles en las regiones de interés definidas para cada plaza, implementa un sistema de debounce que confirma cambios al detectarlos en 3 frames consecutivos para evitar falsos positivos, y envía peticiones PATCH al API de app-core en app.spotly.cat con el nuevo estado de la plaza. El MacBook no necesita estar en la red del laboratorio — se conecta mediante hotspot 4G y accede al sistema a través del túnel Cloudflare.

Cuando se implemente el pipeline en la máquina virtual, el proceso seguirá el mismo patrón que las otras máquinas: transferencia de código mediante lxc file push, creación de entorno virtual Python, instalación de ultralytics (YOLOv8), opencv-python-headless para procesamiento sin GUI, y httpx para cliente HTTP. Node Exporter y el agente Wazuh serán instalados en ese momento según el procedimiento estándar.

Las reglas de firewall OVN permitirán tráfico desde vision hacia app-core en puerto 8000, bloqueando acceso directo a la base de datos en data. El acceso a internet será permitido para descarga del modelo YOLOv8 (\~6 MB, descargado automáticamente en primer uso) y actualizaciones de paquetes.

| Parámetro               | Valor                   |
| ----------------------- | ----------------------- |
| IP OVN                  | 10.10.20.4              |
| Red OVN                 | ovn-app                 |
| Nodo físico             | nodo01                  |
| Sistema operativo       | Ubuntu Server 22.04 LTS |
| Almacenamiento          | Pool remote — MicroCeph |
| Estado                  | RUNNING sin servicios   |
| Ubicación procesamiento | MacBook (temporalmente) |

### 3.6 VM: monitoreo — Stack de Observabilidad

La máquina virtual monitoreo aloja el stack completo de observabilidad del sistema: Prometheus para métricas numéricas, Grafana para visualización y alertas, Loki para centralización de logs. Reside en la red ovn-infra que agrupa servicios internos que nunca deben ser accesibles desde internet.

La máquina tiene dos NICs: interfaz principal en ovn-infra (10.10.10.3) es alcanzable desde el Admin PC, interfaz secundaria en ovn-app (10.10.20.7) permite hacer scraping de Node Exporter en las máquinas virtuales de aplicación. Prometheus fue configurado con un scrape interval de 15 segundos. Los targets incluyen Node Exporter en las máquinas virtuales de aplicación (10.10.20.2, 10.10.20.3, 10.10.20.4, 10.10.10.3) que están UP, y los nodos físicos (192.168.20.11, 192.168.20.12, 192.168.20.13, 192.168.20.20) que aparecen DOWN porque la máquina monitoreo está en una red OVN overlay sin ruta directa a las VLANs físicas.

Loki implementa agregación de logs indexando solo metadatos sin indexar el contenido completo, lo que es significativamente más eficiente en almacenamiento que soluciones basadas en Elasticsearch. Grafana se provisiona automáticamente con el datasource de Prometheus usando acceso proxy (el servidor hace las peticiones a Prometheus, no el navegador del usuario, porque localhost:9090 no es alcanzable desde fuera de la máquina virtual).

El agente Wazuh fue instalado mediante Ansible. En la sesión del 24–26 de abril se detectó que el IP del manager estaba configurado como la cadena literal 'MANAGER\_IP' porque el playbook Ansible no sustituyó la variable correctamente. Fue corregido manualmente con sed para reescribir el fichero ossec.conf del agente.

| Parámetro                | Valor                                  |
| ------------------------ | -------------------------------------- |
| IP principal (ovn-infra) | 10.10.10.3                             |
| IP secundaria (ovn-app)  | 10.10.20.7                             |
| Nodo físico              | nodo03                                 |
| Sistema operativo        | Ubuntu Server 22.04 LTS                |
| Memoria                  | 3 GiB                                  |
| Almacenamiento           | Pool remote — MicroCeph                |
| Estado                   | RUNNING                                |
| Servicios                | Prometheus 2.51.2, Loki 3.0.0, Grafana |
| Puerto Prometheus        | 9090                                   |
| Puerto Loki              | 3100                                   |
| Puerto Grafana           | 3000                                   |

### Acceso a Máquinas Virtuales

El acceso remoto a máquinas virtuales se realiza mediante `lxc exec` desde un nodo físico. Por ejemplo, para abrir una sesión shell en data desde nodo01: `lxc exec data -- bash`. Este comando abre una sesión interactiva con privilegios de root sin requerir SSH, claves o conectividad de red previa entre máquinas virtuales.

Las máquinas virtuales pueden configurarse con SSH, pero solo desde VLAN 20 (management) a través del firewall de OPNsense. El acceso directo desde internet nunca es permitido: todo tráfico externo pasa obligatoriamente por dmz-receiver.

### Snapshots y Recuperación

Cada máquina virtual puede tener snapshots (copias de punto-en-tiempo) tomadas bajo demanda o según cronograma. Un snapshot captura el estado completo del disco en un instante. Los snapshots se almacenan en MicroCeph con versioning automático: se pueden mantener múltiples snapshots sin penalidad de espacio porque MicroCeph deduplica automáticamente bloques idénticos.

El sistema de backup automático (ejecutado desde Backup PC en VLAN 40) exporta máquinas virtuales completas a archivos tar.gz. Esto permite recuperación de desastres sin acceso a los snapshots internos de LXD: si el cluster de MicroCeph falla completamente, los archives tar.gz en Backup PC contienen versiones anteriores de todas las máquinas virtuales.
