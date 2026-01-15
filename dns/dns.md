---
description: >-
  Here is the lab I did in partnership with Ivan Figueroa about DNS, It was
  written in Spanish, but I'm sure that won't be a problem for whoever
  interested.
---

# DNS

1. El DNS (Domain Name System o Sistema de nombre de Dominio) es coordinado y administrado por la ICANN (Corporación de Internet para la asignación de nombres y números) . ICANN asegura que cada nombre de dominio sea único, gestiona los identificadores únicos de Internet, incluyendo las direcciones de protocolo de Internet (IP) y los nombres de dominio.&#x20;

<br>

2. .es - Red.es - El dominio .es lo gestiona Red.es, que es la entidad oficial encargada de todos los dominios de España.

&#x20; .cat - Fundació puntCAT - El .cat está gestionado por la Fundació puntCAT, una fundación que      se encarga de promover y administrar los dominios en catalán.

.edu - EDUCAUSE - El .edu lo gestiona EDUCAUSE, una asociación en Estados Unidos que se encarga de los dominios de universidades y centros educativos acreditados.

ifp.es - IFP (bajo [Red.es](http://red.es)) - El dominio ifp.es pertenece a la organización IFP, pero al ser un .es depende de Red.es, que regula todos los dominios españoles.

<br>

![](<../.gitbook/assets/unknown (3) (1).png>)

<br>

La consulta whois del dominio .es muestra que está gestionado por Red.es, una entidad pública española con sede en Madrid. También aparecen los contactos administrativo y técnico, junto con los servidores de nombres oficiales que se encargan de la resolución de los dominios. El dominio sigue activo, fue creado en 1988 y su gestión se hace desde la web oficial www.nic.es

<br>

![](<../.gitbook/assets/unknown (4) (1).png>)

Al probar el comando whois ifp.es en la terminal aparece un aviso: el dominio .es no tiene un servidor whois público al que se pueda acceder por consola. En su lugar, la consulta debe hacerse directamente en la web oficial de Red.es ([www.dominios.es)](about:blank).



![](<../.gitbook/assets/unknown (5) (1).png>)

En la página oficial de dominios.es sí se muestra la información del dominio ifp.es: está activo, registrado a nombre del Centro Superior de Altos Estudios Internacionales SLU, con alta en 2005 y vigencia hasta 2025. El agente registrador es Nominalia.<br>

3\.

![](<../.gitbook/assets/unknown (6) (1).png>)

192.168.64.1 es el DNS de mi red local.

9.9.9.9 es un quad9 (DNS publico)

198.153.194.1 es el DNS del Servidor SecurityServices.

&#x20;208.67.220.123 es el OpenDNS (Cisco)&#x20;

<br>

Dentro de cada bloque:

* Cached Name - tiempo en milisegundos que tarda el servidor DNS en responder a consultas que ya tiene guardadas en su caché.
* Uncached Name - tiempo en milisegundos que tarda en responder a consultas nuevas, que no estaban en caché.
* &#x20;DotCom Lookup - tiempo en milisegundos para resolver nombres de dominio terminados en .com.
* &#x20;Min / Avg / Max - valores mínimo, promedio y máximo del tiempo de respuesta medido.
* Std.Dev - desviación estándar; indica cuánto varían los tiempos de respuesta (cuanto más bajo, más estable).
* Reliab% - porcentaje de fiabilidad; 100% significa que el servidor respondió siempre sin fallos.

<br>

![](<../.gitbook/assets/unknown (7) (1).png>)

\
![](<../.gitbook/assets/unknown (8) (1).png>)

Lo hemos hecho con linux también, con el comando dig (Domain Information grouper) que sirve para hacer preguntas al sistema DNS y ver respuestas crudas.&#x20;

<br>

A partir de los resultados se concluye que el servidor más rápido es el DNS local (192.168.64.1),ya que al estar dentro de la red responde casi al instante.

De los servidores públicos, los que mostraron mejor rendimiento fueron Quad9 (9.9.9.9), OpenDNS (208.67.220.123) y SecurityServices (198.153.194.1),todos con buena velocidad y un 100% de fiabilidad. Esto confirma que la elección de un DNS influye directamente en la velocidad de resolución de nombres y que, y que además de usar el DNS del router, existen opciones DNS públicas muy rápidas y seguras.

HERRAMIENTAS ONLINE

<br>

* [https://ping.eu](https://ping.eu/) : Permite ver IP públicas y usar utilidades como whois, dns lookup, traceroute y ping.

![](<../.gitbook/assets/unknown (9) (1).png>)

Aquí se muestra los datos básicos de registro, como propietario, empresa y fechas

<br>

* [Dominios.es](https://www.dominios.es/) : Es la herramienta oficial para consultar información sobre dominios con extensión .es.

![](<../.gitbook/assets/unknown (10) (1).png>)

Muestra la disponibilidad de varios dominios con extensión .es.

<br>

* [https://centralops.net/co](https://centralops.net/co): Ofrece el servicio Domain Dossier, que combina en un mismo informe datos de whois, rutas de red y servidores.

![](<../.gitbook/assets/unknown (11) (1).png>)

Informe Domain Dossier en CentralOps: combina WHOIS con datos de ruta y servidores, útil para ver varias comprobaciones en un único informe

* https://www.nslookup.io : Permite revisar registros DNS A, MX, NS, TXT, entre otros de cualquier dominio.

![](<../.gitbook/assets/unknown (12) (1).png>)

Se ve direcciones (A/AAAA), servidores de correo (MX), servidores de nombres (NS)

<br>

* [https://dnslookup.es](https://dnslookup.es/): Realiza un análisis completo de la configuración DNS de un dominio y detecta errores.

### ![](<../.gitbook/assets/unknown (13) (1).png>)

muestra el estado general de la configuración DNS y lista fallos o advertencias que pueden afectar la disponibilid ad o seguridad del dominio

### Qué información me brinda el WHOIS

Cuando realizo una búsqueda WHOIS, obtengo datos sobre un dominio:

* Quién aparece como titular persona o empresa, salvo que esté oculto por privacidad.
* La empresa con la que se registró el dominio.
* Las fechas de creación, renovación y caducidad.
* Los servidores que permiten que el dominio funcione.

En pocas palabras, el WHOIS ayuda a saber a quién pertenece un dominio y en qué condiciones está registrado.

<br>

### Qué es el registry de la base de datos

El registry es la entidad que administra la lista oficial de todos los dominios bajo una extensión. Por ejemplo, en el caso de .es, el registry es NIC Spain, que guarda la base de datos oficial de los nombres terminados en .es.\
Es como el “almacén central” donde se guarda la información oficial de esos dominios.

\
Qué es el registrar del dominio

El registrar es la empresa que vende y gestiona el dominio. Nos da a entender que es como el comercio al que se paga para registrar el nombre. Esa empresa se conecta con el registry para inscribir oficialmente el dominio y también permite renovarlo, cambiar datos o modificar los servidores de nombres.

### Qué es el DNSSEC y qué implicaciones tiene

El DNSSEC (Domain Name System Security Extensions) es un conjunto de medidas que se añaden al sistema de nombres de dominio (DNS) para darle seguridad.\
Se entiende como si al mensaje que me envía un servidor DNS se le añadiera una especie de firma electrónica que permite comprobar que la respuesta realmente viene de la fuente correcta y que no fue modificada en el camino.

Normalmente, cuando un ordenador pregunta “¿qué dirección tiene este dominio?”, recibe una respuesta sin comprobar si es auténtica. Eso abre la puerta a que alguien pueda engañar con una respuesta falsa. Con DNSSEC, en cambio, la respuesta lleva una firma que confirma que la información es legítima.

#### Ventajas del DNSSEC

1. Protección contra fraudes: Evita que un atacante pueda redirigir a una página falsa cuando se crea que estoy entrando a una página legítima.\
   <br>
2. Confianza en la navegación: Da la seguridad de que los nombres de dominio que uso están resolviendo a las direcciones correctas.\
   <br>
3. Mayor seguridad en servicios críticos: Utiliza servicios de correo o banca en línea, DNSSEC reduce el riesgo de que la información acabe en manos equivocadas.\
   <br>

#### Riesgos o desafíos del DNSSEC

1. Complejidad en la configuración: No es tan simple de activar; tanto la empresa que gestiona el dominio (registrar) como la autoridad que administra la extensión (registry) deben coordinarse para que funcione.\
   <br>
2. Errores graves: Si los registros de firma no se configuran bien, el dominio puede dejar de funcionar. En ese caso, los usuarios no podrán acceder hasta que se corrija el error.\
   <br>
3. Mayor tamaño en las respuestas: Como las respuestas DNS llevan información extra firmas digitales, pesan más. Esto puede generar problemas en redes muy limitadas o en dispositivos antiguos.\
   <br>
4. No protege todo: DNSSEC asegura que la dirección sea auténtica, pero no cifra la información que viaja después. Para eso todavía necesita HTTPS.\
   <br>

#### Implicaciones prácticas

Como usuario o administrador, las implicaciones son claras:

* Si un dominio tiene DNSSEC bien configurado, se puede confiar más en que se esta entrando en el sitio real.
* Ser dueño de un dominio, activar DNSSEC da más seguridad, pero debe tener cuidado: un error puede dejarlo inaccesible.<br>

4\.

![](<../.gitbook/assets/unknown (14) (1).png>)

Mediante el comando nslookup se realizaron consultas a distintos dominios (aliexpress.com, google.com, facebook.com, tv3.cat e ifp.es) utilizando servidores DNS públicos. En todos los casos se obtuvieron direcciones IP válidas y coincidentes, aunque las respuestas aparecieron como no autoritativas, ya que provenían de la caché de los resolvers. Los resultados evidencian que los servidores DNS empleados resuelven los dominios de manera coherente y fiable.

![](<../.gitbook/assets/unknown (15) (1).png>)

En el ejercicio realicé dos consultas con el comando nslookup al dominio tv3.cat, usando dos servidores DNS distintos: Google (8.8.8.8) y Cloudflare (1.1.1.1). En ambos casos, la respuesta fue no autoritativa, lo que significa que procedía del caché del resolver y no directamente del servidor autoritativo del dominio. Los dos servidores devolvieron la misma dirección IP (185.104.134.129), mostrando coherencia entre resolutores diferentes.

\
<br>

5\.

La opción -type=soa en nslookup muestra el registro SOA de un dominio, que indica qué servidor es el principal, el correo de contacto y los tiempos básicos que usan los servidores para actualizar o mantener la información.



![](<../.gitbook/assets/unknown (16) (1).png>)

google.com

Origin: ns1.google.com  (servidor primario de la zona.)

Mail addr: dns-admin.google.com (contacto administrativo de Google.)

Serial: 809951109 - (versión de la zona en ese momento.)

Refresh: 900 segundos (15 min).

Retry: 900 segundos (15 min).

Expire: 1800 segundos (30 min).

Minimum: 60 segundos (1 min).

<br>

![](<../.gitbook/assets/unknown (17) (1).png>)

<br>

![](<../.gitbook/assets/unknown (18) (1).png>)

<br>

![](<../.gitbook/assets/unknown (19) (1).png>)

<br>

6\.

El comando nslookup -debug dominio.com muestra el TTL (Time To Live), que indica cuántos segundos un registro DNS permanece en la caché antes de caducar. Cada dominio puede tener valores diferentes.

![](<../.gitbook/assets/unknown (20) (1).png>)

Type = A - dirección IPv4 del dominio.

internet address = 142.250.184.174 → IP de Google.

ttl = 5 - ese registro estará 5 segundos en la caché del DNS antes de caducar.

type = AAAA -  dirección IPv6 del dominio.

has AAAA address = 2a00:1450:4003:808::200e - IP IPv6 de Google.

ttl = 37 - este registro IPv6 se mantendrá 37 segundos en caché.



![](<../.gitbook/assets/unknown (21) (1).png>)

NS (Name Server): indica qué servidores gestionan el dominio.



![](<../.gitbook/assets/unknown (22).png>)

MX (Mail Exchange): indica qué servidores reciben el correo del dominio.



7\.

![](<../.gitbook/assets/unknown (23).png>)

PTR (Pointer): hace la resolución inversa: de una IP obtiene el nombre de dominio.



### 7.1  ¿Cómo se pueden ver qué servidores DNS tienes asignados en Windows?&#x20;

Con el comando ipconfig que se usa en Windows para ver la configuración de red del equipo. Permite saber la dirección IP, la puerta de enlace y los servidores DNS en uso.

![](<../.gitbook/assets/unknown (24).png>)

<br>

7.2  ¿Cómo los puedes cambiar, y el resultado de haberlos cambiado, poniendo como los tres primeros de la lista que te ha dado DNS benchmark de la actividad 1?<br>

Para cambiar los DNS en Windows solo hay que ir al Panel de control, abrir el Centro de redes y recursos compartidos y entrar en la configuración del adaptador de red. En la conexión que usas (Wi-Fi o cable) se entra en Propiedades - TCP/IPv4 y se marca la opción de usar DNS manual. Ahí se escriben los 2 servidores más rápidos que dio el Benchmark. Desde ese momento, el PC usará esos DNS cada vez que navegues por internet.



### **8.  ¿Cómo podemos modificar el servidor DNS en nuestro dispositivo móvil?**<br>

En los móviles Android, cada vez que entras en una web o abres una app, el sistema necesita traducir el nombre del sitio, como google.com, a una dirección IP. Eso lo hace el servidor DNS que tengas configurado. Por defecto, Android usa el que le da tu operador o el router de casa, pero se puede cambiar por otros más rápidos o seguros.

El cambio se hace desde los ajustes de Wi-Fi, entrando en la red que estés usando y abriendo la configuración avanzada. Allí puedes pasar de automático a manual y escribir los servidores DNS que prefieras, por ejemplo los que salieron como más rápidos en el Benchmark. Desde ese momento, el móvil usará esos servidores en lugar de los de tu operador.

En las versiones más recientes de Android también está la opción de DNS privado. En vez de poner direcciones numéricas, se introduce el nombre del servicio, como dns.google o 1dot1dot1dot1.cloudflare-dns.com. Así, todas las conexiones del móvil pasan por ese DNS, lo que puede hacer la navegación más rápida y también más segura.

\
<br>

9\.

<br>

ipconfig

9.1

&#x20;![](<../.gitbook/assets/unknown (25).png>)

El comando ipconfig /displaydns muestra en Windows la lista de dominios que ya fueron consultados y que están guardados en la memoria. Esa caché permite que al volver a acceder a esas páginas, el sistema no tenga que preguntar otra vez al servidor DNS y las cargue más rápido.



\
10\.&#x20;

### **Wireshark**

10.1

&#x20;![](<../.gitbook/assets/unknown (26).png>)

La consulta fue realizado con la IP de facebook esta consulta que es IP: 157.240.243.35<br>

10.2

![](<../.gitbook/assets/unknown (27).png>)

Resolución DNS al acceder al dominio aliexpress.com desde el navegador, mostrando la consulta y la respuesta del servidor DNS.\
<br>

10.3

![](<../.gitbook/assets/unknown (28).png>)

Consulta DNS tipo A al dominio lidl.es y respuesta con la dirección IP correspondiente.



10.4

![](<../.gitbook/assets/unknown (29).png>)

Salida de nslookup al dominio ciber-bbn.es con tipo NS. Se muestran tanto respuestas no autoritativas como la respuesta final de los servidores autoritativos de Cloudflare.

<br>

![](<../.gitbook/assets/unknown (30).png>)

Consulta DNS a ciber-bbn.es preguntando directamente a un servidor autoritativo (Cloudflare). La respuesta que contiene la flag AA, que indica respuesta autoritativa.

\
<br>

10.5 captura de nypizzeria: Alina hemos intentado realizar este apartado con nuestra máquina virtual que tenemos de windows pero no, no has funcionado.

<br>

### 10.6 ¿En qué puerto (tipo) se origina la petición y hacia qué puerto destino?

Cuando se hace una consulta DNS, el ordenador abre un puerto cualquiera (uno de los que se llaman “efímeros”, por encima de 1024) y la envía al puerto 53, que es donde siempre están escuchando los servidores DNS.

<br>

### 10.7 ¿Qué protocolo?

Las consultas viajan usando el protocolo DNS sobre UDP. DNS es el sistema que traduce nombres en direcciones IP, y UDP es un protocolo rápido que no necesita confirmación de cada paquete. Solo si la respuesta es demasiado grande se cambia a TCP.

### 10.8 ¿Cuál es el servidor de DNS?

El servidor DNS es simplemente la máquina a la que tu equipo pregunta por las direcciones. Puede ser tu propio router, que reenvía la petición, o un servidor público como Google (8.8.8.8) o Cloudflare (1.1.1.1).

### 10.9 Tiempo de vida o TTL

Cada respuesta DNS viene con un TTL (Time To Live). Es un número en segundos que indica cuánto tiempo esa información puede guardarse en caché antes de caducar y obligar a pedirla de nuevo.

### 10.10 ¿Qué significa type A y class IN?

Un registro tipo A es el que asocia un nombre de dominio con una dirección IPv4 concreta (por ejemplo, google.com puede resolverse en 142.250.xx.xx). La clase IN indica que ese registro pertenece a Internet, que es la clase estándar usada en la mayoría de consultas DNS.

### 10.11 ¿Qué información aparece en las flags?

En las flags vemos si el mensaje es una consulta o una respuesta, si la respuesta es autoritativa (flag AA), si el cliente pidió recursión (flag RD), si la respuesta fue truncada (flag TC) y también el código de estado (por ejemplo, NOERROR o NXDOMAIN).

<br>

11.1

Lo marcado como 1 es una respuesta DNS negativa que incluye un registro SOA (Start of Authority). Eso significa que el servidor principal de la zona contestó que no existe el nombre buscado. Básicamente, el DNS confirma que no puede resolver esa petición.

SOA: es un registro que indica el servidor principal de una zona DNS y guarda datos básicos de administración.

\
<br>

11.2

Lo marcado como 2 corresponde a una consulta de tipo PTR, usada en la resolución inversa. Aquí se traduce la dirección IP 9.9.9.9 a su nombre de host, que en este caso es dns9.quad9.net. Es como el proceso contrario de un registro A, que va de dominio a IP.

PTR: es un registro que sirve para la resolución inversa, es decir, pasar de una IP a un nombre de dominio.<br>

12\. ![](<../.gitbook/assets/unknown (31).png>)

&#x20;Se realizó por medio servidor Ubuntu y probo el comando host -t soa para varios dominios.

Lo que se pudo visualizar en la captura es, básicamente, qué servidor es el “jefe” o responsable de cada dominio en internet.

[facebook.com](http://facebook.com):  **Dice que el responsable es a.ns.facebook.com y que la persona de contacto está indicada como dns.facebook.com.**

**tv3.cat: Aquí el que manda es un servidor de Amazon ns-797.awsdns-35.net.**

**aliexpress.com: Aparece que el principal es ns1.alibabadns.com.**

[**ifp.es**](http://ifp.es)**:  En este caso el responsable es ns3.acens.net.**<br>

Después de cada servidor, también salen unos números largos. Esos son valores que sirven para controlar cada cuánto tiempo se actualiza la información, cuánto tarda en caducar.



### 13. Uso del comando dig

El comando dig sirve para hacer consultas al sistema de nombres de dominio DNS.

Con dig puedo preguntar a Internet qué servidor responde por un dominio, qué IP tiene, qué servidores de correo usa, o qué servidores de nombres lo gestionan.

<br>

![](<../.gitbook/assets/unknown (32).png>)

dig soa [facebook.com](http://facebook.com) y did soa tv3.cat:  me dice quién es el servidor principal del dominio.<br>

![](<../.gitbook/assets/unknown (33).png>)

dig soa aliexpress y dig soa ifp.es:  me dice quién es el servidor principal del dominio.

\
![](<../.gitbook/assets/unknown (34).png>)

+short A

<br>

Ejecuté el comando dig +short A para los dominios que se nos pidió, que sirve para obtener solo la dirección IP asociada al nombre.

<br>

Cuando pregunté por facebook.com, me devolvió la IP 157.240.243.35.

<br>

Con tv3.cat, apareció la IP 185.104.134.129.

<br>

Para aliexpress.com, salieron dos IPs: 47.246.173.30 y 47.246.173.237.

<br>

Finalmente, con ifp.es, vi dos direcciones: 104.18.14.196 y 104.18.15.196.

\
\
<br>

![](<../.gitbook/assets/unknown (35).png>)

<br>

+short MX

El comando dig +short MX para averiguar qué servidores de correo recibe emails en nombre de cada dominio.

<br>

Para facebook.com, me devolvió smtpin.vvv.facebook.com. Ese es el servidor encargado de gestionar los correos.

<br>

Con tv3.cat, aparecieron dos servidores: smtp1.ccma.cat y smtp2.ccma.cat. Esto significa que tienen más de un servidor de correo, probablemente para repartir la carga o tener respaldo.

<br>

En el caso de aliexpress.com, salió mx2.mail.aliyun.com, lo cual tiene sentido porque AliExpress pertenece al grupo Alibaba, que usa su propia plataforma de correo.

<br>

Finalmente, con ifp.es, hay varios servidores: poster31.iberlayer.com, poster41.iberlayer.com, poster11.iberlayer.com y poster21.iberlayer.com. Aquí usan varios para asegurar que siempre haya un servidor disponible para recibir mensajes.

<br>

![](<../.gitbook/assets/unknown (36).png>)

\
<br>

+short NS



Esta vez utilizamos dig +short NS para ver qué servidores de nombres gestionan cada dominio.

**Para facebook.com, me salieron cuatro:**

d.ns.facebook.com

b.ns.facebook.com

c.ns.facebook.com

a.ns.facebook.com

**Esto significa que Facebook reparte la gestión de su dominio entre varios servidores propios.**<br>

**Con tv3.cat, aparecieron servidores de Amazon (AWS):**

[ns-489.awsdns-61.com](http://ns-489.awsdns-61.com)

ns-1064.awsdns-05.org

ns-797.awsdns-35.net

ns-1711.awsdns-21.co.uk

**Esto me dice que TV3 confía la gestión de su DNS a la infraestructura de Amazon.**



**Para aliexpress.com, vi dos servidores principales:**

ns1.alibabadns.com

ns2.alibabadns.com

**Esto confirma que AliExpress usa sus propios servidores de nombres de Alibaba.**<br>

**Finalmente, para ifp.es, encontré tres servidores de Acens:**

ns7.acens.net

ns4.acens.net

ns3.acens.net

**Es decir, Acens es la empresa que gestiona el DNS de ese dominio.**

<br>

![](<../.gitbook/assets/unknown (37).png>)

![](<../.gitbook/assets/unknown (38).png>)

![](<../.gitbook/assets/unknown (39).png>)

**any:**

de Facebook, TV3, AliExpress e IFP.

Escribí el comando dig facebook.com any.

Con esta consulta me devolvió varios tipos de registros a la vez:

<br>

* Un registro MX, que muestra el servidor de correo: smtpin.vvv.facebook.com.
* Registros HTTPS, que indican servicios relacionados con conexiones seguras.
* El registro SOA, que me dice que el servidor principal es [a.ns.facebook.com](http://a.ns.facebook.com).
* Un registro A, con la dirección IP 157.240.243.35.
* Un registro AAAA, con la dirección en IPv6 2a03:2880:f170:81:face:b00c:0:25de.
* Varios registros NS, que son los servidores de nombres: a.ns.facebook.com, b.ns.facebook.com, c.ns.facebook.com, [d.ns.facebook.com](http://d.ns.facebook.com).
* Y hasta un registro HINFO, que contiene información técnica general.

<br>

En resumen, en una sola consulta obtuve de golpe toda la información clave: el servidor de correo, la IP, los servidores de nombre y el servidor principal de la zona.

<br>

Luego hice lo mismo con dig tv3.cat any.

Aquí los resultados fueron más simples:<br>

* Un registro A con la IP: 185.104.134.129.
* Y cuatro registros NS que son de Amazon:
* [ns-797.awsdns-35.net](http://ns-797.awsdns-35.net)
* [ns-1711.awsdns-21.co.uk](http://ns-1711.awsdns-21.co.uk)
* [ns-489.awsdns-61.com](http://ns-489.awsdns-61.com)
* ns-1064.awsdns-05.org

<br>

En este caso, no aparecieron MX ni otros tipos de registros, pero sí nos quedó claro que la gestión del DNS de TV3 la lleva Amazon.

\
\
<br>
