# Herramientas del Cluster — Stack de Aplicación Completo

### Introducción

Spotly está construido sobre un stack de tecnologías de código abierto que trabajan en conjunto para procesar imágenes, almacenar datos, servir APIs y exponer servicios al público. Este documento describe cada herramienta, su propósito en Spotly, y cómo se relacionan entre sí.



### FastAPI — Backend REST y WebSocket

FastAPI es un framework web Python de alto rendimiento para construir APIs REST y WebSocket. En Spotly, FastAPI corre en la VM app-core (10.10.20.3) escuchando en puerto 8000.

**Funciones en Spotly:**

* **POST /stream/push**: recibe frames base64 codificados de la cámara, los almacena en frame\_buffer en memoria
* **GET /stream**: WebSocket bidireccional que emite eventos de cambio de estado de plazas en tiempo real a clientes
* **PATCH /api/spots/{celda}**: recibe cambios de estado (occupied/free) del detector de visión
* **GET /api/spots**: lista estado actual de todas las plazas
* **WebSocket /ws**: conexión persistente para clientes móviles que reciben eventos en tiempo real

FastAPI genera Swagger automáticamente: `http://app-core:8000/docs` muestra documentación interactiva.

**Deployment:**

Código en `/opt/spotly/backend/`, entorno virtual en `/opt/spotly/venv/`. Servicio systemd `spotly-api` inicia el proceso. Traefik en dmz-receiver proxea `/api/*` hacia `app-core:8000`.

**Dependencias Python:**

fastapi, uvicorn (servidor ASGI), psycopg2 (cliente PostgreSQL), python-dotenv (variables de entorno), pydantic (validación de datos).

### PostgreSQL 15 — Base de Datos Relacional

PostgreSQL 15 corre en VM data (10.10.20.2) en puerto 5432. Almacena:

* **Tabla spots**: id (PK), celdaId, estado (occupied/free), timestamp, ubicación geográfica
* **Tabla eventos**: id (PK), spotId (FK), tipo (ocupación/liberación), timestamp, confianza (ML)
* **Tabla usuarios**: id (PK), email, nombre, role, created\_at
* **Tabla notificaciones**: id (PK), userId (FK), tipo, mensaje, leído, timestamp
* **Vistas**: espacios\_disponibles (COUNT de spots con estado=free), eventos\_últimos\_minutos (filtrado por tiempo)
* **Triggers**: al actualizar estado de plaza, disparar `pg_notify` hacia FastAPI

**Acceso:**

Solo desde ovn-app (10.10.20.0/24). Método autenticación: scram-sha-256. Usuario: spotlyuser. Database: spotlydb.

**Integración con FastAPI:**

FastAPI usa psycopg2 con pool de conexiones. Cuando se actualiza una plaza, el trigger PostgreSQL ejecuta `NOTIFY spotly_updates, 'plaza:celda1:free'`. FastAPI tiene un listener en LISTEN spotly\_updates que recibe la notificación y emite a todos los clientes WebSocket conectados.

**Backups:**

Exportado diariamente con `pg_dump -h 10.10.20.2 -U spotlyuser spotlydb > spotlydb-FECHA.sql`. Restore: `psql -h 10.10.20.2 -U spotlyuser spotlydb < spotlydb-FECHA.sql`.

### YOLO + OpenCV — Visión Artificial

YOLO (You Only Look Once) es un modelo de detección de objetos entrenado en millones de imágenes para reconocer coches, personas, bicicletas, etc. OpenCV es la librería que captura frames de la cámara, las procesa, y ejecuta YOLO.

**Dónde se ejecuta:**

El pipeline de visión no corre en el cluster Spotly; corre en el MacBook de Ivan via la VM vision (10.10.20.4) como entorno validación. La detección actualmente ocurre en MacBook con 4G hotspot que envía PATCH app.spotly.cat/api/spots/{celda} cuando detecta cambios.

**Script: detector.py (en MacBook)**

```
while True:
    frame = camera.read()
    boxes = yolo.detect(frame)  # inferencia
    for box in boxes:
        celda_id = gps_to_celda(box.location)
        if is_ocupada(box) and not was_ocupada_last_3_frames:
            send_PATCH(app.spotly.cat, celda_id, 'occupied')
```

El script requiere detección consistente en 3 frames consecutivos antes de reportar cambio (evitar falsos positivos).

**Validación en vision VM:**

La VM vision (10.10.20.4) almacena modelos YOLO preentrenados para pruebas offline. Las librerías ultralytics y PyTorch están instaladas ahí pero el pipeline principal usa el MacBook.

### Traefik — Reverse Proxy Inverso

Traefik v3 corre en dmz-receiver (10.10.30.2) en puerto 80. Es el único punto de entrada a la aplicación desde el exterior (el público accede a través de app.spotly.cat que enruta a Traefik).

**Funciones de Traefik:**

* Escucha conexiones HTTP/HTTPS en localhost:80
* Analiza headers (Host, Path)
* Enruta a servicios internos según reglas configuradas
* Redirige HTTP → HTTPS automáticamente
* Inserta headers de seguridad (CORS, X-Frame-Options)
* Comprime respuestas (gzip)

