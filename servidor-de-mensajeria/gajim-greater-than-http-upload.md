# Gajim -> HTTP Upload

## Introducción

En este laboratorio se implementó un servidor de mensajería basado en XMPP utilizando ejabberd en Ubuntu Server, con validación completa de mensajería y transferencia de archivos mediante XEP-0363 (HTTP Upload).

El objetivo fue comprender no solo la configuración funcional, sino también los problemas reales relacionados con certificados TLS, resolución de nombres, IPv6, y validación de autoridad certificadora en clientes modernos.

***

## 1. Fundamentos teóricos

### 1.1 ¿Qué es XMPP?

XMPP (Extensible Messaging and Presence Protocol) es un protocolo abierto para mensajería instantánea y presencia.

Funciona de forma similar al correo electrónico:

Cada usuario tiene un identificador llamado JID:

```
usuario@dominio
```

El dominio no es correo real: es el dominio XMPP gestionado por el servidor.

En este laboratorio:

```
dominio = aoki.lala.local
```

Ejemplo de cuentas:

```
admin@aoki.lala.local
ivan@aoki.lala.local
```

***

### 1.2 ¿Qué es ejabberd?

ejabberd es un servidor XMPP escrito en Erlang que:

* Gestiona autenticación de usuarios
* Maneja presencia
* Permite mensajería 1 a 1 y MUC
* Integra servicios HTTP internos
* Soporta XEP-0363 (HTTP Upload)

Archivo principal de configuración:

```
/opt/ejabberd/conf/ejabberd.yml
```

***

### 1.3 ¿Qué es Gajim?

Gajim es un cliente XMPP moderno que:

* Soporta TLS estricto
* Implementa XEP-0363 correctamente
* Verifica certificados con SAN
* Rechaza HTTP inseguro para upload

Es importante porque clientes antiguos como Pidgin no manejan correctamente upload moderno.

***

### 1.4 ¿Cómo funciona XEP-0363 (HTTP Upload)?

XMPP no envía archivos directamente por el puerto 5222.

El flujo real es:

1. El cliente solicita un slot de subida vía XMPP.
2. El servidor responde con una URL temporal HTTPS.
3. El cliente sube el archivo por HTTPS.
4. El servidor genera un enlace.
5. El enlace se envía por XMPP al destinatario.

Esto requiere:

* Endpoint HTTP activo
* TLS válido
* Certificado confiable

***

## 2. Arquitectura del laboratorio

### Máquina 1 – Servidor

Ubuntu Server\
ejabberd instalado en:

```
/opt/ejabberd
```

Dominio XMPP:

```
aoki.lala.local
```

### Máquina 2 – Cliente

Debian\
Cliente XMPP: Gajim

Ambas máquinas deben:

* Poder comunicarse por red
* Resolver el dominio correctamente
* Tener coherencia entre dominio XMPP y certificado TLS

***

## 3. Puertos utilizados

| Puerto | Protocolo | Uso                                   |
| ------ | --------- | ------------------------------------- |
| 5222   | TCP       | XMPP cliente-servidor                 |
| 5280   | TCP       | HTTP panel web                        |
| 5443   | TCP       | HTTPS upload y servicios HTTP seguros |

Importante:

El upload moderno exige HTTPS.\
Por eso se usó finalmente el puerto 5443.

***

## 4. Configuración completa del servidor ejabberd

***

### 4.1 Dominio XMPP

En `ejabberd.yml`:

```yaml
hosts:
  - aoki.lala.local
```

Esto define el dominio de todas las cuentas.

***

### 4.2 Listener HTTPS (obligatorio para upload)

```yaml
- port: 5443
  ip: "::"
  module: ejabberd_http
  tls: true
  request_handlers:
    /admin: ejabberd_web_admin
    /upload: mod_http_upload
```

Esto habilita:

* TLS
* Endpoint `/upload`
* Panel admin HTTPS

***

### 4.3 Módulo mod\_http\_upload

```yaml
mod_http_upload:
  put_url: https://@HOST@:5443/upload
  get_url: https://@HOST@:5443/upload
  max_size: 10485760
```

Esto indica:

