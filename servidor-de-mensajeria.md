# Servidor de mensajería

### Servidor de mensajería XMPP con ejabberd en  Debian Trixie (lab con 2 VMs)

#### Objetivo

Implementar un servidor de mensajería instantánea basado en XMPP usando ejabberd en Ubuntu Server, crear usuarios locales, habilitar administración web y validar la comunicación desde dos máquinas virtuales distintas mediante un cliente XMPP (Pidgin).

***

### 1. Conceptos básicos

#### 1.1. ¿Qué es XMPP?

XMPP (Extensible Messaging and Presence Protocol) es un protocolo abierto para mensajería instantánea y presencia. Utiliza identificadores llamados JID (Jabber ID) con formato:

* `usuario@dominio`

El “dominio” del JID no es correo electrónico real; representa el dominio XMPP gestionado por el servidor.

#### 1.2. ¿Por qué ejabberd?

ejabberd es un servidor XMPP estable y ampliamente usado. Ofrece:

* Gestión por línea de comandos (`ejabberdctl`)
* Panel web de administración
* Soporte de módulos (MUC, upload, etc.)
* Buen soporte/documentación

***

### 2. Arquitectura del laboratorio (2 VMs)

* VM1 (Servidor): Ubuntu Server con ejabberd.
* VM2 (Cliente): otra VM (Kali o Ubuntu) con Pidgin para pruebas.
* Red: conectividad IP entre ambas VMs (ping exitoso).

Punto clave: aunque cada VM tenga su propio hostname de sistema, todas las cuentas XMPP deben usar el mismo dominio XMPP configurado en ejabberd.

***

### 3. Instalación de ejabberd (binario)

#### 3.1. Descarga del instalador

Se utilizó el instalador binario oficial `.run` para mantener la estructura típica bajo `/opt/` (coincide con el enfoque del laboratorio y facilita el control mediante `ejabberdctl`).

El instalador crea:

* `/opt/ejabberd-XX.XX/` (directorio de versión)
* En muchos casos, un enlace `/opt/ejabberd/` hacia la versión instalada

#### 3.2. Ejecución del instalador

1. Se otorgaron permisos de ejecución al instalador.
2. Se ejecutó como root.
3. Se instaló como servicio para iniciar automáticamente.

#### 3.3. Verificación del estado

Se verificó el servicio con:

* `ejabberdctl status`

***

### 4. Rutas y archivos relevantes

En el servidor:

* `/opt/ejabberd/conf/ejabberd.yml`\
  Configuración principal (YAML).
* `/opt/ejabberd/conf/server.pem`\
  Certificado TLS local (auto-firmado en laboratorio).
* `/opt/ejabberd/conf/cacert.pem`\
  Certificado/CA de apoyo.
* `/opt/ejabberd/logs/ejabberd.log`\
  Log principal para diagnóstico.

En el cliente (VM2):

* `/etc/hosts`\
  Resolución manual del dominio XMPP cuando no existe DNS interno.

En Kali (si aplica):

* `/etc/resolv.conf`\
  Configuración de DNS para resolver repositorios durante `apt update`.

***

### 5. Configuración del dominio XMPP en ejabberd

#### 5.1. Diferencia entre hostname del sistema y dominio XMPP

* **Hostname del sistema (`hostnamectl`)**: nombre interno de la máquina.
* **Dominio XMPP (`hosts` en ejabberd.yml)**: dominio lógico servido por ejabberd y utilizado por los JID.

No es obligatorio que sean idénticos, pero es esencial que el dominio XMPP:

* Sea consistente en toda la configuración.
* Se resuelva correctamente hacia la IP del servidor (DNS interno o `/etc/hosts`).

#### 5.2. Ajuste de `hosts`

En `ejabberd.yml` se definió el dominio XMPP:

```yaml
hosts:
  - aoki.lala.local
```

Este valor define el dominio válido para los usuarios y sus JID:

* `admin@aoki.lala.local`
* `punky@aoki.lala.local`

***

### 6. Configuración de permisos administrativos (ACL)

#### 6.1. Situación inicial

La configuración contenía reglas de acceso que referenciaban el rol `admin`, pero no existía una definición explícita de quién era admin en `acl:`.

