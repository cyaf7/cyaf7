---
description: >-
  Se implementó ClamAV como sistema antivirus integrado en Postfix mediante el
  mecanismo Milter, permitiendo analizar los correos antes de su entrega. El
  objetivo es detectar y bloquear contenido malo.
icon: head-side-mask
---

# ClamAV

El objetivo de esta fase del proyecto es incorporar un sistema de protección antivirus al servidor de correo previamente implementado. Para ello, se instala y configura **ClamAV**, un motor antivirus de código abierto, junto con **ClamAV-Milter**, el componente encargado de integrarse con Postfix para analizar mensajes en tiempo real.

Con esta integración, el servidor de correo adquiere la capacidad de:

* Analizar automáticamente los mensajes entrantes y salientes.
* Detectar archivos maliciosos o patrones de malware.
* Rechazar o aceptar mensajes según la política definida.
* Evitar la propagación de software malicioso dentro del dominio.

Esta implementación añade una capa crítica de seguridad al sistema SMTP ya funcional.

### Conceptos Fundamentales del Sistema Antivirus

Antes de describir la configuración técnica, es importante comprender los componentes implicados.

## ClamAV

ClamAV es un motor antivirus diseñado para sistemas Linux y servidores de correo. Funciona mediante el uso de firmas digitales que permiten identificar patrones conocidos de malware.

Sus componentes principales son:

* **clamd** -> Demonio que realiza el análisis en segundo plano.
* **freshclam** -> Servicio encargado de actualizar las firmas de virus.
* **clamscan** -> Herramienta manual de análisis bajo demanda.

ClamAV no bloquea correos por sí solo. Necesita integrarse con el servidor SMTP.

***

### ClamAV-Milter

Un _milter_ (Mail Filter) es un mecanismo que permite a Postfix delegar el análisis de mensajes a procesos externos.

ClamAV-Milter:

1. Recibe el mensaje desde Postfix.
2. Envía el contenido al demonio clamd.
3. Recibe el resultado del análisis.
4. Devuelve una acción a Postfix (accept o reject).

Esta arquitectura permite el análisis en tiempo real antes de que el correo sea entregado al buzón.

***

### Qué es un socket?

Este es uno de los conceptos más importantes.

### &#x20;Definición simple

Un **socket** es un punto de comunicación entre procesos.

Es como un “enchufe” donde dos programas se conectan para hablar.

***

### En nuestro caso usamos un socket UNIX

Ubicación:

```
/var/run/clamav/clamav-milter.ctl
```

No es un archivo normal.

Puedes comprobarlo con:

```
file /var/run/clamav/clamav-milter.ctl
```

Te dirá algo como:

```
socket
```

***

### Por qué usar socket y no puerto TCP?

Porque:

* Es más rápido
* Es más seguro (solo local)
* No expone el servicio a la red

***

### &#x20;Qué es /run o /var/run?

Son directorios donde el sistema guarda:

* Archivos temporales
* PID de procesos
* Sockets
* Información de servicios activos

Son temporales y se recrean en cada arranque.

***

### Qué es un archivo EICAR?

EICAR es un estándar mundial de prueba antivirus.

No es un virus real.

Es una cadena de texto diseñada para que:

* Todos los antivirus la detecten
* Sin causar daño

Texto:

```
X5O!P%@AP[4\PZX54(P^)7CC)7}$EICAR-STANDARD-ANTIVIRUS-TEST-FILE!$H+H*
```

***

### &#x20;Qué significa “milter-reject”?

Cuando vimos en el log:

```
milter-reject: END-OF-MESSAGE
5.7.1 Command rejected
```

Significa:

* El mensaje fue enviado al milter
* El milter detectó algo
* Postfix rechazó el correo antes de entregarlo

No llegó al buzón.

***

### Qué es un log?

Un log es un archivo donde el sistema registra eventos.

En correo:

```
/var/log/mail.log
```

Ahí puedes ver:

* Conexiones SMTP
* Autenticaciones
* Errores
* Rechazos
* Intervención del antivirus

***

### Qué es chroot?&#x20;

Chroot es un mecanismo de aislamiento.

Hace que un servicio vea solo una parte del sistema de archivos,\
como si fuera su “mini sistema”.

Por eso tuvimos problemas con el socket inicialmente.

***

### Qué son los Milters?

**Milters (Mail Filters)** son procesos externos que se integran con el servidor de correo (Postfix) para analizar o modificar un mensaje durante la sesión SMTP, antes de que sea aceptado definitivamente.

En la práctica:

Postfix recibe el correo -> lo envía al milter -> el milter lo analiza (antivirus, firma DKIM, antispam, etc.) -> devuelve una decisión (aceptar, rechazar o modificar) -> Postfix actúa según esa respuesta.

Sirven para añadir capas de seguridad y control en tiempo real sobre los correos entrantes y salientes.

## &#x20;Resumen conceptual completo

| Elemento | Qué es         | Qué hace                    |
| -------- | -------------- | --------------------------- |
| Postfix  | MTA            | Recibe y entrega correo     |
| ClamAV   | Antivirus      | Detecta malware             |
| clamd    | Servicio       | Analiza archivos            |
| milter   | Filtro SMTP    | Conecta Postfix y ClamAV    |
| Socket   | Canal local    | Comunicación entre procesos |
| Maildir  | Formato buzón  | Guarda correos              |
| EICAR    | Archivo prueba | Simula virus                |
| Log      | Registro       | Guarda eventos              |

## IMPLEMENTACIÓN DE CLAMAV EN POSTFIX&#x20;



## 1️. Objetivo

Integrar un sistema antivirus en el flujo SMTP del servidor para:

* Analizar correos entrantes
* Detectar malware
* Rechazar mensajes infectados
* Registrar eventos en logs

***