**Configuración de Rutas:**

```
app.spotly.cat/         → Nginx frontend-user:3000
app.spotly.cat/api/*    → FastAPI app-core:8000
app.spotly.cat/admin*   → Traefik dashboard:8081 (opcional)
app.spotly.cat/grafana* → Grafana monitoreo:3000
```

Las rutas se definen en `/etc/traefik/traefik.yaml` o `docker-compose.yml` con labels.

**Certificados SSL:**

Cloudflare emite certificados Let's Encrypt automáticamente. Traefik confía en Cloudflare como proxy HTTPS y comunica internamente con app-core via HTTP (seguro porque está en localhost).

### Nginx — Servidor Web Frontend

Nginx corre en dmz-receiver (misma VM que Traefik) sirviendo la interfaz web (`frontend-user`) en puerto 3000. El código frontend es una SPA (Single Page Application) en Vue.js o React.

**Contenido Servido:**

* index.html (application shell)
* bundle.js (código JavaScript compilado)
* CSS (estilos)
* Assets estáticos (imágenes, iconos)

**Comunicación con Backend:**

JavaScript en el navegador hace peticiones fetch/axios hacia `/api/*` (enrutadas por Traefik a FastAPI). WebSocket se conecta a `/stream` para recibir actualizaciones en tiempo real del estado de plazas.

### Wazuh — Monitoreo de Seguridad

Wazuh es una plataforma de monitoreo de seguridad y conformidad que detecta malware, intrusiones, cambios no autorizados de archivos. En Spotly:

**Wazuh Manager** corre en Admin PC como contenedor Docker (imagen oficial v4.7.2). Escucha en puerto 1514 TCP/UDP.

**Wazuh Agents** se instalan en todos los nodos (nodo01, nodo02, nodo03) y VMs (data, app-core, dmz-receiver, monitoreo, ldap-server). Los agentes reportan logs, cambios de archivo, y alertas al Manager.

**Funciones de Wazuh:**

* Monitoreo de archivos críticos (/etc/ssh/sshd\_config, Netplan): alerta si se modifica
* Detección de intrusión: analiza logs de fail2ban, SSH, acceso a archivos sensibles
* Monitoreo de integridad: suma MD5 de binarios críticos, alerta si cambian
* Recolección de logs: centraliza logs de systemd, aplicaciones
* Alertas: dispara cuando detecta anomalías

**Acceso al Dashboard:**

Desde Admin PC (máquina donde corre el Manager). Usuario y contraseña generados durante instalación Docker.

### Redis — Caché en Memoria (Futuro)

Redis aún no está implementado en Spotly, pero está pendiente. Se usaría para:

* Cachear resultados de consultas frecuentes (spots disponibles por zona)
* Almacenar sesiones de usuarios
* Cola de tareas de background (procesamiento de imágenes)

Se desplegaría en una VM adicional en ovn-app, o como pod Podman en app-core.

### Prometheus + Grafana + Loki — Stack de Observabilidad

**Prometheus 2.51.2** (monitoreo, 10.10.10.3): Scraping de métricas desde endpoints Prometheus cada 15 segundos. Targets:

* Node Exporter en nodos (estado down actualmente, sin ruta desde OVN)
* cAdvisor en nodos (opcional, no instalado)
* Métricas de aplicación custom desde FastAPI

**Grafana** (10.10.10.3, puerto 3000): Dashboard que visualiza métricas de Prometheus y logs de Loki. Datasource Prometheus configurado con access=proxy (servidor Grafana hace peticiones, no browser directamente).

**Loki 3.0.0** (10.10.10.3): Sistema de logs distribuido. Promtail (agente) en cada nodo lee logs de systemd y los envía a Loki. Loki los indexa por labels (hostname, servicio) permitiendo búsquedas rápidas desde Grafana.

### Relación Entre Componentes

**Flujo de una Solicitud:**

1. Cliente (navegador móvil) abre app.spotly.cat
2. DNS resuelve a Cloudflare → enruta a dmz-receiver via túnel
3. cloudflared en dmz-receiver enruta a Traefik:80
4. Traefik: Host=app.spotly.cat, Path=/ → Nginx:3000
5. Nginx devuelve frontend-user (HTML + JS)
6. JS ejecuta: fetch('/api/spots') → Traefik → FastAPI:8000
7. FastAPI consulta PostgreSQL: SELECT \* FROM spots
8. PostgreSQL devuelve JSON → FastAPI → Traefik → Cliente
9. JS conecta WebSocket /stream → Traefik → FastAPI
10. Cuando cámara detecta cambio: PATCH /api/spots/celda1 → FastAPI → PostgreSQL trigger → pg\_notify → WebSocket broadcast → Cliente recibe actualización en tiempo real

**Monitoreo Paralelo:**

Prometheus scraping Node Exporter en nodos → Grafana visualiza CPU/memoria → Alertas si excesan umbrales. Logs de fail2ban en nodos → Loki → búsqueda en Grafana. Wazuh monitorea integridad de archivos → alerta si algo cambia.