* Upload vía HTTPS
* Uso del dominio definido
* Límite 10MB

***

### 4.4 Certificado TLS correcto (SAN obligatorio)

El certificado debía incluir:

```
DNS:aoki.lala.local
```

Se generó con:

```bash
sudo openssl req -new -x509 -days 365 -nodes \
-keyout /opt/ejabberd/conf/server.pem \
-out /opt/ejabberd/conf/server.pem \
-subj "/CN=aoki.lala.local" \
-addext "subjectAltName=DNS:aoki.lala.local"
```

Reinicio:

```
sudo systemctl restart ejabberd
```

Verificación:

```
openssl x509 -in server.pem -noout -text | grep -A2 "Subject Alternative Name"
```

***

## 5. Instalación y configuración de Gajim

***

### 5.1 Instalación

En Debian:

```
sudo apt update
sudo apt install gajim
```

***

### 5.2 Configuración básica

Datos:

| Campo    | Valor                |
| -------- | -------------------- |
| Username | ivan                 |
| Domain   | aoki.lala.local      |
| Password | definida en ejabberd |

***

### 5.3 Configuración avanzada

En Advanced:

Server:

```
172.20.10.4
```

Port:

```
5222
```

Encryption:

```
Require encryption
```

Motivo de usar IP manual:

El sistema resolvía el dominio por IPv6 link-local (fe80::), causando errores TLS.

***

## 6. Problemas reales encontrados

***

### Problema 1 – ACME con dominio .local

Let’s Encrypt no emite certificados para .local.

Solución:

* Usar certificado autofirmado.

***

### Problema 2 – Error “certificate does not match expected identity”

Causa:\
El certificado no tenía SAN.

Solución:\
Regenerar con SAN correcto.

***

### Problema 3 – Resolución IPv6 incorrecta

`getent hosts aoki.lala.local` devolvía fe80::.

Solución:

Editar `/etc/hosts` en Debian:

```
172.20.10.4 aoki.lala.local
```

***

### Problema 4 – Error upload:

“The signing certificate authority is not known”

Causa:\
Certificado autofirmado no confiable.

***

## 7. Solución definitiva del problema de certificado

### Paso 1 – Extraer certificado público

Servidor:

```
sudo openssl x509 -in /opt/ejabberd/conf/server.pem \
-out /tmp/ejabberd-server.crt

sudo chmod 644 /tmp/ejabberd-server.crt
```

***

### Paso 2 – Copiar al cliente

```
scp /tmp/ejabberd-server.crt amelia@172.20.10.6:/home/amelia/
```

***

### Paso 3 – Instalar como CA

Cliente:

```
sudo cp ~/ejabberd-server.crt \
/usr/local/share/ca-certificates/ejabberd.crt

sudo update-ca-certificates
```

***

Resultado:

Upload funcionando correctamente.

***

## 8. Checklist de verificación

#### Si no conecta:

* Verificar puerto 5222 abierto
* Verificar resolución con `getent hosts`
* Verificar SAN del certificado

#### Si upload da 404:

* Verificar que `/upload` esté en request\_handlers
* Verificar que mod\_http\_upload esté en modules
* Verificar puerto correcto (5443)

#### Si upload da error de certificado:

* Verificar que el cliente confíe en la CA
* Verificar instalación en `/usr/local/share/ca-certificates/`
* Ejecutar `update-ca-certificates`

#### Si dice “certificate does not match”:

* Verificar SAN con openssl
* Verificar que el dominio del certificado coincida exactamente con el dominio XMPP

***

## 9. Conclusión técnica

Este laboratorio demostró:

* La importancia del SAN en certificados modernos.
* Cómo IPv6 puede interferir con validación TLS.
* Por qué XEP-0363 exige HTTPS.
* Cómo instalar manualmente una autoridad certificadora en Linux.
* La diferencia entre certificado público y clave privada.
* La relación estricta entre dominio XMPP y certificado TLS.

En producción:

* Se usaría dominio público.
* Se usaría Let’s Encrypt.
* No sería necesario instalar certificados manualmente en clientes.

En laboratorio:

* La instalación manual de la CA es parte del proceso de validación.
