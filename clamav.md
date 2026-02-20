---
description: >-
  Se implementó ClamAV como sistema antivirus integrado en Postfix mediante el
  mecanismo Milter, permitiendo analizar los correos antes de su entrega. El
  objetivo es detectar y bloquear contenido malo.
---

# ClamAV

¿Qué es Postfix?

**Postfix** es un **MTA (Mail Transfer Agent)**.

&#x20;¿Qué significa eso?

Un MTA es el software que:

* Recibe correos (SMTP)
* Decide a dónde enviarlos
* Los entrega al buzón local o a otro servidor

Es el “corazón” del servidor de correo.

## 2️. ¿Qué es ClamAV?

**ClamAV** es un **motor antivirus de código abierto**.

Su función es:

* Analizar archivos
* Compararlos contra firmas de malware conocidas
* Detectar amenazas

No es un firewall.\
No bloquea conexiones.\
Solo analiza contenido.

## 3️. ¿Qué es clamd?

`clamd` es el **daemon** (servicio en segundo plano) de ClamAV.

#### &#x20;¿Qué es un daemon?

Un daemon es un proceso que:

* Se ejecuta continuamente en segundo plano
* No necesita intervención directa del usuario
* Espera solicitudes

En nuestro caso:

`clamd` espera que alguien le envíe un archivo para analizar.

***

## 4️. ¿Qué es un milter?

**Milter = Mail Filter**

Es un mecanismo que permite a Postfix:

* Enviar el mensaje a un filtro externo
* Esperar una respuesta
* Aceptar o rechazar el mensaje según esa respuesta

En nuestro caso:

Postfix -> llama al milter\
Milter -> envía el mensaje a clamd\
Clamd -> responde “infectado” o “limpio”\
Milter -> le dice a Postfix qué hacer

***

## 5️. ¿Qué es un socket?

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

### ¿Por qué usar socket y no puerto TCP?

Porque:

* Es más rápido
* Es más seguro (solo local)
* No expone el servicio a la red

***

## 6️. ¿Qué es /run o /var/run?

Son directorios donde el sistema guarda:

* Archivos temporales
* PID de procesos
* Sockets
* Información de servicios activos

Son temporales y se recrean en cada arranque.

***

## 7️. ¿Qué es Maildir?

Maildir es un formato de buzón.

En vez de guardar todos los correos en un solo archivo (mbox),\
Maildir guarda cada mensaje como un archivo separado.

Estructura:

```
Maildir/
   ├── new/
   ├── cur/
   └── tmp/
```

Esto evita corrupción y mejora rendimiento.

***

## 8️. ¿Qué es un archivo EICAR?

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

## 9️. ¿Qué significa “milter-reject”?

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

## ¿Qué es un log?

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

## 1️¿Qué es chroot?&#x20;

Chroot es un mecanismo de aislamiento.

Hace que un servicio vea solo una parte del sistema de archivos,\
como si fuera su “mini sistema”.

Por eso tuvimos problemas con el socket inicialmente.

***

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

## IMPLEMENTACIÓN DE CLAMAV EN POSTFIX (PASO A PASO COMPLETO)

***

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

***

## 3️. Activar servicios

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

***

## 4️. Verificar sockets del antivirus

```
ls -la /var/run/clamav/
```

Debe existir:

* clamd.ctl
* clamav-milter.ctl

#### Qué es esto

Son **sockets UNIX** → canales de comunicación interna entre procesos.

***

## 5️. Integrar con Postfix

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

### 6️. Cómo PROBAR que funciona

#### PRUEBA 1 — Motor antivirus

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

***

#### PRUEBA 2 — Flujo Postfix (bloqueo real)

Enviar con sendmail:

```
sudo sendmail -v usuario@dominio < /tmp/eicar.txt
```

Revisar log:

```
sudo tail -n 120 /var/log/mail.log
```

Debe aparecer:

```
milter-reject
Command rejected
```

Esto prueba que el milter está interceptando.

***

#### PRUEBA 3 — Desde Thunderbird (prueba real de usuario)

#### Paso 1: Crear archivo eicar.txt en tu PC

Contenido exacto:

```
X5O!P%@AP[4\PZX54(P^)7CC)7}$EICAR-STANDARD-ANTIVIRUS-TEST-FILE!$H+H*
```

#### Paso 2: Adjuntarlo en Thunderbird

* Crear nuevo correo
* Adjuntar eicar.txt
* Enviar al servidor

#### Paso 3: Resultado esperado

El envío debe:

* Fallar
* Mostrar error SMTP
* No entregarse

#### Paso 4: Ver logs en servidor

```
sudo tail -f /var/log/mail.log
```

Debe verse algo como:

```
milter-reject
Infected
```

Esto demuestra protección en entorno real.

***

#### &#x20;PRUEBA 4 — Verificar que NO llegó al buzón

```
ls /home/usuario/Maildir/new
```

El mensaje no debe existir.

***

#### PRUEBA 5 — Verificar sockets activos

```
test -S /var/run/clamav/clamav-milter.ctl && echo OK
```
