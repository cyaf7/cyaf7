# Implementación de Rspamd en servidor Postfix

## 1. Introducción

Con el objetivo de reforzar el sistema de seguridad del servidor de correo electrónico, fue implementado Rspamd como solución de filtrado antispam integrada mediante el mecanismo Milter de Postfix.

El entorno ya contaba con autenticación SPF, DKIM y DMARC correctamente configuradas, además de servicios complementarios para almacenamiento de correo y resolución DNS.

La integración se realizó en modo milter, permitiendo que cada mensaje SMTP recibido fuese analizado en tiempo real antes de su entrega final.

### Que es Rspamd?

Rspamd analiza los correos que llegan a un servidor y decide si son spam usando reglas y un sistema de puntuación. Examina el contenido, la reputación del remitente y elementos como firmas digitales (DKIM). En lugar de solo bloquear o permitir, le da puntos al mensaje: si parece sospechoso suma puntos, y si parece legítimo los resta. Cuando se usa junto a Postfix (que es el servidor que recibe y envía los correos), Rspamd actúa como el “filtro de seguridad” que revisa los emails antes de que lleguen al usuario.

***

## 2. Requisitos Previos (Infraestructura ya instalada)

Antes de la implementación de Rspamd, el entorno debía contar con:

| Componente | Función                    |
| ---------- | -------------------------- |
| Postfix    | MTA (Mail Transfer Agent)  |
| Dovecot    | Servidor IMAP/POP3         |
| Bind9      | Servidor DNS               |
| OpenDKIM   | Firma DKIM                 |
| ClamAV     | Antivirus                  |
| SPF        | Autenticación de remitente |
| DMARC      | Política de autenticación  |

Estos servicios debían encontrarse funcionando correctamente antes de integrar Rspamd.

***

## 3. Puertos Importantes en la Arquitectura

| Servicio                     | Puerto      | Protocolo   | Función                  |
| ---------------------------- | ----------- | ----------- | ------------------------ |
| Postfix (SMTP)               | 25          | TCP         | Recepción de correo      |
| Rspamd Milter                | 11332       | TCP (local) | Comunicación con Postfix |
| Rspamd Controller (opcional) | 11334       | HTTP        | Interfaz web             |
| Dovecot IMAP                 | 143         | TCP         | Acceso correo            |
| Dovecot IMAPS                | 993         | TCP         | IMAP seguro              |
| DNS (Bind9)                  | 53          | TCP/UDP     | Resolución SPF/DKIM      |
| ClamAV-Milter                | Socket Unix | Local       | Antivirus                |
| OpenDKIM                     | Socket Unix | Local       | Firma DKIM               |

En esta implementación se utilizó principalmente el puerto:

```
127.0.0.1:11332
```

para el worker milter de Rspamd.

***

## 4. Instalación de Rspamd

Se procedió a la instalación del paquete oficial:

```
sudo apt update
sudo apt install rspamd
```

Verificación del servicio:

```
sudo systemctl status rspamd
```

El servicio debe encontrarse activo (active/running).

***

## 5. Estructura de Directorios Relevantes

| Ruta                    | Descripción                 |
| ----------------------- | --------------------------- |
| /etc/rspamd/            | Configuración principal     |
| /etc/rspamd/local.d/    | Configuración personalizada |
| /etc/rspamd/override.d/ | Reemplazo total de módulos  |
| /var/lib/rspamd/        | Datos y estadísticas        |
| /run/rspamd/            | Socket y PID                |
| journalctl -u rspamd    | Logs del servicio           |

Las modificaciones fueron realizadas exclusivamente en `local.d`.

***

## 6. Configuración del Worker Milter

Archivo configurado:

```
/etc/rspamd/local.d/worker-proxy.inc
```

Contenido aplicado:

```
bind_socket = "127.0.0.1:11332";
milter = yes;
timeout = 120s;

milter_headers {
  extended = true;
}
```

Explicación técnica:

* `bind_socket` define el punto de conexión.
* `milter = yes` activa modo milter.
* `timeout` evita cortes por consultas DNS.
* `extended = true` permite insertar cabeceras detalladas.

Reinicio del servicio:

```
sudo systemctl restart rspamd
```

Verificación de escucha:

```
sudo ss -lntp | grep 11332
```

Debe mostrarse Rspamd escuchando en 127.0.0.1:11332.

***

## 7. Integración con Postfix (Cadena de Milters)

Archivo:

```
/etc/postfix/main.cf
```

Configuración aplicada:

```
smtpd_milters = unix:/var/run/clamav/clamav-milter.ctl, unix:/run/opendkim/opendkim.sock, inet:127.0.0.1:11332
non_smtpd_milters = $smtpd_milters
milter_protocol = 6
milter_default_action = accept
```

Explicación:

* ClamAV analiza malware.
* OpenDKIM firma mensajes.
* Rspamd evalúa spam.
* Los filtros se ejecutan en orden.
* `milter_default_action = accept` define comportamiento fail-open.

Reinicio:

```
sudo systemctl restart postfix
```

Verificación:

```
sudo postconf -n | grep -E "smtpd_milters|milter_"
```

***

## 8. Verificación Operativa

### Confirmación de Rspamd activo

```
sudo ss -lntp | grep 11332
```

### Confirmación de milters en Postfix

```
sudo postconf -n | grep smtpd_milters
```

### Confirmación de sockets Unix

ClamAV:

```
ls -la /var/run/clamav/clamav-milter.ctl
```

OpenDKIM:

```
ls -la /run/opendkim/opendkim.sock
```

***

## 9. Pruebas Externas SMTP

Desde máquina distinta:

```
nc -v 172.20.10.3 25
```

Mensaje enviado:

```
EHLO attacker.com
MAIL FROM:<ho@attacker.com>
RCPT TO:<amelia@lala.local>
DATA
From: "Ho Attacker" <ho@attacker.com>
To: amelia@lala.local
Date: Sat, 21 Feb 2026 17:50:00 +0000
Message-ID: <test-ext-001@attacker.com>
Subject: Prueba integración completa

Mensaje de validación.
.
QUIT
```

***

## 10. Evidencia Esperada

En el código fuente del mensaje deben aparecer:

* Authentication-Results
* spf=none o fail
* dkim=none o pass
* dmarc=none o fail
* X-Spam
* X-Rspamd-Score
* X-Rspamd-Queue-Id

***

## 11. Errores Observados

#### 11.1 Connection refused en 127.0.0.1:11332

Causa:\
Rspamd no estaba escuchando o worker no configurado.

Solución:\
Revisión de worker-proxy.inc y reinicio del servicio.

#### 11.2 valid hostname or network address required

Causa:\
Error tipográfico generando `:127.0.0.1:11332`.

Solución:\
Corrección exacta a `inet:127.0.0.1:11332`.

#### 11.3 Socket incorrecto ClamAV

Error:\
No such file or directory.

Causa:\
Ruta incorrecta del socket.

Solución:\
Verificar con `ls -la` y ajustar ruta real.

#### 11.4 skip SPF/DKIM checks for local networks

Causa:\
IP dentro de rango 172.16.0.0/12 considerado local.

Solución:\
Realizar prueba desde IP no considerada local.

#### 11.5 Cabeceras X-Spam no visibles

Causa:\
Mensaje anterior a integración o no pasó por smtpd.

Solución:\
Enviar nuevo mensaje externo tras confirmar configuración.
