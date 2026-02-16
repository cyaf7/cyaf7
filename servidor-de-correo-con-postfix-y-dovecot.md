---
description: >-
  Este trabajo presenta la configuración de un servidor de correo electrónico en
  Linux, integrando servicios de envío, recepción y acceso al correo. Se pone
  especial énfasis en la correcta comunicación
---

# Servidor de correo con Postfix y Dovecot

En este proyecto se implementa un servidor de correo electrónico completo utilizando Postfix como servidor SMTP, Dovecot para el acceso a los buzones mediante IMAP y un servidor DNS local para la resolución de nombres. Además, se integran clientes de correo como Roundcube y Thunderbird para verificar el funcionamiento del sistema desde distintas máquinas.

Durante el desarrollo se analizan configuraciones funcionales y de seguridad, incluyendo el uso de TLS y autenticación SMTP. El trabajo permite comprender el papel de cada servicio, detectar errores comunes de configuración y validar el correcto funcionamiento del correo electrónico tanto a nivel local como desde clientes externos.



| Servicio        | Protocolo | Puerto | Seguridad          | Uso en el proyecto                                   |
| --------------- | --------- | ------ | ------------------ | ---------------------------------------------------- |
| SMTP            | SMTP      | 25     | Ninguna            | Envío local y pruebas en texto plano                 |
| SMTP Submission | SMTP      | 587    | STARTTLS           | Envío seguro desde clientes (Thunderbird, Roundcube) |
| SMTP SSL        | SMTPS     | 465    | SSL/TLS            | Puerto seguro alternativo (no utilizado)             |
| IMAP            | IMAP      | 143    | Ninguna / STARTTLS | Acceso al correo en texto plano o con STARTTLS       |
| IMAPS           | IMAP      | 993    | SSL/TLS            | Acceso seguro al buzón                               |
| POP3            | POP3      | 110    | Ninguna            | Servicio disponible pero no utilizado                |
| POP3S           | POP3      | 995    | SSL/TLS            | Servicio disponible pero no utilizado                |
| HTTP            | HTTP      | 80     | Ninguna            | Acceso a Roundcube vía navegador                     |

Capturas y explicaciones

<figure><img src=".gitbook/assets/Screenshot 2026-02-15 143004.png" alt=""><figcaption></figcaption></figure>

En este paso se establece el nombre completo del servidor, y es fundamental pues Postfix se identifica usando el hostname.



<figure><img src=".gitbook/assets/Screenshot 2026-02-15 143220.png" alt=""><figcaption></figcaption></figure>

El archivo /etc/mailname indica **qué dominio usarán las direcciones de correo saliente**. Con eso&#x20;

* Los correos se envían como user@lala.local
* Postfix no inventa dominios incorrectos



<figure><img src=".gitbook/assets/Screenshot 2026-02-15 143835.png" alt=""><figcaption></figcaption></figure>

Aquí se vincula el nombre del servidor con la IP local.&#x20;

Esto permite que:

* El sistema se resuelva a sí mismo sin depender del DNS
* Postfix y Dovecot no fallen al resolver aoki.lala.local



<div><figure><img src=".gitbook/assets/Screenshot 2026-02-15 144153 (1).png" alt=""><figcaption></figcaption></figure> <figure><img src=".gitbook/assets/Screenshot 2026-02-15 144221 (1).png" alt=""><figcaption></figcaption></figure></div>

Instalación de Postfix – Internet Site  y Dominio del sistema de correo

se instala con&#x20;





<figure><img src=".gitbook/assets/Screenshot 2026-02-15 144516.png" alt=""><figcaption></figcaption></figure>

Verificación del estado de Postfix



<figure><img src=".gitbook/assets/Screenshot 2026-02-15 144942.png" alt=""><figcaption></figcaption></figure>

Esto indica que:

* Cada usuario tendrá su correo en `~/Maildir`
* Se usa **Maildir** en lugar de **mbox**
* Cada mensaje es un archivo -> más seguro y modern



<figure><img src=".gitbook/assets/Screenshot 2026-02-15 145028.png" alt=""><figcaption></figcaption></figure>

Cada vez que se modifica `main.cf`:

* Es obligatorio reiniciar Postfix
* Así se aplican las nuevas directivas





<figure><img src=".gitbook/assets/Screenshot 2026-02-15 145223.png" alt="" width="305"><figcaption></figcaption></figure>

Cada usuario del sistema:

* Representa **un buzón de correo**
* Tendrá dirección `usuario@lala.local`
* Su contraseña será la usada para IMAP/SMTP AUTH



**Instalación de Mailutils**

Con:

{% hint style="info" %}
sudo apt install mailutils
{% endhint %}

Mailutils es:

