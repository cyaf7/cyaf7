---
description: >-
  En este laboratorio se implementa un servidor de correo local completo
  utilizando Postfix, Dovecot y Bind9. Posteriormente, se integra una capa de
  seguridad mediante TLS y autenticación SMTP.
---

# Mail server

El objetivo de este proyecto es diseñar e implementar una infraestructura de correo electrónico funcional en un entorno local controlado. Para ello, se configuran los componentes esenciales de un sistema de mensajería: un servidor SMTP (Postfix) encargado del envío y entrega de correos, un servidor IMAP (Dovecot) responsable del acceso a los buzones, y un servidor DNS (Bind9) que permite la resolución correcta del dominio.

El dominio utilizado es **lala.local**, y el servidor principal es **aoki.lala.local**, con dirección IP **172.20.10.3**.

En esta primera parte se construye la base funcional del sistema: instalación de Postfix, configuración de Maildir, creación de usuarios y verificación del envío local.



## 1. Configuración de identidad del servidor

***

### 1.1 Modificación de `/etc/hosts`

Editar:

```
sudo nano /etc/hosts
```

Configurar:

```
127.0.0.1   localhost
127.0.1.1   aoki.lala.local aoki
```

#### Explicación breve

* `127.0.0.1 localhost` → dirección loopback estándar.
* `127.0.1.1 aoki.lala.local aoki`\
  Asocia el hostname del sistema al entorno local.
* `aoki.lala.local` → nombre completo (FQDN).
* `aoki` → alias corto.

Esto permite que el servidor se identifique correctamente a sí mismo.

<figure><img src=".gitbook/assets/Screenshot 2026-02-20 140708.png" alt=""><figcaption></figcaption></figure>



***

### 1.2 Verificación del hostname

```
hostname
```

```
cat /etc/mailname
```

Explicación:

* `hostname` muestra el nombre actual del sistema.
* `/etc/mailname` define el dominio utilizado por herramientas de correo en Debian/Ubuntu.

<figure><img src=".gitbook/assets/Screenshot 2026-02-20 140758.png" alt=""><figcaption></figcaption></figure>

## 2. Configuración de Maildir en Postfix

Postfix, por defecto, puede utilizar formato _mbox_. En este laboratorio se configuró el formato **Maildir**, que almacena cada mensaje como un archivo independiente.

Editar:

```
sudo nano /etc/postfix/main.cf
```

Añadir:

```
home_mailbox = Maildir/
mailbox_command =
```

<figure><img src=".gitbook/assets/Screenshot 2026-02-18 141204.png" alt=""><figcaption></figcaption></figure>



* `home_mailbox = Maildir/`\
  Indica que el buzón del usuario estará en `~/Maildir/`.
* `mailbox_command =`\
  Se deja vacío para evitar el uso de comandos externos de entrega.

Reiniciar el servicio:

```
sudo systemctl restart postfix
```

Para aplicar los cambios realizados en el archivo de configuración.

## 3. Creación de usuarios del sistema

Se crean los usuarios que actuarán como cuentas de correo.

```
sudo adduser ameliana
```

* `adduser` crea un usuario del sistema.
* Se genera automáticamente su directorio `/home/ameliana`.
* Se asigna UID y grupo correspondiente.

<figure><img src=".gitbook/assets/Screenshot 2026-02-18 141334.png" alt="" width="308"><figcaption></figcaption></figure>

\
4\. Envío del primer correo local (prueba Postfix)
--------------------------------------------------

Para comprobar que Postfix entrega correctamente en Maildir:

```
echo "first message" | mail -s "1" ameliana@lala.local
```

* `echo "first message"` → contenido del correo.
* `|` → redirige la salida hacia el comando `mail`.
* `mail -s "1"` → define el asunto.
* `ameliana@lala.local` → destinatario local.

<figure><img src=".gitbook/assets/Screenshot 2026-02-18 142339.png" alt=""><figcaption></figcaption></figure>

\
5\. Verificación del buzón Maildir
----------------------------------

Comprobar que el mensaje fue almacenado:

```
ls -la /home/ameliana/Maildir/new
```

Explicación:

* `new/` contiene mensajes recién entregados.
* Si aparece un archivo, el sistema funciona correctamente.

<figure><img src=".gitbook/assets/Screenshot 2026-02-18 142403.png" alt=""><figcaption></figcaption></figure>

## 6. Inspección del contenido del correo

```
cat /home/ameliana/Maildir/new/<archivo_del_mensaje>
```