#### 6.2. Definición de admin

En `ejabberd.yml` se añadió:

```yaml
acl:
  local:
    user_regexp: ""
  loopback:
    ip:
      - 127.0.0.0/8
      - ::1/128
  admin:
    user:
      - "admin@aoki.lala.local"
```

Esto habilita permisos administrativos al usuario `admin@aoki.lala.local`.

***

### 7. Ajuste de `trusted_network`

#### 7.1. Situación inicial

El fichero incluía:

```yaml
trusted_network:
  allow: loopback
```

Esto limita ciertas acciones a la propia máquina.

#### 7.2. Cambio para laboratorio

Para permitir pruebas desde otras máquinas del lab, se cambió a:

```yaml
trusted_network:
  allow: all
```

Nota: en producción, este valor debe restringirse (riesgo de abuso/registro no deseado).

***

### 8. Error 1: ACME/Let’s Encrypt fallando por dominio `.local`

#### 8.1. Síntoma

Tras aplicar cambios y reiniciar, ejabberd dejó de funcionar correctamente. `ejabberdctl status` reportaba que el nodo no estaba activo.

#### 8.2. Diagnóstico

Se revisó el log:

* `/opt/ejabberd/logs/ejabberd.log`

Apareció un error indicando que ejabberd intentaba pedir certificados mediante ACME (Let’s Encrypt) para `aoki.lala.local` y subdominios, fallando porque `.local` no es un sufijo público válido.

#### 8.3. Solución aplicada (laboratorio)

En `ejabberd.yml` se desactivó la emisión ACME:

1. En el listener HTTP (puerto 5280) se eliminó el handler:
   * `/.well-known/acme-challenge: ejabberd_acme`
2. Se eliminó o comentó el bloque `acme:` si existía.

Resultado: ejabberd inició usando certificados locales auto-firmados (válido para laboratorio).

***

### 9. Control del servicio y validación

Se utilizaron comandos de administración:

* `ejabberdctl start`
* `ejabberdctl restart`
* `ejabberdctl status`

Y en caso de fallo:

* revisión de `/opt/ejabberd/logs/`

***

### 10. Creación de usuarios locales

#### 10.1. Creación del admin

Se creó el usuario admin con:

* `ejabberdctl register admin aoki.lala.local <contraseña>`

#### 10.2. Creación de un segundo usuario

Se creó un segundo usuario para pruebas:

* `ejabberdctl register punky aoki.lala.local <contraseña>`

#### 10.3. Verificación

Se listaron usuarios registrados con:

* `ejabberdctl registered-users aoki.lala.local`

***

### 11. Acceso web de administración

#### 11.1. URL

Acceso al panel web:

* `http://IP_DEL_SERVIDOR:5280/admin`

#### 11.2. Error 2: login fallando por username incompleto

El acceso fallaba cuando se introducía solo `admin`.

Solución:\
En ejabberd el usuario administrador debe autenticarse como JID completo:

* `admin@aoki.lala.local`

Resultado: acceso correcto al panel.

***

### 12. Pruebas desde dos VMs (cliente Pidgin)

#### 12.1. Requisito: resolución del dominio XMPP

En la VM cliente, si no existe DNS interno, es necesario que el dominio XMPP resuelva al IP del servidor.

En VM2 se añadió en:

* `/etc/hosts`

Ejemplo:

* `IP_DEL_SERVIDOR aoki.lala.local`

Esto permite que el cliente XMPP resuelva el dominio del servidor.

#### 12.2. Instalación del cliente

En VM2 se instaló Pidgin.

#### 12.3. Configuración de cuenta en Pidgin

Para la cuenta `punky`:

* Username: `punky`
* Domain: `aoki.lala.local`
* Password: la definida en el registro
* Connect server: `aoki.lala.local` (o IP del servidor)
* Port: 5222

Durante la conexión se solicitó aceptar el certificado TLS auto-firmado, esperado en laboratorio.

#### 12.4. Prueba de mensajería

Para enviar mensajes se utilizó el JID completo del destinatario, por ejemplo:

* `admin@aoki.lala.local`
