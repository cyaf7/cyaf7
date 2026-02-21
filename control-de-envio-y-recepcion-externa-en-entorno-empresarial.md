# Control de envío y recepción externa en entorno empresarial

### 1. Introducción

En un entorno empresarial no todos los usuarios deben tener los mismos privilegios. Es habitual que:

* Algunos usuarios puedan enviar correos al exterior.
* Otros solo puedan comunicarse dentro del dominio.
* Algunos usuarios puedan recibir correo externo.
* Otros solo deban recibir correo interno.

Este control no se realiza a nivel de sistema operativo, sino a nivel del MTA (Postfix) y mediante políticas almacenadas en base de datos.

***

## 2. Modelo de datos para control de políticas

En la base de datos `mailserver`, la tabla `users` puede ampliarse para incluir permisos específicos.

```sql
ALTER TABLE users
ADD COLUMN allow_external_send BOOLEAN DEFAULT FALSE,
ADD COLUMN allow_external_receive BOOLEAN DEFAULT TRUE;
```

#### Significado de los campos

* `allow_external_send`
  * TRUE → puede enviar correo fuera de los dominios gestionados.
  * FALSE → solo puede enviar correo interno.
* `allow_external_receive`
  * TRUE → puede recibir correo desde Internet.
  * FALSE → solo puede recibir correo desde usuarios internos autenticados.

***

## 3. Control de envío externo (Outbound Control)

### 3.1 Concepto

Cuando un usuario autenticado intenta enviar correo, Postfix puede:

1. Validar que el usuario existe.
2. Consultar si tiene permiso para enviar fuera.
3. Evaluar si el dominio destinatario es interno o externo.
4. Aceptar o rechazar el envío.

***

### 3.2 Archivo de consulta MySQL

Ruta:

```
/etc/postfix/mysql-allow-external-send.cf
```

Contenido:

```ini
user = mailuser
password = password_segura
hosts = 127.0.0.1
dbname = mailserver
query = SELECT allow_external_send FROM users WHERE email='%s'
```

Este archivo permite a Postfix consultar si el usuario autenticado tiene permiso para enviar correo externo.

***

### 3.3 Definición de dominios internos

Crear archivo:

```
/etc/postfix/internal_domains
```

Contenido:

```
empresa.com OK
empresa.es OK
filial.com OK
proyectos.net OK
```

Generar mapa:

```bash
sudo postmap /etc/postfix/internal_domains
```

***

### 3.4 Configuración en main.cf

Archivo:

```
/etc/postfix/main.cf
```

Añadir o modificar:

```ini
smtpd_sender_restrictions =
    check_sender_access mysql:/etc/postfix/mysql-allow-external-send.cf,
    permit_sasl_authenticated

smtpd_recipient_restrictions =
    permit_mynetworks,
    permit_sasl_authenticated,
    check_recipient_access hash:/etc/postfix/internal_domains,
    reject_unauth_destination
```

***

### 3.5 Lógica de funcionamiento

1. Usuario se autentica.
2. Postfix consulta `allow_external_send`.
3. Si el destinatario pertenece a un dominio interno → permitido.
4. Si el destinatario es externo:
   * Si allow\_external\_send = TRUE → permitido.
   * Si allow\_external\_send = FALSE → rechazado.

Este mecanismo evita que cuentas internas limitadas puedan enviar correo a Internet.

***

## 4. Control de recepción externa (Inbound Control)

### 4.1 Concepto

Cuando el servidor recibe correo desde Internet (puerto 25), debe:

1. Verificar que el dominio es gestionado por el servidor.
2. Verificar que el usuario existe.
3. Consultar si puede recibir correo externo.

***

### 4.2 Archivo de consulta MySQL

Ruta:

```
/etc/postfix/mysql-allow-external-receive.cf
```

Contenido:

```ini
user = mailuser
password = password_segura
hosts = 127.0.0.1
dbname = mailserver
query = SELECT allow_external_receive FROM users WHERE email='%s'
```

***

### 4.3 Configuración en main.cf

```ini
smtpd_recipient_restrictions =
    check_recipient_access mysql:/etc/postfix/mysql-allow-external-receive.cf,
    permit_mynetworks,
    permit_sasl_authenticated,
    reject_unauth_destination
```

***

### 4.4 Lógica de funcionamiento

Cuando llega un correo desde Internet:

1. Postfix verifica que el dominio está en `virtual_mailbox_domains`.
2. Verifica que el usuario existe.
3. Consulta `allow_external_receive`.
4. Si el valor es FALSE y el origen es externo → rechaza.

***

## 5. Seguridad crítica: evitar Open Relay

La directiva:

```ini
reject_unauth_destination
```

Es obligatoria en cualquier servidor empresarial.

Su función es:

* Impedir que el servidor reenvíe correo hacia dominios externos no gestionados.
* Evitar que el servidor se convierta en un relay abierto.

Sin esta línea, el servidor podría ser explotado para enviar spam masivo.

***

## 6. Flujo completo en entorno empresarial

### 6.1 Envío

1. Usuario virtual se autentica mediante SASL.
2. Postfix consulta base de datos.
3. Se verifica permiso de envío externo.
4. Si procede, el correo se firma (DKIM).
5. Se envía a Internet.

***

### 6.2 Recepción

1. Servidor externo conecta al puerto 25.
2. Postfix verifica dominio gestionado.
3. Verifica usuario existente.
4. Verifica permiso de recepción externa.
5. Entrega al buzón virtual.

***

## 8. Resumen conceptual

| Elemento           | Ubicación                   | Función                      |
| ------------------ | --------------------------- | ---------------------------- |
| Permisos envío     | MySQL                       | Control granular por usuario |
| Permisos recepción | MySQL                       | Control de recepción externa |
| Restricciones      | main.cf                     | Aplicación de política       |
| Dominios internos  | internal\_domains           | Definición de red interna    |
| Protección relay   | reject\_unauth\_destination | Seguridad básica SMTP        |