## 2️. Instalación de ClamAV

### 2.1 Instalar paquetes

```
sudo apt update
sudo apt install -y clamav clamav-daemon clamav-freshclam clamav-milter
```

#### Qué hace cada paquete

| Paquete          | Función                               |
| ---------------- | ------------------------------------- |
| clamav           | Herramientas de escaneo manual        |
| clamav-daemon    | Servicio clamd (motor en memoria)     |
| clamav-freshclam | Actualiza firmas                      |
| clamav-milter    | Filtro que conecta Postfix con ClamAV |

<figure><img src=".gitbook/assets/Screenshot 2026-02-22 at 8.35.27 pm.png" alt=""><figcaption></figcaption></figure>

### 3️. Activar servicios

```
sudo systemctl enable --now clamav-daemon
sudo systemctl enable --now clamav-milter
sudo systemctl enable --now clamav-freshclam
```

Verificar:

```
sudo systemctl status clamav-daemon clamav-milter --no-pager
```

Debe aparecer:

```
active (running)
```

<figure><img src=".gitbook/assets/Screenshot 2026-02-22 at 8.38.35 pm.png" alt="" width="375"><figcaption></figcaption></figure>



### Verificación del Motor (clamd)

Logs reales observados:

* Base de datos cargada correctamente.
* Más de 3.6 millones de firmas.
* SelfCheck OK.
* Socket UNIX recibido desde systemd.

Directorio de firmas:

```
/var/lib/clamav/
```

Archivo de log principal:

```
/var/log/clamav/clamav.log
```

***

### &#x20;Configuración de ClamAV-Milter

Archivo:

```
/etc/clamav/clamav-milter.conf
```

Configuración utilizada:

```
MilterSocket /var/run/clamav/clamav-milter.ctl
MilterSocketGroup clamav
MilterSocketMode 666
```

***

**MilterSocket ->** Ruta del socket UNIX que usará Postfix para comunicarse con el milter.

**MilterSocketGroup ->** Grupo propietario del socket.

**MilterSocketMode 666 ->** Permisos amplios en laboratorio para evitar problemas de acceso. En producción se recomienda 660.

### Verificación del Socket

Después de reiniciar:

```
sudo systemctl restart clamav-daemon
sudo systemctl restart clamav-milter
```

Se verifica:

```
ls -la /var/run/clamav/
```

Se observa:

* clamav-milter.ctl
* clamav-milter.pid
* clamd.ctl

Esto confirmó que el milter estaba activo.

<figure><img src=".gitbook/assets/Screenshot 2026-02-23 140658.png" alt=""><figcaption></figcaption></figure>

## 5️. Integrar con Postfix

Se configuró Postfix para delegar el análisis de mensajes al servicio ClamAV-Milter mediante un socket UNIX. Se estableció una política fail-open (`milter_default_action = accept`) para evitar bloqueo de correo en caso de fallo del antivirus.

Editar:

```
/etc/postfix/main.cf
```

Agregar:

```
smtpd_milters = unix:/var/run/clamav/clamav-milter.ctl
non_smtpd_milters = $smtpd_milters
milter_default_action = accept
milter_protocol = 6
```

Reiniciar:

```
sudo systemctl restart postfix
```

Verificar:

```
sudo postconf -n | grep smtpd_milters
```

***

### 6️. Probando funcionamiento:

#### &#x20;Motor antivirus

Crear archivo EICAR:

```
echo 'X5O!P%@AP[4\PZX54(P^)7CC)7}$EICAR-STANDARD-ANTIVIRUS-TEST-FILE!$H+H*' > /tmp/eicar.txt
```

Escanear:

```
clamscan /tmp/eicar.txt
```

Debe mostrar:

```
Eicar-Signature FOUND
```

Esto prueba que el motor funciona.

<figure><img src=".gitbook/assets/Screenshot 2026-02-23 134945.png" alt="" width="375"><figcaption></figcaption></figure>

### Prueba mediante Telnet

Conectar al servidor SMTP:

```
telnet localhost 25
```

Enviar manualmente:

<pre><code>EHLO test
MAIL FROM:&#x3C;test@lala.local>
RCPT TO:&#x3C;ameliana@lala.local>
DATA
Subject: EICAR Test

<strong>X5O!P%@AP[4\PZX54(P^)7CC)7}$EICAR-STANDARD-ANTIVIRUS-TEST-FILE!$H+H*
</strong>.
QUIT
</code></pre>

### Verificación en registros

Ejecutar:

```
sudo journalctl -u postfix -u clamav-milter -n 50 --no-pager

o 

sudo tail -n 80 /var/log/mail.log | grep -Ei "clamav|milter|infect|eicar"
```

Se realizó una prueba de envío SMTP conteniendo el patrón EICAR. La interacción registrada en el sistema confirmó que el mensaje fue analizado por ClamAV-Milter antes de su procesamiento final por Postfix.

<figure><img src=".gitbook/assets/Screenshot 2026-02-23 142752.png" alt="" width="375"><figcaption></figcaption></figure>

<figure><img src=".gitbook/assets/Screenshot 2026-02-23 142810 (1).png" alt=""><figcaption></figcaption></figure>

#### &#x20;Verificar que NO llegó al buzón

```
ls /home/usuario/Maildir/new
```

El mensaje no debe existir.

#### Verificar sockets activos

```
test -S /var/run/clamav/clamav-milter.ctl && echo OK
```

<figure><img src=".gitbook/assets/Screenshot 2026-02-23 170949.png" alt=""><figcaption></figcaption></figure>

Los headers `X-Virus-Scanned` y `X-Virus-Status: Clean` confirman que ClamAV analizó el mensaje durante la sesión SMTP y no detectó malware. Esto demuestra que el milter está activo e integrado correctamente con Postfix.
