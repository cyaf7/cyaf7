---
icon: trillium
---

# Mattermost

## Mattermost en Debian/Ubuntu sin Docker (PostgreSQL + systemd) — lab con 2 máquinas

### Qué es Mattermost y para qué sirve

Mattermost es una plataforma de mensajería y colaboración tipo Slack (chat por canales, mensajes directos, equipos, roles, integraciones). A diferencia de ejabberd (que implementa el protocolo XMPP), Mattermost es una aplicación web centralizada que funciona sobre HTTP(S) y guarda todo en base de datos.

Por qué tiene sentido instalarlo aunque ya exista ejabberd:

* ejabberd/XMPP: protocolo abierto, clientes XMPP, enfoque más “infraestructura/protocolo”.
* Mattermost: aplicación web completa, administración desde UI, canales/equipos, auditoría, integraciones tipo webhooks/bots.

Compatibilidad (uso típico):

* Navegador web.
* Apps de escritorio y móvil (según el entorno).
* Integraciones vía HTTP (webhooks, bots, etc.).

***

### Arquitectura del laboratorio (2 máquinas)

* Servidor: 172.20.10.4
  * PostgreSQL (base de datos)
  * Mattermost (aplicación web)
* Cliente: 172.20.10.6
  * Navegador (para probar acceso remoto al servidor)

La aplicación queda accesible desde cualquier máquina que llegue por red al servidor en el puerto de Mattermost.

***

### Puertos esenciales

| Puerto   | Servicio          | Para qué se usa                                      |
| -------- | ----------------- | ---------------------------------------------------- |
| 8065/TCP | Mattermost (HTTP) | Acceso web al chat                                   |
| 5432/TCP | PostgreSQL        | Base de datos de Mattermost (normalmente solo local) |

En el laboratorio se trabajó con HTTP simple (8065). Más adelante se puede poner Nginx reverse proxy + HTTPS.

***

### Descarga del instalador y copia al servidor

#### Problema encontrado

La descarga en la VM tardaba demasiado (red lenta). Solución: descargar desde el Mac y transferir al servidor.

#### Transferencia con `scp`

`scp` copia archivos entre máquinas usando SSH (cifrado). Se usó para pasar el `.tar.gz` al servidor.

En el Mac (ejemplo conceptual):

```
scp ~/Downloads/mattermost-10.4.2-linux-amd64.tar.gz amelia@172.20.10.4:/tmp/
```

En el servidor se verificó:

```
ls -lh /tmp/mattermost-10.4.2-linux-amd64.tar.gz
```

***

### Instalación en el servidor

#### 1) Extraer en /opt

Carpeta tocada: `/opt`

Comando:

```
sudo tar -xzf /tmp/mattermost-10.4.2-linux-amd64.tar.gz -C /opt
```

Resultado esperado:

* Se crea `/opt/mattermost` (con binarios y config).

#### 2) Crear carpeta de datos

Carpeta tocada:

* `/opt/mattermost/data`

Comandos:

```
sudo mkdir -p /opt/mattermost/data
```

#### 3) Crear usuario de sistema para ejecutar el servicio

Esto NO crea usuarios “de chat”; solo crea un usuario Linux para correr el proceso con menos privilegios.

Comandos:

```
sudo useradd --system --user-group mattermost
sudo chown -R mattermost:mattermost /opt/mattermost
```

Verificación de estructura:

```
ls /opt/mattermost
```

Deberías ver (entre otras):

* `bin/`
* `config/`
* `data/`

***

### PostgreSQL para Mattermost

#### Problema encontrado

Al crear el usuario en PostgreSQL hubo un error de sintaxis porque la contraseña no estaba entre comillas.

#### 1) Entrar a psql

```
sudo -u postgres psql
```

#### 2) Crear base y usuario

Dentro de `psql`, los comandos correctos (nota: contraseña entre comillas):

```
CREATE DATABASE mattermost;
CREATE USER mmuser WITH PASSWORD 'camilly';
ALTER DATABASE mattermost OWNER TO mmuser;
\q
```

***

### Configuración de Mattermost (config.json)

Archivo tocado:

* `/opt/mattermost/config/config.json`

#### 1) Editar `DataSource` (SqlSettings)

Se configuró para PostgreSQL local:

```
sudo nano /opt/mattermost/config/config.json
```

En la sección `"SqlSettings"`:

```
"DriverName": "postgres",
"DataSource": "postgres://mmuser:camilly@localhost:5432/mattermost?sslmode=disable"
```

#### Problema encontrado: JSON inválido

Mattermost falló por un error de parsing del JSON.

Se validó con:

```
sudo python3 -m json.tool /opt/mattermost/config/config.json > /dev/null
```

Se corrigió el error de sintaxis (faltaba una coma u otro separador según el mensaje: `Expecting ',' delimiter...`).

***

### Servicio systemd

Archivo tocado:

* `/etc/systemd/system/mattermost.service`

#### 1) Crear el servicio

```
sudo nano /etc/systemd/system/mattermost.service
```

Contenido base usado:

