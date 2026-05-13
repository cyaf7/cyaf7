# Arquitectura de la Aplicación

### Visión General

La aplicación Spotly implementa una arquitectura de tres capas: capa de presentación (frontends web y móvil), capa de lógica de negocio (API REST y WebSocket), capa de persistencia (base de datos relacional). Los datos fluyen bidireccionalemente: cambios en las plazas de aparcamiento originados en el pipeline de visión artificial se persisten en la base de datos y se notifican a los clientes en tiempo real sin polling.

La arquitectura está diseñada para tolerancia a fallos: si un nodo físico se apaga, las máquinas virtuales migran a otros nodos automáticamente sin perder datos (replicación en MicroCeph). Si el servidor de base de datos falla, los datos permanecen accesibles en MicroCeph. Si el reverse proxy falla, Cloudflare redirige el tráfico al siguiente punto de entrada disponible.

### Flujo de Datos End-to-End

#### Diagrama Visual

<figure><img src="../.gitbook/assets/Screenshot 2026-05-13 at 9.05.18 pm.png" alt=""><figcaption></figcaption></figure>



#### Descripción del Flujo

El flujo de un cambio de estado de plaza desde su origen hasta la visualización en la aplicación móvil sigue estos pasos:

**En resumen**: La cámara en el MacBook envía eventos a través de Cloudflare (4G internet) → dmz-receiver recibe y enruta a través de Traefik → app-core (FastAPI) persiste en PostgreSQL → Base de datos dispara pg\_notify → WebSocket broadcast a clientes → Actualización en tiempo real en móviles.

**Origen del evento**: Una cámara USB ELP-USBFHD01M-BL170 captura frames continuamente. El script detector.py en el MacBook de Ivan procesa frames localmente usando OpenCV y YOLOv8, calculando varianza de píxeles en las regiones de interés definidas para cada plaza. Un sistema de debounce confirma el cambio cuando el mismo estado se detecta en 3 frames consecutivos, evitando falsos positivos por sombras o movimiento transitorio.

**Transmisión al API**: Cuando se confirma un cambio, detector.py envía una petición PATCH a `https://app.spotly.cat/api/spots/{celda}` con body JSON `{"estado": "ocupado"}` o `{"estado": "libre"}`. El MacBook se conecta a internet mediante hotspot 4G sin necesidad de estar en la red del laboratorio.

**Entrada pública**: El dominio app.spotly.cat es un CNAME gestionado por Cloudflare que apunta al túnel spotly-tunnel. Cloudflare intercepta la petición HTTPS, verifica que la ruta es válida, y la reenvía a través del túnel QUIC hacia cloudflared en dmz-receiver (10.10.30.2).

**Reverse Proxy**: cloudflared recibe la petición en localhost:80. Traefik (reverse proxy) examina el PathPrefix `/api/` y la enruta hacia `http://10.10.20.3:8000` (app-core). Traefik mantiene la petición HTTPS, reescribiendo los headers necesarios para que FastAPI vea la petición como si viniera directamente del cliente.

**Lógica de Negocio**: FastAPI en app-core recibe la petición PATCH. El endpoint `/api/spots/{celda}` busca la plaza en PostgreSQL y actualiza el campo `estado_actual`. Esta actualización dispara un trigger en la base de datos que ejecuta `pg_notify('spot_change', '{"celda":"A1","estado":"ocupado"}')`.

**Persistencia**: PostgreSQL 15 en data (10.10.20.2) almacena el nuevo estado. El trigger registra el evento en una tabla de historial para auditoría y análisis. Los datos se replican automáticamente en MicroCeph a los otros dos nodos (replicación factor 3), garantizando que la información persiste aunque data falle.

**Notificación en Tiempo Real**: FastAPI tiene un proceso interno que escucha el canal pg\_notify 'spot\_change' usando `LISTEN spot_change`. Cuando la notificación llega, el WebSocket manager (ws\_manager) hace broadcast del cambio a todos los clientes WebSocket conectados mediante `ws_manager.broadcast({"event":"spot_change","data":{"celda":"A1","estado":"ocupado"}})`.

**Visualización en Cliente**: La aplicación móvil mantiene una conexión WebSocket abierta a `wss://app.spotly.cat/ws`. Cuando recibe el evento broadcast, actualiza el mapa en tiempo real sin hacer polling: la plaza A1 cambia de color inmediatamente en la pantalla del usuario. La latencia total desde que la cámara detecta el cambio hasta que se visualiza en el móvil es típicamente 2–5 segundos (procesamiento local + transmisión + enrutamiento + base de datos + broadcast).

### Componentes Principales

#### Capa de Presentación

El frontend de usuario es una aplicación React compilada con Vite y servida por Nginx en dmz-receiver puerto 3000. Conecta al API mediante peticiones REST para datos históricos y WebSocket para actualizaciones en tiempo real. Mantiene una lista de plazas, renderiza el mapa de aparcamiento como grid, colorea cada plaza según su estado (rojo = ocupado, verde = libre), y actualiza instantáneamente cuando llegan eventos WebSocket.

El frontend de administración es una segunda aplicación React en el mismo Nginx puerto 8081 que permite configurar parámetros del sistema, ver estadísticas agregadas de ocupación, gestionar usuarios y actualizar el fichero de configuración de regiones de interés (roi.json). Durante desarrollo es accesible públicamente a través del túnel, pero en producción debe estar accesible únicamente vía WireGuard VPN.

#### Capa de Lógica de Negocio

FastAPI en app-core implementa los siguientes endpoints principales:

`GET /api/health` — Endpoint de verificación de salud. Devuelve `{"status":"ok"}` si el servicio está operativo. Utilizado por balanceadores de carga y sistemas de monitoreo para verificar que el API está respondiendo.