* Un **cliente de correo en terminal (MUA)**
* Útil para pruebas locales
* Permite verificar que Postfix entrega correos sin clientes gráficos



<figure><img src=".gitbook/assets/Screenshot 2026-02-15 145441.png" alt=""><figcaption></figcaption></figure>

Este comando hace lo siguiente:

* `echo "holis hola"` → genera el cuerpo del mensaje
* `mail -s "teste postfix"` → usa Mailutils para enviar un correo con asunto
* `amelia@lala.local` → destinatario local del sistema

&#x20;Aquí **Postfix ya está funcionando** como MTA:

* acepta el correo
* lo entrega al buzón del usuario



<figure><img src=".gitbook/assets/Screenshot 2026-02-15 145849.png" alt="" width="464"><figcaption></figcaption></figure>

"Cannot open mailbox /var/mail/amelia: No such file or directory\
No mail for amelia"

El comando `mail` por defecto busca correo en **formato mbox**

Pero luego vemos que **el correo SÍ llegó**.

Cada archivo dentro de `Maildir/new` representa:

* un correo individual
* entregado correctamente por Postfix



<figure><img src=".gitbook/assets/Screenshot 2026-02-15 150216.png" alt=""><figcaption></figcaption></figure>

Con esto se instala:

* **Dovecot** como MDA (Mail Delivery Agent)
* Servicios para:
  * POP3
  * IMAP

&#x20;Dovecot será el encargado de:

* exponer los correos almacenados en Maildir
* permitir acceso desde clientes como Thunderbird o Roundcube

<div><figure><img src=".gitbook/assets/Screenshot 2026-02-15 150741.png" alt=""><figcaption></figcaption></figure> <figure><img src=".gitbook/assets/Screenshot 2026-02-15 150427.png" alt=""><figcaption></figcaption></figure></div>

Configuración del formato de buzón en Dovecot en el archivo: `/etc/dovecot/conf.d/10-mail.conf`

Aquí se le dice a Dovecot:

* dónde están los correos
* que use **Maildir**
* que coincida con la configuración de Postfix

Despues hacemos un **sudo systemctl restart dovecot.**



En:

<figure><img src=".gitbook/assets/Screenshot 2026-02-15 151029.png" alt=""><figcaption></figcaption></figure>

<figure><img src=".gitbook/assets/Screenshot 2026-02-15 151120 (2).png" alt=""><figcaption></figcaption></figure>

Lo cambiamos para **disable\_plaintext\_auth = no,** asi se permite autenticación en texto plano cuando se usa TLS. Y luego se reinicia Dovecot para asegurar que la configuración está cargada correctamente con **sudo systemctl restart dovecot.**



<figure><img src=".gitbook/assets/Screenshot 2026-02-15 151345.png" alt=""><figcaption></figcaption></figure>

Comprueba que Dovecot responde en el puerto IMAP y que el servicio está activo. La respuesta OK indica que el servidor IMAP acepta conexiones.



<figure><img src=".gitbook/assets/Screenshot 2026-02-15 151630.png" alt=""><figcaption></figcaption></figure>

Aqui se instala el servicio DNS que permitirá resolver el dominio del servidor de correo. Y se confirma que está activo con **sudo systemctl status bind9**.



<figure><img src=".gitbook/assets/Screenshot 2026-02-15 152146.png" alt=""><figcaption></figcaption></figure>

Abrimos el **/etc/bind/named.conf.local** que es el archivo donde se declaran las zonas DNS gestionadas por el servidor, y se añade la zona lala.local apuntando al archivo de zona correspondiente.



<figure><img src=".gitbook/assets/Screenshot 2026-02-15 152716.png" alt=""><figcaption></figcaption></figure>

Creación del archivo de zona: El archivo donde se definen los registros DNS del dominio.

Aquí se define:

* el **SOA** (autoridad de la zona)
* el **servidor DNS**
* el registro **A** del servidor de correo

Esto deja la base correcta para añadir servicios (MX, SMTP, IMAP…).

Y luego un **sudo systemctl restart bind9,** para la recarga la configuración DNS y para aplicar los cambios.



<figure><img src=".gitbook/assets/Screenshot 2026-02-15 152911.png" alt=""><figcaption></figcaption></figure>

Aqui he visto mi error, y hecho un **sudo cp /etc/bind/db.lalal.local /var/cache/bind/db.lala.local** pues habia creado el fichero el la carpeta errada.&#x20;



<figure><img src=".gitbook/assets/Screenshot 2026-02-15 153113.png" alt=""><figcaption></figcaption></figure>

Con **sudo systemctl restart bind9 se r**ecarga la configuración DNS para aplicar los cambios nuevamente. Y luego comprueba que el archivo de zona no contiene errores de sintaxis.&#x20;