```
[Unit]
Description=Mattermost
After=network.target postgresql.service
Requires=postgresql.service

[Service]
Type=notify
User=mattermost
Group=mattermost
WorkingDirectory=/opt/mattermost
ExecStart=/opt/mattermost/bin/mattermost server
TimeoutStartSec=360
LimitNOFILE=49152
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

#### Problema encontrado: `ExecStart` incorrecto

Si `ExecStart` se dejaba sin `server`, el binario mostraba ayuda y salía con código 1. Se corrigió para que fuera:

* `ExecStart=/opt/mattermost/bin/mattermost server`

#### 2) Activar y arrancar

```
sudo systemctl daemon-reload
sudo systemctl enable mattermost
sudo systemctl start mattermost
```

Ver estado:

```
sudo systemctl status mattermost --no-pager
```

***

### Error de permisos en PostgreSQL (schema public)

#### Problema encontrado

Al arrancar, Mattermost conectaba a la DB pero fallaba migraciones:

* `pq: permission denied for schema public`

Se solucionó dando permisos al usuario en el esquema.

Entrar a psql:

```
sudo -u postgres psql
```

Ejecutar (línea por línea):

```
\c mattermost
GRANT ALL PRIVILEGES ON DATABASE mattermost TO mmuser;
GRANT USAGE, CREATE ON SCHEMA public TO mmuser;
ALTER SCHEMA public OWNER TO mmuser;
\q
```

Reiniciar:

```
sudo systemctl restart mattermost
sudo systemctl status mattermost --no-pager
```

***

### Verificación de escucha y acceso

#### 1) Confirmar puerto 8065 escuchando

```
sudo ss -lntp | grep 8065
```

#### 2) Probar localmente (en el servidor)

```
curl -I http://localhost:8065 | head -n 5
```

Resultado esperado:

* `HTTP/1.1 200 OK`

***

### Probar desde 2 máquinas (cliente 172.20.10.6)

En el cliente, abrir en navegador:

* `http://172.20.10.4:8065`

Si no resuelve por nombre y quieres usar dominio, puedes usar `/etc/hosts` en el cliente (opcional):

```
sudo nano /etc/hosts
```

Y añadir:

```
172.20.10.4 aoki.lala.local
```

Entonces podrías abrir:

* `http://aoki.lala.local:8065`

***

### Carpetas y archivos que se tocaron

Servidor (172.20.10.4):

* `/tmp/`
  * se copió el tar.gz aquí antes de extraer
* `/opt/mattermost/`
  * instalación de la app
* `/opt/mattermost/config/config.json`
  * configuración principal (DB)
* `/opt/mattermost/data/`
  * datos de la app
* `/etc/systemd/system/mattermost.service`
  * servicio systemd
* PostgreSQL:
  * DB `mattermost`, usuario `mmuser`

Cliente (172.20.10.6):

* Navegador para probar acceso
* (Opcional) `/etc/hosts` si se quiere usar nombre en vez de IP

***

### Checklist rápido de troubleshooting

* El servicio no arranca:
  * `sudo systemctl status mattermost --no-pager`
  * `sudo journalctl -u mattermost -n 120 --no-pager`
* Error de JSON:
  * `sudo python3 -m json.tool /opt/mattermost/config/config.json > /dev/null`
* Error DB permisos/migraciones:
  * revisar grants en `public`
* No abre desde otra máquina:
  * `curl -I http://localhost:8065`
  * `ss -lntp | grep 8065`
  * comprobar conectividad/red entre máquinas

### &#x20;A tener en cuenta:

## Creación y gestión de usuarios en Mattermost (laboratorio)

#### 1. Cómo funciona realmente la creación de usuarios

En Mattermost existen 3 modos posibles:

1. Open Server → cualquier persona puede crear cuenta.
2. Invitación por email → requiere SMTP configurado.
3. Creación manual por administrador desde System Console.

En el laboratorio no configuramos SMTP, por lo tanto:

* No se pueden enviar invitaciones por correo.
* El registro depende de si el servidor permite "Open Server".

***

#### 2. Por qué no puedes crear usuarios

Si al cerrar sesión no aparece el botón “Create Account”, significa que:

* El servidor NO está en modo Open.
* Solo los administradores pueden crear usuarios.

Eso es configuración de seguridad, no un error.

***

#### Solución 1 (recomendada en laboratorio)

Activar registro abierto temporalmente.

#### Paso 1

Entrar como admin (camilly).

Ir a:

System Console\
→ Authentication\
→ Email

Buscar opción:

Enable Open Server

Activarla.

Guardar cambios.

***

#### Paso 2

Cerrar sesión.

En la pantalla inicial ahora debería aparecer:

Create Account

Crear por ejemplo:

Username: ivan\
Email: ivan@lala.local\
Password: cualquier contraseña

***

#### Si NO aparece esa opción

Entonces lo hacemos directamente por base de datos.

***

#### Solución 2 — Verificación técnica en PostgreSQL

Primero ver usuarios actuales:

```
sudo -u postgres psql -d mattermost -c "SELECT username, email, roles FROM users;"
```

Si solo aparece camilly, significa que no hay más usuarios creados.

***

#### Método alternativo: crear usuario desde línea de comandos (si UI falla)

Algunas versiones nuevas usan subcomando db, no user.

Primero probar:

```
sudo -u mattermost /opt/mattermost/bin/mattermost db --help
```

Si no hay comando de usuario, la única vía es interfaz web.

***

#### 3. Cómo hacer que dos usuarios se comuniquen

Una vez creado segundo usuario:

1. Ambos deben estar en el mismo Team.
2. Ir a:
   * Town Square (canal público), o
   * Crear canal nuevo.

Para mensaje directo:

* Click en el nombre del usuario.
* Start Direct Message.

No se necesita configuración adicional.

***

#### Importante entender

En Mattermost:

* Los usuarios no son del sistema Linux.
* No se crean con useradd.
* No se crean en PostgreSQL manualmente.
* Se crean desde la aplicación.

PostgreSQL solo almacena datos.
