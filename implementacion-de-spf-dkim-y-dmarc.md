---
description: Postfix + Bind9 + OpenDKIM
---

# IMPLEMENTACIÓN DE SPF, DKIM Y DMARC

## 1. Configuración de SPF

### 1.1 Inserción del registro SPF

Se modificó el archivo de zona DNS:

```
/var/cache/bind/db.lala.local
```

Se añadió el siguiente registro:

```
@ IN TXT "v=spf1 ip4:172.20.10.3 -all"
```

### 1.2 ¿Qué es SPF y qué se está configurando aquí?

SPF es un mecanismo de autenticación que permite a un dominio declarar qué servidores están autorizados a enviar correo en su nombre.\
Su objetivo principal es evitar suplantación de identidad basada en IP (spoofing).

Cuando un servidor recibe un correo, consulta el DNS del dominio del remitente y verifica si la IP emisora está autorizada.

En este caso:

* `v=spf1` identifica el registro como SPF.
* `ip4:172.20.10.3` autoriza únicamente esa IP.
* `-all` indica fallo explícito para cualquier IP no incluida.

Con esta política, cualquier servidor distinto de 172.20.10.3 que intente enviar correo como `lala.local` no estará autorizado.

***

### 1.3 Validación de la zona DNS

```
sudo named-checkzone lala.local /var/cache/bind/db.lala.local
```

Se verifica que el archivo de zona no contiene errores de sintaxis.

***

### 1.4 Aplicación de cambios

```
sudo systemctl restart bind9
```

***

### 1.5 Comprobación del registro SPF

```
dig @localhost -t TXT lala.local
```

Se confirma que el registro SPF está publicado correctamente.

### Simular envío desde IP NO autorizada

Desde una máquina distinta a `172.20.10.3`, ejecutar:

```
nc -v 172.20.10.3 25
```

Cuando aparezca el banner SMTP, escribir:

```
HELO attacker.com
MAIL FROM:<prueba@lala.local>
RCPT TO:<usuario@lala.local>
DATA
Subject: Prueba SPF

Mensaje de prueba
.
QUIT
```

### Revisar logs después de la prueba

En el servidor:

```
sudo tail -n 50 /var/log/mail.log
```

Se debe buscar:

* La IP desde la que se conectó.
* Si hubo `reject`.
* Si hubo advertencias relacionadas con SPF.

Esto confirma que el servidor procesó la prueba correctamente.



## Configuración de DKIM

### 2.1 Instalación del servicio de firma

```
sudo apt update
sudo apt install -y opendkim opendkim-tools
```

Se instala OpenDKIM, que actuará como servicio de firma.

***

### 2.2 ¿Qué es DKIM y qué se va a configurar?

DKIM (DomainKeys Identified Mail) es un mecanismo de firma criptográfica para correo saliente.

El servidor:

* Firma el mensaje con una clave privada.
* Inserta la firma en la cabecera (`DKIM-Signature`).
* Publica la clave pública en DNS para que pueda validarse.

Esto garantiza integridad del mensaje y autenticidad del dominio firmante.

***

### 2.3 Verificación del usuario del servicio

```
id opendkim
getent group opendkim
```

Se confirma la existencia del usuario que ejecuta el servicio, necesario para la gestión de permisos.

***

### 2.4 Creación del directorio de claves

```
sudo mkdir -p /etc/opendkim/keys/lala.local
sudo chown -R opendkim:opendkim /etc/opendkim/keys
sudo chmod 750 /etc/opendkim/keys /etc/opendkim/keys/lala.local
```

Se prepara el entorno donde se almacenará la clave privada.

***

### 2.5 Generación de la clave DKIM

```
cd /etc/opendkim/keys/lala.local
sudo opendkim-genkey -s default -d lala.local
```

Se generan:

* Clave privada (`default.private`)
* Registro TXT con clave pública (`default.txt`)

***

### 2.6 Ajuste de permisos

```
sudo chown opendkim:opendkim default.private
sudo chmod 600 default.private
```

Se restringe el acceso a la clave privada.

***

### 2.7 Configuración de OpenDKIM

Archivo:

```
/etc/opendkim.conf
```

Se añadieron las siguientes líneas:

```
Socket local:/run/opendkim/opendkim.sock
KeyTable /etc/opendkim/KeyTable
SigningTable /etc/opendkim/SigningTable
TrustedHosts /etc/opendkim/TrustedHosts
```

***

### 2.8 Qué es un milter?

Un **milter** (mail filter) es un servicio externo conectado a Postfix que puede inspeccionar o modificar mensajes durante el flujo SMTP. En esta configuración, **OpenDKIM actúa como milter**, recibiendo el correo por un socket UNIX y añadiendo la cabecera de firma DKIM.

OpenDKIM actúa como milter:

* Postfix recibe el correo.
* Antes de enviarlo, lo pasa al milter.
* El milter añade la firma DKIM.

El socket `/run/opendkim/opendkim.sock` permite la comunicación entre Postfix y OpenDKIM.

### 2.7 KeyTable / SigningTable / TrustedHosts

**/etc/opendkim/KeyTable**

