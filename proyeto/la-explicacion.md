# La explicación

### Elección inicial de la arquitectura

Al inicio del proyecto, una de las principales dificultades fue decidir qué arquitectura utilizar para construir la infraestructura. Después de varias discusiones y análisis, se decidió implementar una solución basada en MicroCloud, con el objetivo de trabajar con virtualización, alta disponibilidad y almacenamiento distribuido.

Esta decisión implicó construir toda la infraestructura prácticamente desde cero, incluyendo:

* red;
* firewall;
* almacenamiento distribuido;
* virtualización;
* automatización;
* conectividad externa.



### Adquisición y preparación del hardware

El primer problema encontrado estuvo relacionado con el hardware disponible. Inicialmente, el ordenador destinado al firewall únicamente disponía de una interfaz de red física (NIC), lo que impedía separar correctamente la WAN y la LAN.

Para solucionar esta limitación, se adquirió un adaptador USB Ethernet externo, permitiendo añadir una segunda interfaz de red y posibilitando la implementación del firewall en bare metal.

Sin embargo, la obtención de todo el hardware necesario tomó más tiempo del esperado, provocando que aproximadamente una semana y media del proyecto se consumiera únicamente en la preparación inicial de la infraestructura.

⸻

### Problemas con VLANs y configuración del switch

Una vez montada la infraestructura física, comenzó la configuración de las VLANs. Durante esta etapa surgieron múltiples dificultades relacionadas con el switch utilizado, ya que era un modelo antiguo cuya configuración difería significativamente de la esperada.

Inicialmente, las VLANs eran creadas correctamente, pero no existía comunicación entre los equipos y el firewall, incluso antes de instalar MicroCloud.

Durante el proceso de troubleshooting, también se descubrió que uno de los cables de red estaba defectuoso. Dicho cable era precisamente el encargado de proporcionar conexión a internet. Tras reemplazarlo, finalmente fue posible establecer conectividad entre firewall, switch y ordenadores.

Otro error importante fue la configuración incorrecta de las interfaces VLAN. En un principio, las VLANs fueron asociadas a la interfaz WAN (em0) en lugar de la interfaz LAN (ue0).

Como consecuencia:

* no existía conectividad interna;
* los hosts no podían comunicarse con el firewall;
* las VLANs quedaban asociadas a la interfaz equivocada.

Tras corregir la configuración y asociar las VLANs a la interfaz correcta (ue0), la conectividad comenzó a funcionar adecuadamente.

Además, durante la configuración del switch se produjo una confusión importante relacionada con una funcionalidad denominada LACP (Link Aggregation Control Protocol).

Inicialmente se pensó que dicha opción correspondía a la configuración de trunks para VLANs, es decir, permitir el paso de múltiples VLANs a través de un mismo puerto o cable de red.

Sin embargo, posteriormente se descubrió que:

* la funcionalidad trunk y LACP eran conceptos distintos;
* el trunk permitía transportar múltiples VLANs etiquetadas por el mismo puerto;
* LACP estaba orientado a agregación de enlaces y balanceo de ancho de banda.

Esta confusión generó retrasos en la configuración inicial de la red, ya que el switch utilizaba terminología diferente a la esperada y la documentación disponible era limitada debido a la antigüedad del equipo.

⸻

#### Configuración de interfaces en los nodos

Cada nodo del cluster necesitaba al menos dos interfaces de red:

* una dedicada al tráfico interno de MicroCloud;
* otra destinada a gestión remota y acceso SSH.

Por ello, fue necesario configurar VLANs individualmente mediante Netplan en cada nodo Ubuntu.

La segmentación de red se convirtió en un requisito fundamental, ya que sin ella MicroCloud no podía funcionar correctamente.

⸻

#### Configuración del firewall

Otro reto importante fue la configuración inicial del firewall. Por defecto, el firewall trabajaba bajo una política deny all, bloqueando tráfico tanto de entrada como de salida.

Fue necesario:

* crear reglas específicas;
* permitir comunicación entre VLANs;
* habilitar acceso a internet;
* configurar el enrutamiento interno.

Debido al tiempo limitado del proyecto, no se realizó un diseño avanzado de subredes. El objetivo principal pasó a ser conseguir que toda la infraestructura funcionara correctamente dentro del plazo disponible.

⸻

#### Instalación de MicroCloud

Una vez estabilizada la red, comenzó la instalación del entorno MicroCloud:

* LXD;
* MicroCeph;
* MicroOVN.

Durante esta etapa se descubrió que todos los nodos debían utilizar exactamente las mismas versiones de cada componente.

El nodo 3 presentaba versiones incompatibles, impidiendo la creación del cluster. Fue necesario reinstalar los componentes y estandarizar las versiones antes de ejecutar:

* microcloud init en el nodo principal;
* microcloud join en los nodos secundarios.

⸻

#### Problemas de almacenamiento y uso de Loop Devices

MicroCeph requiere múltiples discos para funcionar correctamente. Inicialmente se intentó utilizar particiones tradicionales, pero el sistema no reconocía adecuadamente los bloques necesarios para el almacenamiento distribuido.

Tras consultar la documentación oficial de Canonical, se descubrió que la solución recomendada consistía en utilizar Loop Devices.

Los Loop Devices permiten que archivos sean tratados por el sistema operativo como discos independientes, posibilitando:

* replicación;
* balanceo de carga;
* almacenamiento distribuido.

Después de implementar esta solución, MicroCeph consiguió detectar correctamente los dispositivos de almacenamiento.