<figure><img src=".gitbook/assets/Screenshot 2026-02-15 153737.png" alt=""><figcaption></figcaption></figure>

El dig comprueba que el nombre del servidor se resuelve a la dirección IP correcta.

Esto confirma que:

* BIND responde correctamente
* el servidor DNS local funciona
* el nombre se resuelve a la IP correcta



<figure><img src=".gitbook/assets/Screenshot 2026-02-15 154311.png" alt="" width="375"><figcaption></figcaption></figure>

La idea aquí es clara:

* el dominio `lala.local` envía correo a `email.lala.local`
* todos los servicios apuntan al mismo servidor físico



<figure><img src=".gitbook/assets/Screenshot 2026-02-15 154724.png" alt=""><figcaption></figcaption></figure>

Pero en DNS:

* **un registro MX NO puede apuntar a un CNAME**
* el MX **debe apuntar a un A o AAAA**

**Y con el cambio arriba :**

* `email.lala.local` tiene una IP real
* el MX apunta a un nombre válido
* la zona cumple el estándar DNS



<figure><img src=".gitbook/assets/Screenshot 2026-02-15 154802.png" alt=""><figcaption></figcaption></figure>

Se comproba: que la zona es válida, el DNS está bien construido, yel correo puede depender del DNS sin problemas.



<figure><img src=".gitbook/assets/Screenshot 2026-02-15 160205.png" alt=""><figcaption></figcaption></figure>

<figure><img src=".gitbook/assets/Screenshot 2026-02-15 160249.png" alt="" width="375"><figcaption></figcaption></figure>

MySQL se instaló para: almacenar la base de datos de Roundcube, y para gestionar usuarios, sesiones y preferencias del webmail.



<figure><img src=".gitbook/assets/Screenshot 2026-02-15 160449.png" alt=""><figcaption></figcaption></figure>

Durante esta instalación: Se instala Roundcube (webmail); Se instala el conector MySQL; Se instalan dependencias como Apache y PHP.

Roundcube será el **cliente web (MUA)** que:

* se conecta por IMAP a Dovecot
* envía correos vía SMTP a Postfix



<figure><img src=".gitbook/assets/Screenshot 2026-02-15 161850.png" alt=""><figcaption></figcaption></figure>

Significa: que Apache está funcionando, pero **todavía no hay un alias configurado** para /roundcube. Es el comportamiento normal antes de configurar Apache.



<figure><img src=".gitbook/assets/Screenshot 2026-02-15 162203.png" alt=""><figcaption></figcaption></figure>

Esto le dice a Apache que `/roundcube` apunta al directorio real de Roundcube, y que se permita el acceso desde el navegador.

Aquí se:

* copia el sitio por defecto
* adapta para servir Roundcube
* mantiene una configuración clara y separada



<figure><img src=".gitbook/assets/Screenshot 2026-02-15 162420.png" alt=""><figcaption></figcaption></figure>

Esto indica que:

* el sitio ya estaba activo
* Apache reconoce la configuración

Luego: **sudo systemctl reload apache2**



<figure><img src=".gitbook/assets/Screenshot 2026-02-15 162616.png" alt="" width="359"><figcaption></figcaption></figure>

Este comando sigue redirecciones (-L) y muestra solo cabeceras (-I), el resultado indica 301 Moved Permanently (Apache redirige a /roundcube/), 200 OK (Roundcube responde), se crea la cookie roundcube\_sessid y Apache junto con PHP funcionan correctamente.



<figure><img src=".gitbook/assets/Screenshot 2026-02-15 163850.png" alt=""><figcaption></figcaption></figure>

#### Configuración de resolución de nombres en el cliente

En la máquina cliente (/etc/hosts): Este paso permite que el cliente resuelva el servidor sin depender del DNS y que las pruebas se centren en el correo y no en la resolución de nombres.



<figure><img src=".gitbook/assets/Screenshot 2026-02-15 164034.png" alt=""><figcaption></figcaption></figure>

Resultado: paquetes enviados y recibidos, latencias bajas y 0 % de pérdida, esto confirma conectividad de red, nombre bien resuelto y comunicación cliente/servidor correcta.

<div><figure><img src=".gitbook/assets/Screenshot 2026-02-15 164145 (1).png" alt=""><figcaption></figcaption></figure> <figure><img src=".gitbook/assets/Screenshot 2026-02-15 164126.png" alt=""><figcaption></figcaption></figure></div>

Se muestra la pantalla de login de Roundcube desde otra máquina, lo que confirma acceso por red, y tras autenticarse la usuaria aparece la bandeja de entrada con el mensaje “teste postfix”, cuyo contenido coincide con el enviado desde la terminal.

