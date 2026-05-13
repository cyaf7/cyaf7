# Backup y Recuperación

### Estrategia de Backup — Arquitectura de 3 Capas

El backup en Spotly implementa tres capas de redundancia, cada una protegiendo contra un tipo de fallo distinto:

**Capa 1: Replicación en MicroCeph (recuperación en minutos).** MicroCeph replica automáticamente todos los datos en los tres nodos del cluster. Si un nodo físico se apaga, los datos permanecen accesibles a través de los otros dos nodos. Si un disco se corrompe, Ceph detecta la corrupción y rehace el bloque a partir de las réplicas. El factor de replicación es 3: cada bloque existe en tres nodos simultáneamente. El RPO (Recovery Point Objective) es cercano a cero porque la replicación es sincrónica.

**Capa 2: Backup programado a servidor externo (recuperación en horas).** Cada lunes, miércoles y viernes a las 02:00 AM, un playbook de Ansible ejecuta en el servidor de backup (192.168.40.10, VLAN 40) que sincroniza hacia una ubicación remota mediante SCP (Secure Copy Protocol sobre SSH). Los datos respaldados incluyen:

* Exportación completa de máquinas virtuales LXD (snapshots en formato tar.gz).
* Configuración de nodos (archivos de sistema, Netplan, configuración de OVN).
* Bases de datos PostgreSQL (dump completo en SQL).

Estos archivos se almacenan en `/home/ansible/backups_vms/` con estructura de directorios:

* `/home/ansible/backups_vms/vms/NombreVM-AAAA-MM-DD.tar.gz`
* `/home/ansible/backups_vms/nodos/config_nodoXX-AAAA-MM-DD.tar.gz`

La política de retención es: mantener backups de hoy y ayer, eliminar los de hace dos días. Esto requiere 6 conjuntos de backup en la semana (3 días de backup × 2 versiones cada uno), ocupando aproximadamente 60GB en el servidor de backup (estimado para base de datos + VMs + configuración).

**Capa 3: Backup físico externo (recuperación en días).** Semanalmente, se copia un subconjunto de datos críticos a un disco SanDisk externo de 1TB. Este backup físico se almacena fuera del laboratorio para proteger contra desastres (incendio, inundación, robo) que afecten toda la infraestructura. El disco externo se guarda en un lugar seguro y se actualiza manualmente cada viernes.

### Implementación del Backup Programado

#### Acceso SSH Seguro

El servidor de backup (192.168.40.10) accede a los nodos mediante SSH utilizando una clave privada ED25519 almacenada en `/root/.ssh/id_ed25519_ansible`. Esta clave debe ser restringida en permisos: `chmod 600` garantiza que solo root pueda leerla. La clave pública correspondiente se distribuye a todos los nodos en el archivo `~ansible/.ssh/authorized_keys`.

El acceso SSH desde VLAN 40 (backup) a VLAN 20 (management) está permitido por el firewall OPNsense únicamente durante la ventana horaria de backup (02:00 a 06:00 AM). Fuera de esta ventana, OPNsense bloquea todo tráfico desde VLAN 40, aislando completamente el servidor de backup. Esta política se implementa mediante Firewall Schedules en OPNsense.

#### Playbook de Backup

El playbook de backup se almacena en `/root/ansible/playbooks/backup_total.yml` y es ejecutado por cron en el servidor de backup. El playbook realiza las siguientes tareas:

**Exportar máquinas virtuales:** Para cada VM (data, app-core, vision, ldap-server, monitoreo, dmz-receiver), ejecuta `lxc export nombre-vm /tmp/nombre-vm-FECHA.tar.gz --optimized-storage`. El flag `--optimized-storage` comprime el archivo reduciendo el tamaño a aproximadamente 2-3GB por VM.

**Sincronizar configuración de nodos:** Mediante `tar -czvf`, crea un archivo comprimido de directorios críticos de cada nodo: `/etc/netplan`, `/etc/hostname`, `/etc/hosts`, `/var/snap/lxd/common/lxd/networks` (configuración OVN). Se transfiere mediante SCP.

**Exportar base de datos PostgreSQL:** Mediante `pg_dump -h 10.10.20.2 -U spotlyuser spotlydb > spotlydb-FECHA.sql`, genera un dump SQL completo de la base de datos. El archivo SQL incluye esquema, datos, triggers y vistas, permitiendo una restauración completa desde cero.

**Sincronizar archivos al servidor de backup:** Utiliza SCP para copiar todos los archivos generados a `/home/ansible/backups_vms/` en 192.168.40.10. SCP transfiere ficheros de forma segura sobre SSH sin necesidad de un servidor FTP/NFS externo.

