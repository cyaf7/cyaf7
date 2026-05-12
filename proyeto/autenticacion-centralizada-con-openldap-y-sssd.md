# Autenticación centralizada con OpenLDAP y SSSD

### Descripción general

El proyecto Spotly requiere un mecanismo de autenticación centralizado para gestionar de forma unificada los accesos SSH de los miembros del equipo a los nodos físicos del clúster MicroCloud. El equipo configuró un servidor OpenLDAP en el Admin PC y desplegó el cliente SSSD en los tres nodos físicos.

***

### Arquitectura

El servidor LDAP reside en el Admin PC y no en una VM del clúster por una razón técnica concreta: si el servidor LDAP estuviera en una VM, la autenticación de los nodos dependería de que el clúster estuviera operativo, creando una dependencia circular que impediría el acceso administrativo en caso de fallo.

| Componente       | Máquina  | IP            | Función                |
| ---------------- | -------- | ------------- | ---------------------- |
| OpenLDAP (slapd) | Admin PC | 192.168.20.20 | Servidor de directorio |
| SSSD             | nodo01   | 192.168.20.11 | Cliente LDAP           |
| SSSD             | nodo02   | 192.168.20.12 | Cliente LDAP           |
| SSSD             | nodo03   | 192.168.20.13 | Cliente LDAP           |

***

### Estructura del directorio

```
dc=spotly,dc=local
├── ou=people
│   ├── uid=camilly
│   ├── uid=chris
│   ├── uid=ivan
│   └── uid=prof
└── ou=groups
    ├── cn=spotly-admin
    └── cn=spotly-readonly
```

#### Atributos de usuario

Cada entrada combina las clases `posixAccount`, `shadowAccount`, `inetOrgPerson` y `ldapPublicKey`. Los atributos POSIX obligatorios para autenticación Linux son:

| Atributo        | Ejemplo             | Propósito             |
| --------------- | ------------------- | --------------------- |
| `uid`           | camilly             | Nombre de usuario     |
| `uidNumber`     | 3001                | ID numérico POSIX     |
| `gidNumber`     | 2000                | ID de grupo principal |
| `homeDirectory` | /home/camilly       | Directorio personal   |
| `loginShell`    | /bin/bash           | Shell de inicio       |
| `userPassword`  | (hash SSHA)         | Contraseña cifrada    |
| `sshPublicKey`  | ssh-ed25519 AAAA... | Clave pública SSH     |

***

### Grupos y modelo de permisos

El fichero `/etc/sudoers.d/spotly` en cada nodo define dos niveles de acceso:

```
%spotly-admin ALL=(ALL:ALL) ALL

%spotly-readonly ALL=(ALL) NOPASSWD: \
    /bin/systemctl status *, \
    /usr/bin/journalctl *, \
    /snap/bin/lxc list, \
    /snap/bin/lxc info *, \
    /usr/bin/tail -f /var/log/*
```

| Grupo           | Miembros             | Acceso                    |
| --------------- | -------------------- | ------------------------- |
| spotly-admin    | camilly, chris, ivan | sudo completo             |
| spotly-readonly | prof                 | solo comandos de consulta |

***

### Configuración SSSD

Fichero `/etc/sssd/sssd.conf` en cada nodo (permisos `600`, propietario `root`):

```ini
[sssd]
services = nss, pam, ssh
config_file_version = 2
domains = spotly.local

[domain/spotly.local]
id_provider = ldap
auth_provider = ldap
chpass_provider = ldap

ldap_uri = ldaps://192.168.20.20
ldap_search_base = dc=spotly,dc=local
ldap_default_bind_dn = cn=admin,dc=spotly,dc=local
ldap_default_authtok_type = password
ldap_default_authtok = ********

ldap_user_search_base = ou=people,dc=spotly,dc=local
ldap_group_search_base = ou=groups,dc=spotly,dc=local

ldap_schema = rfc2307
ldap_id_use_start_tls = False
ldap_tls_reqcert = demand
ldap_tls_cacert = /etc/ssl/certs/spotly-ca.crt

cache_credentials = True
use_fully_qualified_names = False
fallback_homedir = /home/%u
default_shell = /bin/bash
```

> **Nota:** El certificado de la CA interna (`spotly-ca.crt`) se distribuye en `/etc/ssl/certs/` de cada nodo para verificar el TLS del servidor LDAP.

***

### Verificación

#### Resolución de usuarios desde LDAP

```bash
getent passwd camilly chris ivan prof
```

Confirmar que los usuarios no existen localmente:

```bash
grep camilly /etc/passwd
# sin salida — el usuario solo existe en LDAP
```

#### Consulta de grupos

```bash
getent group spotly-admin spotly-readonly
```

#### Conectividad al servidor

```bash
ldapsearch -x -H ldap://192.168.20.20 \
  -b "dc=spotly,dc=local" "(uid=camilly)" uid uidNumber
```

Respuesta esperada: `result: 0 Success`

#### Estado del servicio

```bash
sudo systemctl status sssd
```

Subprocesos esperados activos: `sssd_be`, `sssd_nss`, `sssd_pam`, `sssd_ssh`.

#### Validación del modelo de permisos

Iniciar sesión como `prof` y ejecutar:

```bash
sudo apt update
# → Permission denied

sudo systemctl status sssd
# → OK, comando permitido para spotly-readonly
```

***

_Fuente: Ubuntu SSSD with LDAP — https://ubuntu.com/server/docs/how-to-set-up-sssd-with-ldap_

_Fuente: OpenLDAP Administrator's Guide — https://www.openldap.org/doc/admin26/_

_Fuente: SSSD Documentation — https://sssd.io/docs/_