```
default._domainkey.lala.local lala.local:default:/etc/opendkim/keys/lala.local/default.private
```

**/etc/opendkim/SigningTable**

```
*@lala.local default._domainkey.lala.local
```

**/etc/opendkim/TrustedHosts**

```
127.0.0.1
localhost
```

***

### 2.8 Reinicio y verificación del socket (paso clave)

```
sudo systemctl restart opendkim
sudo systemctl status opendkim --no-pager
sudo ls -la /run/opendkim/opendkim.sock
```

### 2.9 Integración con Postfix

Archivo:

```
/etc/postfix/main.cf
```

Configuración añadida:

```
smtpd_milters = unix:/run/opendkim/opendkim.sock
non_smtpd_milters = $smtpd_milters
milter_protocol = 6
milter_default_action = accept
```

Con esto, Postfix delega la firma del mensaje al milter DKIM.

Reinicio:

```
sudo systemctl restart postfix
```

### PRUEBAS DKIM

#### &#x20;Ver publicación de la clave pública en DNS

```
dig @localhost -t TXT default._domainkey.lala.local
```

Se aparece el TXT con la clave pública.

<figure><img src=".gitbook/assets/Screenshot 2026-02-23 155727.png" alt="" width="375"><figcaption></figcaption></figure>

#### &#x20;Envío de correo y ver la firma en headers

1. Se envía un correo desde una cuenta `@lala.local`.
2. Se abre el correo recibido.
3. Se visualiza el código fuente (View Source).

**Qué buscar (evidencia):**

* Cabecera `DKIM-Signature:` presente.

<figure><img src=".gitbook/assets/Screenshot 2026-02-23 160155.png" alt=""><figcaption></figcaption></figure>

#### Logs del servicio OpenDKIM

```
sudo journalctl -u opendkim -n 80 --no-pager
```

**Qué buscar:**

* arranque correcto
* eventos de firma
* errores de permisos / socket

<figure><img src=".gitbook/assets/Screenshot 2026-02-23 160343.png" alt=""><figcaption></figcaption></figure>

## Configuración de DMARC

### ¿Qué es DMARC?

DMARC (Domain-based Message Authentication, Reporting & Conformance) es un mecanismo de autenticación que trabaja sobre SPF y DKIM.

Su función principal es indicar qué debe hacerse cuando un correo falla las validaciones de SPF o DKIM, y además exige que el dominio visible en el campo `From:` esté alineado con el dominio autenticado.

En otras palabras, DMARC no firma ni autoriza IPs directamente; actúa como una política que decide cómo tratar los resultados de SPF y DKIM.



### Configuración aplicada

En el archivo de zona:

```
_dmarc IN TXT "v=DMARC1; p=none; adkim=r; aspf=r"
```

`_dmarc` es el subdominio obligatorio para la política.\
`v=DMARC1` indica la versión del protocolo.\
`p=none` establece política de monitorización.\
`adkim=r` define alineación relajada para DKIM.\
`aspf=r` define alineación relajada para SPF.

***

### Validación

```
dig @localhost -t TXT _dmarc.lala.local
```

<figure><img src=".gitbook/assets/Screenshot 2026-02-23 162950.png" alt="" width="375"><figcaption></figcaption></figure>

## Errores y solución

#### Usuario `opendkim` inexistente

**Problema:** El servicio no iniciaba correctamente.\
**Solución:** Verificar con `id opendkim` y reinstalar el paquete si era necesario.

***

#### Permisos incorrectos en la clave privada

**Problema:** No aparecía `DKIM-Signature` en los correos.\
**Solución:** Ajustar permisos:

```
chown opendkim:opendkim default.private
chmod 600 default.private
```

***

#### Socket no encontrado

**Problema:** Postfix no podía comunicarse con OpenDKIM.\
**Solución:** Verificar existencia del socket y reiniciar servicios:

```
systemctl restart opendkim
systemctl restart postfix
```

***

#### Registro DNS no visible

**Problema:** SPF o DMARC no aparecían al hacer `dig`.\
**Solución:** Validar zona y reiniciar Bind9:

```
named-checkzone lala.local /var/cache/bind/db.lala.local
systemctl restart bind9
```

#### KIM no firmaba aunque el servicio estaba activo

**Problema:** OpenDKIM estaba “running” pero los correos no tenían `DKIM-Signature`.\
**Causa:** Faltaba configurar `smtpd_milters` o no se reinició Postfix.\
**Solución:** Verificar con:

```
postconf -n | grep milter
```

Y reiniciar Postfix.

***

#### Ruta incorrecta en KeyTable

**Problema:** Error al iniciar OpenDKIM o fallo de firma.\
**Causa:** Ruta mal escrita en `KeyTable`.\
**Solución:** Corregir la ruta de `default.private` y reiniciar el servicio.

***

#### Error de sintaxis en el archivo de zona

**Problema:** SPF o DMARC no aparecían tras editarlos.\
**Causa:** Error de formato en el archivo DNS.\
**Solución:** Validar siempre con:

```
named-checkzone lala.local /var/cache/bind/db.lala.local
```