<figure><img src=".gitbook/assets/Screenshot 2026-02-15 164457.png" alt=""><figcaption></figcaption></figure>

Postfix escucha en 0.0.0.0:25 y :::25, lo que indica que el servidor acepta conexiones SMTP y está disponible tanto en IPv4 como en IPv6.

<figure><img src=".gitbook/assets/Screenshot 2026-02-15 164649.png" alt="" width="375"><figcaption></figcaption></figure>

Esta configuración hace que Roundcube se conecte a Dovecot por IMAP, envíe correos a Postfix de forma local y utilice las credenciales del usuario autenticado. Archivo: `/etc/roundcube/config.inc.php.` y hice los seguintes cambios:

<figure><img src=".gitbook/assets/Screenshot 2026-02-15 165930.png" alt=""><figcaption></figcaption></figure>

Esta configuración indica que Roundcube no se autentica contra SMTP, entrega el correo a Postfix local y este lo acepta por ser un envío interno, válido en entornos locales o de laboratorio.

<figure><img src=".gitbook/assets/Screenshot 2026-02-15 170958 (2).png" alt=""><figcaption></figcaption></figure>

<figure><img src=".gitbook/assets/Screenshot 2026-02-15 171854 (1).png" alt="" width="375"><figcaption></figcaption></figure>

Este comando captura tráfico SMTP en la interfaz loopback filtrando el puerto 25, en la salida se observan comandos como EHLO, MAIL FROM, RCPT TO y DATA en texto plano, lo que demuestra que SMTP en el puerto 25 no cifra el contenido y que cualquiera con acceso al tráfico podría leer el correo, justificando que no es seguro usar solo el puerto 25.



AUTH + 587

<figure><img src=".gitbook/assets/Screenshot 2026-02-15 172111.png" alt=""><figcaption></figcaption></figure>

Se revisa la configuración TLS de Dovecot y se observa que existen certificados configurados, que se utilizan claves almacenadas en /etc/ssl/private y que TLS está disponible para los servicios IMAP y POP3, lo que prepara el sistema para usar IMAPS en el puerto 993 y STARTTLS sobre IMAP.

<figure><img src=".gitbook/assets/Screenshot 2026-02-15 172618.png" alt=""><figcaption></figcaption></figure>

<figure><img src=".gitbook/assets/Screenshot 2026-02-15 172827.png" alt=""><figcaption></figcaption></figure>

La configuración con smtpd\_tls\_cert\_file, smtpd\_tls\_key\_file y smtpd\_tls\_security\_level=may indica que Postfix ofrece TLS, permite al cliente usar STARTTLS y es compatible con clientes modernos como Thunderbird o Roundcube.

<figure><img src=".gitbook/assets/Screenshot 2026-02-15 172958.png" alt=""><figcaption></figcaption></figure>

<figure><img src=".gitbook/assets/Screenshot 2026-02-15 173222.png" alt=""><figcaption></figcaption></figure>

<figure><img src=".gitbook/assets/Screenshot 2026-02-15 173435.png" alt="" width="360"><figcaption></figcaption></figure>

<figure><img src=".gitbook/assets/Screenshot 2026-02-15 173548.png" alt="" width="263"><figcaption></figcaption></figure>

<figure><img src=".gitbook/assets/Screenshot 2026-02-15 173613.png" alt="" width="256"><figcaption></figcaption></figure>

<figure><img src=".gitbook/assets/Screenshot 2026-02-15 173920.png" alt=""><figcaption></figcaption></figure>

<figure><img src=".gitbook/assets/Screenshot 2026-02-15 174521.png" alt=""><figcaption></figcaption></figure>

<figure><img src=".gitbook/assets/Screenshot 2026-02-15 174721 (2).png" alt="" width="328"><figcaption></figcaption></figure>

<figure><img src=".gitbook/assets/Screenshot 2026-02-15 175053.png" alt="" width="332"><figcaption></figcaption></figure>

<figure><img src=".gitbook/assets/Screenshot 2026-02-15 175114.png" alt=""><figcaption></figcaption></figure>

<div><figure><img src=".gitbook/assets/Screenshot 2026-02-15 175442.png" alt=""><figcaption></figcaption></figure> <figure><img src=".gitbook/assets/Screenshot 2026-02-15 175433 (1).png" alt=""><figcaption></figcaption></figure></div>

<figure><img src=".gitbook/assets/Screenshot 2026-02-15 175705.png" alt=""><figcaption></figcaption></figure>



outra maquina, aq empieza con 587 + auth

<figure><img src=".gitbook/assets/Screenshot 2026-02-15 180458.png" alt=""><figcaption></figcaption></figure>