⸻

#### Virtualización y alta disponibilidad

Con el cluster operativo, comenzó la creación de máquinas virtuales.

Se decidió utilizar máquinas virtuales en lugar de contenedores debido al mayor nivel de aislamiento:

* kernel independiente;
* separación más estricta entre servicios;
* mayor seguridad.

Una de las principales ventajas observadas fue la alta disponibilidad proporcionada por MicroCloud. Aunque una máquina virtual estuviera físicamente ubicada en un nodo concreto, podía ser accedida desde cualquier nodo del cluster.

Además:

* los discos eran replicados automáticamente;
* el balanceo de carga se distribuía entre nodos;
* nuevos nodos podían añadirse automáticamente al cluster con sincronización completa.

⸻

#### Redes virtuales con MicroOVN

Posteriormente se configuraron Open Virtual Networks (OVN) para permitir la comunicación entre máquinas virtuales distribuidas en distintos nodos.

Esta etapa fue fundamental para garantizar la comunicación correcta entre servicios internos.

⸻

Instalación de servicios e integración de la aplicación

La instalación de servicios dentro de las máquinas virtuales tomó aproximadamente entre dos y tres semanas.

Posteriormente comenzó la integración del código de la aplicación alojado en GitHub.

El flujo de transferencia funcionaba de la siguiente forma:

1. ordenador con entorno gráfico;
2. máquina de Management;
3. nodo del cluster;
4. máquina virtual final.

Durante este proceso surgieron numerosos problemas relacionados con:

* incompatibilidad de versiones;
* diferencias entre entornos;
* dependencias faltantes;
* configuraciones locales incompatibles con producción.

⸻

### Publicación externa y Cloudflare Tunnel

Otro desafío importante fue publicar la aplicación externamente.

Debido a que no existía acceso al firewall principal de la escuela, no era posible abrir puertos directamente hacia internet.

La solución elegida fue utilizar:

* un dominio gratuito obtenido en Nominalia;
* gestión DNS mediante Cloudflare;
* Cloudflare Tunnel para exponer la aplicación de forma segura.

Fue necesario:

* modificar los Name Servers del dominio;
* configurar el DNS en Cloudflare;
* crear el túnel entre la DMZ y el exterior.

Gracias a esta solución, fue posible publicar la aplicación sin necesidad de acceso directo al firewall institucional.

⸻

### Automatización con Ansible

A medida que aumentó el número de máquinas, las configuraciones manuales dejaron de ser viables.

Por ello se utilizó Ansible para automatizar:

* instalación de agentes Wazuh;
* configuración de Fail2Ban;
* despliegue de servicios;
* configuraciones repetitivas en nodos y máquinas virtuales.

La automatización fue esencial para mantener consistencia y rapidez en la administración de la infraestructura.

⸻

### Sistema de backups

Inicialmente no existía una máquina dedicada exclusivamente a backups.

Posteriormente se reutilizó un ordenador que ya se empleaba para la gestión del switch, moviéndolo a la VLAN 40, destinada específicamente a backups.

Además:

* se configuraron snapshots de las máquinas virtuales en MicroCloud;
* se implementaron backups semanales en un disco externo SanDisk.

⸻

#### Problemas específicos del Nodo 2

El nodo 2 presentó múltiples problemas relacionados con almacenamiento.

Inicialmente:

* disponía únicamente de 44 GB útiles;
* el sistema de archivos generaba errores;
* los logs crecían excesivamente.

En un momento determinado, el sistema llegó a acumular aproximadamente 40 GB únicamente en logs.

Como medida preventiva, las máquinas virtuales fueron migradas temporalmente hacia los nodos 1 y 3.

Posteriormente, el almacenamiento fue recreado correctamente y el nodo pasó a disponer de aproximadamente 300 GB funcionales y estables.

⸻

### Resultados obtenidos

Al finalizar el proyecto, la infraestructura consiguió operar de forma estable y funcional:

* cluster MicroCloud operativo;
* alta disponibilidad;
* almacenamiento distribuido;
* virtualización funcional;
* automatización mediante Ansible;
* backups y snapshots;
* publicación externa mediante Cloudflare Tunnel.

La aplicación ya se encuentra funcional, aunque todavía requiere mejoras visuales y optimización del pipeline de actualización del código.

⸻

### Limitaciones y próximos pasos

A pesar del éxito técnico del proyecto, quedó claro que mantener una infraestructura basada en MicroCloud requiere una gran cantidad de hardware:

* más memoria RAM;
* mayor capacidad de almacenamiento;
* expansión física de nodos.

Debido al elevado coste de escalabilidad local, los próximos pasos del proyecto estarán enfocados en migrar hacia soluciones cloud como:

* Amazon Web Services
* Microsoft Azure

⸻

### Conclusión

El proyecto representó una experiencia extremadamente enriquecedora tanto a nivel técnico como profesional.

A pesar de enfrentar numerosos problemas relacionados con:

* redes;
* virtualización;
* almacenamiento;
* automatización;
* compatibilidad;
* conectividad externa;

el equipo consiguió construir una infraestructura completa desde cero utilizando tecnologías que todavía no habían sido trabajadas en clase.

Además del aprendizaje técnico, el proyecto proporcionó experiencia práctica en:

* troubleshooting;
* trabajo en equipo;
* resolución de problemas reales;
* adaptación ante fallos de infraestructura.

Aunque fue un proyecto complejo y desafiante, también resultó una experiencia muy motivadora, valiosa y divertida para el desarrollo profesional futuro.