**Limpiar backups antiguos:** Después de una transferencia exitosa, el script busca archivos más antiguos que 2 días en `/home/ansible/backups_vms/` y los elimina. La orden es: `find /home/ansible/backups_vms -type f -mtime +1 -delete` (elimina archivos modificados hace más de 1 día).

#### Configuración de Cron

La tarea de backup se ejecuta cada lunes, miércoles y viernes a las 02:00 AM. La entrada de crontab en el servidor de backup es:

```
0 2 * * 1,3,5 /usr/bin/ansible-playbook /root/ansible/playbooks/backup_total.yml >> /root/ansible/logs/backup_cron.log 2>&1
```

Los campos son: minuto (0), hora (2), día del mes (_), mes (_), día de la semana (1=lunes, 3=miércoles, 5=viernes). La salida se redirige a `/root/ansible/logs/backup_cron.log` para auditoría.

Logrotate se configura para rotar los logs de backup: `maxsize 100M, rotate 5, daily, compress`. Esto evita que un único archivo de log consuma demasiado espacio después de meses.

### Recuperación de Datos

#### Restaurar una Máquina Virtual

Si una VM se corrompe o se elimina accidentalmente, se restaura desde el backup más reciente:

```
lxc import /home/ansible/backups_vms/vms/nombre-vm-AAAA-MM-DD.tar.gz
```

LXD detecta automáticamente el tipo de máquina (VM o contenedor) desde el archivo de backup e importa con su nombre original. La VM inicia en estado detenido; usar `lxc start nombre-vm` para arrancarla. Los datos del disco se restauran completamente, pero la VM no retiene el estado de memoria (esto es normal para un backup offline).

#### Restaurar Configuración de un Nodo

Si la configuración de red de un nodo se corrompe (por ejemplo, Netplan se sobrescribe accidentalmente), se restaura desde el backup:

```
cd /tmp
scp ansible@192.168.40.10:/home/ansible/backups_vms/nodos/config_nodo01-AAAA-MM-DD.tar.gz .
tar -xzvf config_nodo01-AAAA-MM-DD.tar.gz -C /
netplan apply
```

Esto desempaqueta los archivos de configuración en sus ubicaciones originales. Ejecutar `netplan apply` recarga la configuración de red sin reboot.

**Advertencia:** Restaurar configuración sobrescribe archivos del sistema. Verificar que la fecha del backup es correcta antes de aplicar. Si hay dudas, restaurar en un directorio temporal primero para inspeccionar.

#### Restaurar la Base de Datos PostgreSQL

Si la base de datos PostgreSQL se corrompe, se restaura desde el dump SQL:

```
scp ansible@192.168.40.10:/home/ansible/backups_vms/spotlydb-AAAA-MM-DD.sql /tmp/
psql -h 10.10.20.2 -U spotlyuser spotlydb < /tmp/spotlydb-AAAA-MM-DD.sql
```

Esto reconecta a la base de datos y ejecuta todas las sentencias SQL del dump. Los triggers, vistas e índices se recrean automáticamente. El tiempo de restauración es proporcional al tamaño del dump: típicamente 5-15 minutos para una base de datos de 2-5GB.

### Estado del Sistema de Backup

El servidor de backup (192.168.40.10) tiene aproximadamente 118GB de capacidad en disco. Con una política de retención de 2 versiones de cada backup (hoy + ayer), el espacio utilizado es estable: cada nueva ejecución de backup elimina los archivos de dos días atrás. Los logs de cron indican cuándo se ejecutó cada backup y si tuvo éxito o errores.

### Auditoría y Monitoreo

Los logs de backup se almacenan en `/root/ansible/logs/backup_cron.log` con timestamp de ejecución, salida de ansible-playbook y códigos de error. Revisar periódicamente estos logs para detectar fallos silenciosos (conexión SSH rechazada, disco lleno en servidor de backup, fallo de lxc export).

Los discos de todos los nodos tienen alertas en Prometheus si el uso de espacio supera el 80%. Si un nodo alcanza 100%, los OSDs de MicroCeph se marcan como full y el cluster rechaza escrituras. El playbook de backup monitorea el espacio disponible antes de ejecutar para evitar llenar discos.

### Próximos Pasos

La infraestructura de backup está en producción y funcional. Los siguientes pasos son:

* Documentar procedimiento de recuperación ante desastre (DR) con RTO y RPO específicos para cada tipo de fallo.
* Probar restauración completa en un entorno de prueba mensualmente para verificar que los backups son válidos.
* Considerar backup offsite a un servicio en la nube (AWS S3, Azure Blob) para la capa 3 si se requiere mayor durabilidad.
