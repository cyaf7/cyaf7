# Error principal con Gajim

## Resolución del problema TLS en XMPP HTTP Upload (puerto 5443)

### Contexto

El chat XMPP funcionaba correctamente en el puerto 5222, pero al intentar enviar archivos mediante `mod_http_upload` (XEP-0363) aparecía el error:

The signing certificate authority is not known

El problema ocurría tanto en:

* Debian servidor (172.20.10.4) donde también estaba instalado Gajim
* Debian cliente (172.20.10.6)

***

### Causa del problema

Se verificó el certificado del servicio HTTPS (puerto 5443) con:

```
openssl s_client -connect aoki.lala.local:5443
```

El resultado mostraba:

* subject=CN=aoki.lala.local
* issuer=CN=aoki.lala.local
* Verification error: self-signed certificate

Esto significa que el certificado del servidor era auto-firmado.\
Ningún cliente podía confiar en él porque no estaba firmado por una Autoridad Certificadora (CA).

El módulo HTTP Upload exige validación TLS correcta, por eso Gajim rechazaba la conexión.

***

### Objetivo de la solución

Para resolverlo se necesitaba:

1. Crear una CA propia para el laboratorio.
2. Firmar el certificado del servidor con esa CA.
3. Instalar la CA en todas las máquinas que actúan como cliente.

***

## Paso 1 — Crear una CA propia (en 172.20.10.4)

```
sudo mkdir -p /opt/ejabberd/conf/pki
cd /opt/ejabberd/conf/pki

sudo openssl genrsa -out lab-ca.key 4096

sudo openssl req -x509 -new -nodes \
  -key lab-ca.key \
  -sha256 -days 3650 \
  -subj "/CN=LalaLab-CA" \
  -out lab-ca.crt
```

Resultado:

* lab-ca.key → clave privada de la CA
* lab-ca.crt → certificado público de la CA

***

## Paso 2 — Crear certificado del servidor firmado por la CA

### 1. Crear clave del servidor

```
sudo openssl genrsa -out aoki.key 2048
```

### 2. Crear archivo de configuración con SAN

```
sudo nano aoki.cnf
```

Contenido:

```
[req]
default_bits = 2048
prompt = no
default_md = sha256
distinguished_name = dn
req_extensions = req_ext

[dn]
CN = aoki.lala.local

[req_ext]
subjectAltName = @alt_names

[alt_names]
DNS.1 = aoki.lala.local
IP.1 = 172.20.10.4
```

El SAN es obligatorio en TLS moderno. Sin él, los clientes rechazan el certificado.

### 3. Crear CSR

```
sudo openssl req -new -key aoki.key -out aoki.csr -config aoki.cnf
```

### 4. Firmar con la CA

```
sudo openssl x509 -req \
  -in aoki.csr \
  -CA lab-ca.crt \
  -CAkey lab-ca.key \
  -CAcreateserial \
  -out aoki.crt \
  -days 365 \
  -sha256 \
  -extensions req_ext \
  -extfile aoki.cnf
```

### 5. Crear server.pem para ejabberd

```
sudo bash -c "cat aoki.key aoki.crt > /opt/ejabberd/conf/server.pem"
sudo chown ejabberd:ejabberd /opt/ejabberd/conf/server.pem
sudo chmod 600 /opt/ejabberd/conf/server.pem
```

### 6. Reiniciar ejabberd

```
sudo systemctl restart ejabberd
```

***

### Verificación

```
openssl s_client -connect aoki.lala.local:5443 | grep issuer
```

Resultado correcto:

issuer=CN=LalaLab-CA

Y al final del handshake:

Verify return code: 0 (ok)

Ya no era self-signed.

***

## Paso 3 — Instalar la CA en el servidor (porque también usa Gajim)

En 172.20.10.4:

```
sudo cp /opt/ejabberd/conf/pki/lab-ca.crt \
  /usr/local/share/ca-certificates/lalalab.crt

sudo update-ca-certificates
```

Verificación:

```
openssl s_client -connect aoki.lala.local:5443
```

Debe terminar con:

Verify return code: 0 (ok)

***

## Paso 4 — Instalar la CA en la Debian cliente (172.20.10.6)

### Copiar la CA

En 172.20.10.4:

```
scp /opt/ejabberd/conf/pki/lab-ca.crt \
  amelia@172.20.10.6:/home/amelia/
```

### Instalar en el cliente

En 172.20.10.6:

```
sudo cp ~/lab-ca.crt \
  /usr/local/share/ca-certificates/lalalab.crt

sudo update-ca-certificates
```

### Verificar

```
openssl s_client -connect aoki.lala.local:5443
```

Debe finalizar con:

Verify return code: 0 (ok)

***

## Resultado final

Después de:

* Crear una CA propia
* Firmar el certificado del servidor correctamente
* Configurar SAN adecuado
* Instalar la CA en servidor y cliente
* Reiniciar Gajim

El módulo HTTPS:

https://aoki.lala.local:5443/upload

funciona correctamente y el upload XMPP queda validado con TLS confiable en entorno de laboratorio.

***

## Conclusión técnica

El problema no era el módulo upload, sino la falta de una PKI mínima funcional.

Esta práctica demuestra:

* Diferencia entre certificado auto-firmado y certificado firmado por CA
* Importancia del SAN en TLS moderno
* Funcionamiento del trust store en Debian
* Cómo validar un servicio TLS con openssl
* Que cualquier máquina que actúe como cliente debe confiar explícitamente en la CA