* Permite visualizar cabeceras SMTP.
* Se confirma que Postfix procesó correctamente el mensaje.
* Se observa el `Message-ID`, `From`, `To`, y fecha.

<figure><img src=".gitbook/assets/Screenshot 2026-02-18 142413.png" alt=""><figcaption></figcaption></figure>

## Nota importante sobre el comando "mail"

Al ejecutar:

```
mail
```

Puede aparecer:

```
Cannot open mailbox /var/mail/ameliana
```

<figure><img src=".gitbook/assets/Screenshot 2026-02-18 142350.png" alt=""><figcaption></figcaption></figure>

El mensaje “Cannot open mailbox /var/mail/ameliana” no indica un error del servidor. El comando `mail` intenta leer buzones en formato mbox ubicados en `/var/mail/`, mientras que el sistema fue configurado para utilizar el formato Maildir, donde los correos se almacenan en `~/Maildir/`. Por tanto, la verificación correcta debe realizarse en la carpeta `Maildir/new`.

## 7. Configuración de Dovecot, DNS y Roundcube

En esta fase se habilita el acceso a los buzones mediante IMAP (Dovecot), se configura la resolución del dominio con Bind9 y se integra una interfaz web (Roundcube). Con ello, el sistema deja de ser únicamente un servidor SMTP local y pasa a ser una plataforma de correo completa y accesible.

***

## 7.1 Configuración de Dovecot (IMAP)

Dovecot permitirá que los usuarios accedan a sus buzones almacenados en formato Maildir.

### 7.1 Configuración del formato Maildir

Editar:

```
sudo nano /etc/dovecot/conf.d/10-mail.conf
```

Buscar y configurar:

```
mail_location = maildir:~/Maildir
```

* `mail_location` define dónde Dovecot debe buscar los correos.
* `maildir:` indica el formato.
* `~/Maildir` apunta al directorio personal del usuario.

<figure><img src=".gitbook/assets/Screenshot 2026-02-18 143003.png" alt=""><figcaption></figcaption></figure>

### 7.2 Permitir autenticación en entorno local

Editar:

```
sudo nano /etc/dovecot/conf.d/10-auth.conf
```

Configurar:

```
disable_plaintext_auth = no
```

* Permite autenticación sin cifrado únicamente en entorno local.
* En producción debería usarse siempre con TLS obligatorio.

<figure><img src=".gitbook/assets/Screenshot 2026-02-18 143039.png" alt=""><figcaption></figcaption></figure>

### 7.3 Reinicio del servicio

```
sudo systemctl restart dovecot
```

Aplica cambios en la configuración.

### 7.4 Prueba IMAP

```
telnet localhost 143
```

Si aparece:

```
* OK Dovecot ready
```

Significa que IMAP está operativo.

<figure><img src=".gitbook/assets/Screenshot 2026-02-18 143311.png" alt=""><figcaption></figcaption></figure>

## 8. Configuración del servidor DNS (Bind9)

El DNS permitirá que el dominio `lala.local` resuelva correctamente hacia el servidor.

***

### 8.1 Declaración de zona

Editar:

```
sudo nano /etc/bind/named.conf.local
```

Añadir:

```
zone "lala.local" {
  type master;
  file "/var/cache/bind/db.lala.local";
  };
```

* `zone "lala.local"` define el dominio.
* `type master` indica que este servidor es autoritativo.
* `file` señala dónde está el archivo de registros.

<figure><img src=".gitbook/assets/Screenshot 2026-02-18 143627.png" alt=""><figcaption></figcaption></figure>

### 8.2 Archivo de zona

Editar:

```
sudo nano /var/cache/bind/db.lala.local
```

Contenido:

```
$TTL 604800
@ IN SOA aoki.lala.local. admin.lala.local. (
  1 604800 86400 2419200 604800 )

@     IN NS  aoki.lala.local.
aoki  IN A   172.20.10.3

@     IN MX 10 email.lala.local.
email IN A   172.20.10.3
smtp  IN CNAME aoki.lala.local.
imap  IN CNAME aoki.lala.local.
pop3  IN CNAME aoki.lala.local.
```

#### Explicación de los registros

* **SOA (Start of Authority):** define autoridad y parámetros de la zona.
* **NS:** servidor DNS responsable.
* **A:** asocia nombre a IP.
* **MX:** define el servidor de correo del dominio.
* **CNAME:** crea alias.