`GET /api/spots` — Retorna todas las plazas con su estado actual, coordenadas ROI y timestamp de última actualización. Los clientes llaman este endpoint al arrancar para obtener el estado inicial del mapa.

`PATCH /api/spots/{celda}` — Actualiza el estado de una plaza específica. Origen de la petición: detector.py en MacBook. El endpoint busca la plaza en PostgreSQL, valida que el celda\_num existe, actualiza el campo estado\_actual y dispara el trigger de pg\_notify.

`GET /api/spots/{celda}/history` — Retorna el historial de cambios de una plaza específica (últimos N eventos) para análisis de ocupación por hora.

`WS /ws` — Conexión WebSocket bidirecional. Los clientes se conectan y reciben en tiempo real eventos de cambio de plazas sin polling. FastAPI mantiene un set de conexiones activas y hace broadcast a todas cuando llega una notificación pg\_notify.

Todas las peticiones son procesadas con un pool de conexiones de PostgreSQL (pool\_size=20, max\_overflow=10) que mantiene conexiones abiertas hacia data. Las queries se ejecutan con prepared statements para prevenir SQL injection.

#### Capa de Persistencia

PostgreSQL 15 en data contiene el modelo de datos relacional. La tabla central `parking_spots` tiene columnas: `id` (clave primaria autoincremental), `parking_id` (referencia a aparcamiento), `celda_num` (identificador A1, B3, etcétera), `estado_actual`(libre/ocupado), `roi_coords` (JSON con región de interés para YOLO), `is_active` (boolean para desactivar plazas temporalmente), `updated_at` (timestamp gestionado por trigger).

El trigger `trg_spots_upd` se ejecuta automáticamente en cada UPDATE de la tabla, registra el cambio en una tabla de historial y dispara `pg_notify` en el canal 'spot\_change' con los datos del evento en JSON. Esta es la mecánica que permite que FastAPI se entere inmediatamente de cambios sin polling.

### Integración de Componentes

#### Comunicación DMZ-Receiver a App-Core

dmz-receiver (en ovn-dmz) y app-core (en ovn-app) están en redes OVN separadas. dmz-receiver tiene una segunda NIC (eth1) conectada a ovn-app en IP 10.10.20.6 para permitir que Traefik alcance app-core:8000. Las reglas de firewall OVN permiten explícitamente tráfico TCP puertos 8000 y 8001 desde 10.10.20.6 a 10.10.20.3 (app-core). Sin esta segunda NIC y regla, el reverse proxy estaría en una red aislada sin acceso al backend.

#### Comunicación App-Core a Data

app-core accede a PostgreSQL en data mediante la cadena de conexión `postgresql://spotlyuser:Ghv477!.333@10.10.20.2:5432/spotlydb`. Ambas máquinas están en ovn-app (10.10.20.0/24), pero la regla de firewall OVN explícita permite únicamente tráfico TCP puerto 5432 desde app-core (10.10.20.3) a data (10.10.20.2). Otras máquinas virtuales en ovn-app (vision por ejemplo) están bloqueadas de acceder directamente a PostgreSQL.

#### Monitoreo de Rendimiento

Node Exporter en cada máquina virtual expone métricas del sistema (CPU, memoria, disco, red) en puerto 9100. Prometheus en monitoreo (10.10.20.7 en ovn-app para este propósito) hace scraping cada 15 segundos de todos los Node Exporters. Las métricas se almacenan en la base de datos TSDB de Prometheus durante 15 días.

Grafana se provisiona con Prometheus como datasource. Los dashboards muestran latencia de API (medida desde requestID), uso de conexiones a PostgreSQL, tamaño de la base de datos, tasa de replicación en MicroCeph. Las alertas de Grafana pueden enviar notificaciones (pendiente de implementar) si el API responde lentamente o si MicroCeph degrada.

#### Seguridad del Pipeline

El acceso al API desde internet pasa obligatoriamente por Cloudflare (zona DNS) y luego por Traefik en dmz-receiver. Las reglas de firewall OVN aíslan cada red: ovn-dmz solo puede hablar con ovn-app en puertos específicos. ovn-app solo puede hablar con ovn-infra en puertos específicos (por definir). ovn-infra está completamente bloqueada del exterior.

La autenticación de usuarios (futuro) se implementará con JWT en FastAPI. Toda comunicación entre componentes usa HTTPS (certificado automático de Cloudflare para app.spotly.cat) o HTTP interno cifrado por OVN. Las credenciales de PostgreSQL no se exponen en logs ni en la aplicación móvil.

### Escalabilidad Futura

La arquitectura actual está optimizada para un laboratorio educativo con 3 nodos y máximas 100–200 plazas simultáneamente monitoreadas. Para escalar a producción:

**Más Nodos**: Agregar nodos al cluster MicroCloud aumenta automáticamente la capacidad de replicación de datos (factor de replicación 4–5 es posible) y distribuye la carga de VMs.

**Caché Redis**: Implementar Redis para cachear respuestas frecuentes (lista completa de plazas) evita consultas repetidas a PostgreSQL.

**Múltiples Instancias de App-Core**: Ejecutar múltiples instancias de FastAPI detrás de Traefik con balanceo de carga round-robin permite servir más clientes WebSocket simultáneos.

**Separación de Lectura/Escritura**: PostgreSQL replica (read replicas) pueden servir lecturas de estadísticas sin competir con escrituras de cambios de estado.

**Particionamiento de Datos**: Si el número de aparcamientos crece, particionar la tabla `parking_spots` por `parking_id` permite escalado horizontal de la base de datos.

La arquitectura actual está construida considerando estas extensiones futuras: los servicios no son monolíticos, cada componente puede reemplazarse o multiplicarse independientemente.