<figure><img src=".gitbook/assets/Screenshot 2026-02-18 195307.png" alt="" width="350"><figcaption></figcaption></figure>

### 8.3 Validación de zona

```
sudo named-checkzone lala.local /var/cache/bind/db.lala.local
```

Valida sintaxis y coherencia.

<figure><img src=".gitbook/assets/Screenshot 2026-02-18 195351.png" alt=""><figcaption></figcaption></figure>

Confirmando que no existen errores de sintaxis con el OK.

### 8.4 Prueba de resolución

```
dig @localhost aoki.lala.local
```

* `dig` consulta DNS.
* `@localhost` fuerza consulta al servidor local.
* Se verifica que responde con 172.20.10.3.

<figure><img src=".gitbook/assets/Screenshot 2026-02-18 195540.png" alt="" width="375"><figcaption></figcaption></figure>

## 9. Instalación y configuración de Roundcube

Roundcube proporciona una interfaz web para acceder al correo vía navegador.

***

### 9.1 Configuración Apache

Editar:

```
sudo nano /etc/apache2/sites-available/round.conf
```

Añadir:

```
Alias /roundcube /var/lib/roundcube

<Directory /var/lib/roundcube>
  Require all granted
</Directory>
```

* `Alias` publica Roundcube en `/roundcube`.
* `Require all granted` permite acceso HTTP.

<figure><img src=".gitbook/assets/Screenshot 2026-02-18 210944.png" alt="" width="280"><figcaption></figcaption></figure>

Activar sitio:

```
sudo a2ensite round.conf
sudo systemctl reload apache2
```

<figure><img src=".gitbook/assets/Screenshot 2026-02-18 211042.png" alt=""><figcaption></figcaption></figure>

### 8.2 Configuración de Roundcube

Editar:

```
sudo nano /etc/roundcube/config.inc.php
```

Configurar:

```
$config['imap_host'] = 'localhost:143';
$config['smtp_host'] = 'localhost:25';
$config['smtp_user'] = '';
$config['smtp_pass'] = '';
```

<figure><img src=".gitbook/assets/Screenshot 2026-02-18 211407.png" alt="" width="283"><figcaption></figcaption></figure>

* `imap_host` conecta con Dovecot.
* `smtp_host` conecta con Postfix.
* Usuario y contraseña vacíos en fase inicial para evitar conflictos de autenticación prematuros.

### 8.3 Acceso vía navegador

Acceder a:

```
http://aoki.lala.local/roundcube
```

<figure><img src=".gitbook/assets/Screenshot 2026-02-18 221459.png" alt="" width="350"><figcaption></figcaption></figure>

falta uma img

## 9. Integración de la FASE SEGURA

### (Puerto 587 + TLS + SMTP AUTH)

Hasta este momento, el servidor permitía envío SMTP básico mediante el puerto 25 y acceso IMAP sin cifrado obligatorio. Sin embargo, un entorno moderno de correo electrónico requiere:

* Autenticación obligatoria para el envío.
* Separación entre tráfico de servidor (25) y tráfico de cliente (587).
* Cifrado de extremo a extremo mediante TLS.
* Protección contra open relay.

En esta fase se transforma el servidor funcional en un servidor seguro.

## 9.1 Activación del puerto 587 (Submission)

Editar el archivo:

```
sudo nano /etc/postfix/master.cf
```

Agregar o descomentar:

```
submission inet n - y - - smtpd
  -o syslog_name=postfix/submission
  -o smtpd_tls_security_level=encrypt
```

#### Explicación breve

* `submission` → habilita el puerto 587.
* `smtpd_tls_security_level=encrypt` → obliga a usar TLS en este puerto.
* Se separa el envío autenticado del puerto 25 tradicional.

<figure><img src=".gitbook/assets/Screenshot 2026-02-18 211951.png" alt=""><figcaption></figcaption></figure>

Reiniciar Postfix:

```
sudo systemctl restart postfix
```

Verificar puerto abierto:

```
ss -lntp | grep 587
```

<figure><img src=".gitbook/assets/Screenshot 2026-02-18 212036.png" alt=""><figcaption></figcaption></figure>

## 10. Configuración de TLS en Postfix

Instalar certificados (entorno laboratorio):

```
sudo apt install ssl-cert
```

Editar:

```
sudo nano /etc/postfix/main.cf
```

Agregar sección TLS:

```
# TLS parameters
smtpd_tls_cert_file=/etc/ssl/certs/ssl-cert-snakeoil.pem
smtpd_tls_key_file=/etc/ssl/private/ssl-cert-snakeoil.key
smtpd_tls_security_level=may
```

* `cert_file` → certificado público del servidor.
* `key_file` → clave privada.
* `security_level=may` → TLS opcional en puerto 25.
* En puerto 587 ya es obligatorio por configuración anterior.

<figure><img src=".gitbook/assets/Screenshot 2026-02-18 212132.png" alt=""><figcaption></figcaption></figure>

einiciar:

```
sudo systemctl restart postfix
```

***

## 11. Verificación manual de TLS

Probar negociación:

```
openssl s_client -starttls smtp -connect aoki.lala.local:587
```

Si aparece:

```
Verify return code: 0 (ok)
```

Significa que el certificado fue aceptado y TLS funciona correctamente.

<figure><img src=".gitbook/assets/Screenshot 2026-02-18 212708.png" alt="" width="375"><figcaption></figcaption></figure>

## 12. Integración SMTP AUTH (Postfix ↔ Dovecot)

Ahora se habilita autenticación obligatoria.

***

### 4.1 Instalación de módulos SASL

```
sudo apt install libsasl2-modules sasl2-bin
```

<figure><img src=".gitbook/assets/Screenshot 2026-02-18 212806.png" alt=""><figcaption></figcaption></figure>

### 13. Configuración del socket de autenticación en Dovecot

Editar:

```
sudo nano /etc/dovecot/conf.d/10-master.conf
```

Dentro de `service auth`:

```
unix_listener /var/spool/postfix/private/auth {
  mode = 0666
  user = postfix
  group = dovecot
}
```

* Crea un socket UNIX.
* Postfix delega la autenticación a Dovecot.
* Se establece comunicación segura interna.

<figure><img src=".gitbook/assets/Screenshot 2026-02-18 213053.png" alt="" width="319"><figcaption></figcaption></figure>

Creación del socket de autenticación que permite a Postfix utilizar Dovecot como backend SASL.

Reiniciar Dovecot:

```
sudo systemctl restart dovecot
```

### 14. Activación de SASL en Postfix

Editar:

```
sudo nano /etc/postfix/main.cf
```

Agregar:

```
smtpd_sasl_auth_enable = yes
smtpd_sasl_type = dovecot
smtpd_sasl_path = private/auth
smtpd_sasl_security_options = noanonymous
```

<figure><img src=".gitbook/assets/Screenshot 2026-02-18 213413.png" alt=""><figcaption></figcaption></figure>

Se habilitó explícitamente la autenticación SMTP en Postfix mediante la integración con Dovecot como backend SASL. Esta configuración permite que el servidor exija credenciales válidas antes de aceptar el envío de mensajes desde clientes.

## 15. Protección contra Open Relay

Un open relay es un servidor SMTP que permite el envío de correos sin autenticación a cualquier destino. Esta configuración es altamente insegura y suele ser explotada para envío de spam. En este laboratorio se implementaron restricciones para evitar que el servidor funcione como relay abierto.

Configuración de restricciones SMTP para prevenir el uso indebido del servidor como open relay.

Añadir en `main.cf`:

```
smtpd_sasl_local_domain = $myhostname

smtpd_recipient_restrictions =
  permit_sasl_authenticated,
  reject_unauth_destination
```

* Permite envío solo a usuarios autenticados.
* Impide que el servidor sea utilizado como relay abierto.

## 16. Verificación en Cliente (Thunderbird)

### Configuración IMAP

* Servidor: aoki.lala.local
* Puerto: 993
* Seguridad: SSL/TLS

<figure><img src=".gitbook/assets/Screenshot 2026-02-18 214322.png" alt=""><figcaption></figcaption></figure>

### Configuración SMTP

* Servidor: aoki.lala.local
* Puerto: 587
* Seguridad: STARTTLS

<figure><img src=".gitbook/assets/Screenshot 2026-02-18 214351.png" alt=""><figcaption></figcaption></figure>

<figure><img src=".gitbook/assets/Screenshot 2026-02-18 214436.png" alt=""><figcaption></figcaption></figure>

## 17. Verificación de cifrado en Wireshark

En la captura se observa:

* Inicio STARTTLS
* Negociación TLSv1.3
* Cambio de Cipher
* Datos cifrados (Application Data)

<figure><img src=".gitbook/assets/Screenshot 2026-02-18 215120.png" alt=""><figcaption></figcaption></figure>
