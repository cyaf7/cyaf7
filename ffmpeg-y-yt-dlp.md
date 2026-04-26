# FFmpeg y yt-dlp



###

1. Descripción general de la actividad
2. Máquinas virtuales necesarias
3. Protocolos de streaming
4. Explicacion FFMPEG
5. FFmpeg: definición, instalación y características
6. Contenedores, códecs y remuxing
7. H.264, H.265 y comportamiento del audio
8. yt-dlp: definición, instalación y características
9. Comparativa de códecs AV1, VP9 y H.264
10. Alternativas a FFmpeg y yt-dlp
11. Parte práctica: FFmpeg
12. Parte práctica: yt-dlp
13. Ejemplo propio combinando FFmpeg y yt-dlp
14. Comandos para screenshots
15. Referencias y fuentes

***

## 1. Descripción general de la actividad

El vídeo digital es uno de los pilares fundamentales de la infraestructura de servicios modernos. Desde plataformas de streaming como YouTube o Twitch, hasta sistemas de videoconferencia o cámaras IP de seguridad, todo se apoya en herramientas y protocolos que permiten capturar, comprimir, transmitir y reproducir contenido multimedia de forma eficiente.

En esta actividad se trabaja con dos herramientas de referencia en el mundo multimedia: FFmpeg y yt-dlp.

FFmpeg es el estándar de facto para el procesamiento, conversión y transcodificación de audio y vídeo en entornos profesionales. Es una herramienta de código abierto, funciona en cualquier sistema operativo y es utilizada internamente por múltiples aplicaciones y plataformas multimedia.

yt-dlp es un descargador de vídeo de código abierto, sucesor de youtube-dl, que permite descargar contenido desde múltiples sitios web con control sobre el formato, la calidad y el postprocesamiento del archivo descargado.

También se estudian los principales protocolos de streaming, como RTMP, HLS, RTSP y SRT, ya que son la base del transporte de vídeo a través de internet y redes locales.

**Observación de evaluación:**\
La idea importante de esta actividad no es únicamente ejecutar comandos. Lo fundamental es entender el flujo completo de trabajo multimedia: obtención del contenido, análisis técnico, conversión, compresión, extracción, segmentación y distribución.

***

## 2. Máquinas virtuales necesarias

Para esta actividad se necesita únicamente una máquina virtual, con las siguientes características:

| Parámetro         | Valor recomendado                                    |
| ----------------- | ---------------------------------------------------- |
| Sistema operativo | Ubuntu 22.04 LTS Desktop o Debian 12 Desktop         |
| RAM               | Mínimo 2 GB, recomendado 4 GB                        |
| Disco             | Mínimo 20 GB                                         |
| Entorno gráfico   | Sí, obligatorio                                      |
| Acceso a internet | Sí, para descargar paquetes y vídeos con yt-dlp      |
| Hipervisor        | UTM, en este caso con imagen ARM64 de Ubuntu Desktop |

### 2.1 Por qué se utiliza Desktop y no Server

Se utiliza una distribución con entorno gráfico porque durante la práctica es necesario reproducir y visualizar los vídeos convertidos para comprobar las diferencias de calidad, formato y códec.

Un servidor sin entorno gráfico permitiría instalar y ejecutar FFmpeg, pero no permitiría abrir fácilmente un reproductor multimedia para comprobar visualmente los resultados. En un entorno profesional de servidor se podría utilizar `ffplay`, pruebas automatizadas o análisis por metadatos, pero para una práctica inicial de servicios multimedia resulta más adecuado trabajar con Ubuntu Desktop.

**Observación de evaluación:**\
La elección de Ubuntu Desktop no se debe a que FFmpeg necesite entorno gráfico para funcionar. FFmpeg es una herramienta de línea de comandos. El entorno gráfico se justifica por la necesidad de validar visualmente el resultado de los archivos generados.

### 2.2 Nota para UTM en Apple Silicon / ARM64

Ubuntu 22.04 Desktop para ARM64 funciona correctamente en UTM. En equipos Mac con Apple Silicon es importante descargar la imagen ISO correspondiente a la arquitectura `arm64`, no la versión `amd64`, ya que esta última está pensada para procesadores Intel o AMD de arquitectura x86\_64.

FFmpeg y yt-dlp son compatibles con ARM64, por lo que no hay inconveniente en realizar la práctica en una máquina virtual de este tipo.

***

## 3. Protocolos de streaming

Antes de trabajar directamente con FFmpeg y yt-dlp, es necesario comprender los protocolos que hacen posible la transmisión de vídeo en red.

Un protocolo de streaming define cómo se envía el contenido multimedia desde un origen hasta uno o varios receptores. Dependiendo del caso, se prioriza la baja latencia, la compatibilidad, la seguridad o la tolerancia a pérdidas de red.

### 3.1 Tabla comparativa de protocolos

| Protocolo | Nombre completo              | Funcionalidad principal                                                                                                             | Latencia                             | Red / Transporte                       | Seguridad                                                        | Compatibilidad                                                                                                   |
| --------- | ---------------------------- | ----------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------ | -------------------------------------- | ---------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------- |
| RTMP      | Real-Time Messaging Protocol | Transmisión en tiempo real de audio, vídeo y datos entre un cliente y un servidor, normalmente desde un emisor hacia una plataforma | Baja, aproximadamente 2–5 segundos   | TCP, puerto 1935                       | La versión RTMPS usa TLS/SSL. RTMP básico transmite en claro     | Alta para emisión desde OBS o encoders. Baja para reproducción en navegadores modernos porque Flash ya no existe |
| HLS       | HTTP Live Streaming          | Divide el stream en pequeños segmentos `.ts` servidos sobre HTTP. Es ideal para distribución masiva                                 | Alta, aproximadamente 10–30 segundos | TCP sobre HTTP/HTTPS, puertos 80 o 443 | Hereda la seguridad de HTTPS y puede integrarse con sistemas DRM | Excelente compatibilidad con navegadores modernos, iOS, Android y Smart TVs                                      |
| RTSP      | Real Time Streaming Protocol | Protocolo de control para streaming. Actúa como un mando a distancia del flujo multimedia, con acciones como play, pause o stop     | Muy baja, inferior a 1 segundo       | UDP o TCP, puerto 554                  | No tiene cifrado nativo. RTSPS usa TLS                           | Muy usado en cámaras IP y CCTV. Poco soporte directo en navegadores                                              |
| SRT       | Secure Reliable Transport    | Transmisión fiable sobre redes inestables o de alta latencia, como internet pública. Permite recuperar paquetes perdidos            | Baja, aproximadamente 0,5–2 segundos | UDP, puerto configurable               | Cifrado AES-128/256 integrado                                    | Compatibilidad creciente en FFmpeg, OBS, VLC y entornos broadcast modernos                                       |

### 3.2 Cuándo usar cada protocolo

RTMP se utiliza principalmente para emitir desde un encoder, como OBS, hacia un servidor de streaming como YouTube Live, Twitch o un servidor propio con Nginx-RTMP.

HLS se utiliza para distribuir el stream a muchos espectadores simultáneamente a través de navegadores, móviles, tablets y Smart TVs. Es muy compatible, aunque introduce más latencia porque el contenido se divide en segmentos.

RTSP se utiliza sobre todo en cámaras de seguridad IP y sistemas CCTV. Es útil cuando se requiere latencia muy baja en redes locales o entornos controlados.

SRT se utiliza cuando se necesita fiabilidad en transmisiones a través de internet pública o redes inestables. Es frecuente en retransmisiones deportivas, televisión en directo y producción audiovisual profesional.

**Observación de evaluación:**\
Una forma clara de diferenciar los protocolos es recordar que RTMP suele usarse para enviar contenido hacia una plataforma, mientras que HLS se utiliza para distribuirlo a los usuarios finales. RTSP se asocia mucho a cámaras IP, y SRT destaca por su fiabilidad sobre redes inestables.

***

## 4.&#x20;

## 4.1 Qué es FFmpeg y para qué se utiliza en el procesamiento multimedia

FFmpeg, cuyo nombre se relaciona con Fast Forward Moving Picture Experts Group, es una colección de bibliotecas y herramientas de código abierto para manipular, convertir, grabar y transmitir audio y vídeo. Fue creada en el año 2000 por Fabrice Bellard y actualmente es mantenida por una comunidad activa de desarrolladores.

Se trata de una de las herramientas multimedia más completas y flexibles que existen. Puede leer una gran cantidad de formatos de audio y vídeo, procesarlos de muchas formas distintas y guardarlos en el formato necesario según el caso de uso.

En el ámbito profesional se utiliza para múltiples tareas.

### 4.1.1 Transcodificación

La transcodificación consiste en convertir un vídeo o audio de un códec a otro. Por ejemplo, convertir un vídeo codificado en H.264 a H.265 para ahorrar espacio o ancho de banda.

Esta operación implica decodificar el contenido original y volver a codificarlo en otro formato. Por tanto, suele consumir CPU y puede producir pérdida de calidad si se utiliza compresión con pérdida.

### 4.1.2 Remuxing

El remuxing consiste en cambiar el contenedor de un archivo sin recodificar el vídeo ni el audio. Por ejemplo, pasar de `.mp4` a `.mkv` manteniendo los mismos streams internos.

Esta operación es muy rápida porque no modifica los datos multimedia, solo cambia la forma en la que están empaquetados.

### 4.1.3 Streaming

FFmpeg puede emitir vídeo en tiempo real usando protocolos como RTMP, HLS o SRT. También puede recibir streams de red y procesarlos, grabarlos o retransmitirlos.

### 4.1.4 Edición básica

FFmpeg permite realizar operaciones básicas de edición, como recortar fragmentos, unir clips, añadir marcas de agua, escalar resolución o insertar subtítulos.

### 4.1.5 Extracción de audio

Otra función muy habitual es separar la pista de audio de un vídeo. Esto permite obtener, por ejemplo, un archivo MP3 a partir de un vídeo MP4.

### 4.1.6 Captura de pantalla y grabación

FFmpeg también puede utilizarse para grabar la pantalla del sistema, capturar audio o registrar una fuente multimedia.

### 4.1.7 Procesamiento por lotes

Al ser una herramienta de línea de comandos, FFmpeg se integra fácilmente en scripts de Bash, Python u otros lenguajes. Esto permite automatizar conversiones masivas de archivos.

Internamente, FFmpeg es utilizado por plataformas y aplicaciones como VLC, Plex y muchas herramientas de procesamiento audiovisual.

**Observación de evaluación:**\
La diferencia entre transcodificación y remuxing es una de las ideas más importantes de esta práctica. Transcodificar cambia el códec y suele consumir CPU. Remuxing cambia el contenedor y no debería producir pérdida de calidad.

***

## 4.2 Página oficial y documentación técnica de FFmpeg

La página oficial de FFmpeg es:

```
https://ffmpeg.org
```

La documentación técnica completa se encuentra en:

```
https://ffmpeg.org/documentation.html
```

Otros recursos oficiales importantes son:

```
https://ffmpeg.org/ffmpeg.html
```

Manual de la herramienta `ffmpeg`.

```
https://ffmpeg.org/ffmpeg-codecs.html
```

Lista de códecs soportados.

```
https://ffmpeg.org/ffmpeg-formats.html
```

Lista de formatos soportados.

```
https://trac.ffmpeg.org/wiki
```

Wiki con ejemplos prácticos y guías de uso.

La documentación oficial está principalmente en inglés. Sin embargo, la wiki de FFmpeg contiene muchos ejemplos prácticos organizados por casos de uso, lo que facilita el aprendizaje.

**Observación de evaluación:**\
En documentación técnica conviene citar siempre la fuente oficial antes que blogs o tutoriales externos. Los tutoriales pueden ayudar, pero la referencia principal debe ser la documentación del proyecto.

***

## 4.3 Instalación de FFmpeg en los principales sistemas operativos

### 4.3.1 Instalación en Linux Ubuntu / Debian

En Ubuntu y Debian, FFmpeg está disponible directamente en los repositorios oficiales, por lo que la instalación es sencilla:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install ffmpeg -y
```

El requisito previo principal es tener conexión a internet y permisos de administrador mediante `sudo`.

### 4.3.2 Instalación en macOS

En macOS, FFmpeg se instala normalmente mediante Homebrew, que es un gestor de paquetes para macOS:

```bash
brew install ffmpeg
```

Si Homebrew no está instalado, primero debe instalarse desde:

```
https://brew.sh
```

### 4.3.3 Instalación en Windows

En Windows, la instalación suele ser manual porque el sistema no incluye un gestor de paquetes nativo tan directo como `apt`.

El proceso habitual es el siguiente:

1. Descargar el binario compilado desde una fuente fiable, como `gyan.dev` o los builds de BtbN en GitHub.
2. Extraer el archivo `.zip` en una carpeta, por ejemplo `C:\ffmpeg`.
3. Añadir la ruta `C:\ffmpeg\bin` a la variable de entorno `PATH`.
4. Abrir `cmd` o PowerShell y verificar la instalación con:

```powershell
ffmpeg -version
```

También es posible instalarlo con gestores como Winget:

```powershell
winget install ffmpeg
```

### 4.3.4 Requisitos previos generales

Para un uso básico no se necesita instalar nada adicional. FFmpeg ya incluye sus bibliotecas internas y soporte para muchos códecs.

Para funcionalidades avanzadas, como códecs específicos o soporte de ciertas tecnologías con restricciones de licencia, puede ser necesario compilar FFmpeg manualmente con opciones concretas. Sin embargo, para esta práctica el paquete estándar es suficiente.

***

## 4.4 Características fundamentales de FFmpeg

FFmpeg destaca por su versatilidad. Sus capacidades principales son las siguientes:

### 4.4.1 Soporte universal de formatos y códecs

FFmpeg soporta una gran cantidad de formatos de contenedor, como MP4, MKV, AVI, MOV, WebM, FLV o TS. También soporta numerosos códecs de audio y vídeo, como H.264, H.265/HEVC, AV1, VP9, AAC, MP3 u Opus.

Esto lo convierte en una herramienta compatible con prácticamente cualquier archivo multimedia utilizado en entornos reales.

### 4.4.2 Transcodificación de alto rendimiento

FFmpeg puede recodificar vídeo y audio de un códec a otro. Además, puede utilizar aceleración por hardware mediante tecnologías como NVIDIA NVENC, AMD AMF, Intel Quick Sync o VAAPI en Linux.

La aceleración por hardware permite que algunas conversiones sean mucho más rápidas que usando únicamente la CPU.

### 4.4.3 Streaming en tiempo real

FFmpeg permite recibir y emitir streams de red mediante protocolos como RTMP, HLS, RTSP y SRT. Puede actuar como encoder, receptor, conversor o repetidor de streaming.

### 4.4.4 Filtros y procesamiento avanzado

FFmpeg incluye un sistema de filtros de vídeo y audio. Los filtros de vídeo se aplican normalmente con `-vf`, mientras que los filtros de audio se aplican con `-af`.

Con estos filtros se puede escalar la resolución, recortar imagen, añadir subtítulos, reducir ruido, superponer imágenes o generar efectos.

### 4.4.5 Extracción y manipulación de pistas

FFmpeg permite separar, copiar, eliminar o transformar pistas de vídeo, audio y subtítulos. Por ejemplo, puede extraer solo el audio de un vídeo o eliminar una pista concreta.

### 4.4.6 Remuxing sin pérdida de calidad

Permite cambiar el contenedor de un archivo sin modificar los streams internos. Esto es muy rápido y no produce pérdida de calidad.

### 4.4.7 Automatización mediante scripts

Al funcionar desde línea de comandos, FFmpeg puede integrarse en scripts y procesos automatizados. Esto permite procesar muchos archivos de forma repetitiva y controlada.

**Observación de evaluación:**\
Una respuesta completa sobre FFmpeg debe mencionar tanto su capacidad de conversión como su arquitectura modular: códecs, contenedores, filtros, streams y automatización.

***

## 4.5 Cambio de contenedor sin recodificar: remuxing

Sí, es posible cambiar de contenedor sin recodificar el vídeo ni el audio. Esta operación se llama remuxing o remultiplexado.

### 4.5.1 Qué es el remuxing

Un archivo de vídeo tiene dos partes diferenciadas:

* El contenedor, también llamado formato o wrapper.
* El contenido real, formado por los streams de vídeo, audio y subtítulos.

El contenedor es el archivo externo, por ejemplo `.mp4`, `.mkv`, `.avi` o `.mov`. Define cómo se organizan los datos, cómo se sincronizan audio y vídeo y qué metadatos se guardan.

Los streams son los datos reales. Por ejemplo, el vídeo puede estar comprimido con H.264 y el audio con AAC.

El remuxing cambia únicamente el contenedor. El contenido interno no se modifica. Es similar a sacar una carta de un sobre y meterla en otro sobre diferente: el contenido de la carta sigue siendo el mismo.

### 4.5.2 Cómo hacer remuxing con FFmpeg

La clave está en usar `-c copy`, que indica a FFmpeg que copie los streams tal como están, sin recodificarlos:

```bash
ffmpeg -i video_original.mp4 -c copy video_convertido.mkv
```

Este proceso suele ser casi instantáneo y no produce pérdida de calidad.

### 4.5.3 Limitaciones del remuxing

No todas las combinaciones de contenedor y códec son compatibles. Por ejemplo, H.264 con audio AAC funciona bien dentro de MKV, pero algunos códecs no son compatibles con determinados contenedores.

Si se intenta hacer un remux con una combinación incompatible, FFmpeg mostrará un error.

**Observación de evaluación:**\
La palabra clave es `copy`. Si el comando contiene `-c copy`, `-vcodec copy` o `-acodec copy`, FFmpeg copia streams sin recodificar. Si aparece un códec concreto como `libx264`, entonces hay recodificación.

***

## 4.6 Características técnicas de MP4 y MKV

### 4.6.1 MP4, MPEG-4 Part 14

MP4 es uno de los formatos de vídeo más utilizados en el mundo del consumo. Fue definido dentro del ecosistema MPEG y está estandarizado como ISO/IEC 14496-12.

Sus características técnicas principales son:

* Compatible con códecs de vídeo como H.264, H.265/HEVC, AV1 y MPEG-4.
* Compatible con códecs de audio como AAC, MP3 y AC-3.
* Soporta subtítulos en formatos específicos como MP4TT.
* Permite reproducción progresiva si el archivo está optimizado con `faststart`.
* Tiene soporte nativo en la mayoría de smartphones, Smart TVs, consolas y navegadores.
* Es menos flexible que MKV para múltiples pistas de audio, vídeo o subtítulos.

MP4 conviene cuando el archivo se va a distribuir en internet, redes sociales, móviles, tablets, Smart TVs o navegadores. Es el formato ideal cuando se busca máxima compatibilidad.

### 4.6.2 MKV, Matroska

MKV es un contenedor de código abierto y libre de patentes. Está diseñado para ser flexible y admitir una gran variedad de códecs y pistas.

Sus características técnicas principales son:

* Soporta prácticamente cualquier códec de vídeo y audio.
* Permite múltiples pistas de vídeo, audio y subtítulos en un único archivo.
* Soporta subtítulos SRT, ASS/SSA y PGS.
* Permite capítulos y menús.
* Es muy utilizado en copias de Blu-ray y DVD.
* Puede ofrecer mejor recuperación ante errores en comparación con otros contenedores.
* Tiene menor compatibilidad que MP4 en algunos dispositivos antiguos.

MKV conviene cuando se quieren conservar múltiples pistas de audio, varios idiomas, subtítulos o capítulos. Es especialmente útil para uso personal, servidores multimedia como Plex o Jellyfin y archivado flexible.

**Observación de evaluación:**\
MP4 debe asociarse con compatibilidad. MKV debe asociarse con flexibilidad. Esa diferencia resume la elección técnica entre ambos contenedores.

***

## 4.7 Análisis del comando con libx264

El comando siguiente no solo cambia el contenedor, sino que también recodifica el vídeo:

```bash
ffmpeg -i metal-violin-f616.mp4 -vcodec libx264 metal-violin-f616.mkv
```

La razón es la opción `-vcodec libx264`.

El comando se puede interpretar así:

* `-i metal-violin-f616.mp4`: indica el archivo de entrada.
* `-vcodec libx264`: ordena usar el códec H.264 mediante la biblioteca libx264 para el vídeo de salida.
* `metal-violin-f616.mkv`: indica el archivo de salida y el contenedor MKV.

Al especificar `libx264`, FFmpeg decodifica el vídeo original y lo vuelve a comprimir usando H.264. Por tanto, hay transcodificación.

Si el vídeo original ya estuviera en H.264, se estaría recodificando innecesariamente de H.264 a H.264. Esto consumiría CPU y podría introducir pérdida de calidad por degradación generacional.

Si el objetivo fuera únicamente cambiar el contenedor de MP4 a MKV sin recodificar, el comando correcto sería:

```bash
ffmpeg -i metal-violin-f616.mp4 -c copy metal-violin-f616.mkv
```

**Regla de oro:**\
Si se usa `-vcodec nombre_del_codec` o `-c:v nombre_del_codec`, hay recodificación. Si se usa `-vcodec copy` o `-c:v copy`, no hay recodificación.

**Observación de evaluación:**\
Esta pregunta es muy probable en una práctica o examen porque permite comprobar si se entiende la diferencia entre contenedor y códec.

***

## 4.8 H.264 y H.265

### 4.8.1 H.264, AVC

H.264, también conocido como AVC o Advanced Video Coding, es uno de los estándares de compresión de vídeo más utilizados del mundo. Fue desarrollado conjuntamente por ITU-T y MPEG y publicado en 2003.

Sus características principales son:

* Ofrece buena relación entre calidad y tamaño frente a estándares anteriores como MPEG-2.
* Es el códec habitual en streaming, videollamadas y grabación de cámara en smartphones.
* Tiene soporte hardware muy amplio en procesadores, GPUs, Smart TVs y teléfonos.
* Permite decodificación eficiente, lo que reduce consumo de batería.
* Soporta resoluciones altas, incluido 4K.
* Puede codificarse mediante la biblioteca `libx264` o mediante encoders hardware.

### 4.8.2 H.265, HEVC

H.265, también conocido como HEVC o High Efficiency Video Coding, es el sucesor de H.264. Fue publicado en 2013.

Su objetivo principal es ofrecer una calidad visual similar a H.264 utilizando aproximadamente la mitad del bitrate o del espacio en disco.

### 4.8.3 Comparativa entre H.264 y H.265

| Característica                 | H.264 / AVC                               | H.265 / HEVC                                          |
| ------------------------------ | ----------------------------------------- | ----------------------------------------------------- |
| Año de publicación             | 2003                                      | 2013                                                  |
| Compresión                     | Referencia                                | Aproximadamente 40–50 % más eficiente                 |
| Calidad a mismo bitrate        | Buena                                     | Mejor                                                 |
| Compatibilidad hardware        | Universal                                 | Muy buena en hardware reciente                        |
| Compatibilidad software        | Universal                                 | Buena, aunque algunos navegadores tienen limitaciones |
| Licencias y patentes           | Patentado, ampliamente licenciado         | Sistema de patentes más complejo                      |
| Tiempo de codificación por CPU | Rápido                                    | Más lento                                             |
| Uso típico                     | Streaming estándar, web, videoconferencia | 4K, almacenamiento, Blu-ray 4K                        |

### 4.8.4 Cuándo usar cada uno

H.264 conviene cuando se busca máxima compatibilidad con todo tipo de dispositivos o cuando el hardware del sistema es limitado.

H.265 conviene cuando se desea ahorrar espacio o ancho de banda y los dispositivos de destino son modernos.

**Observación de evaluación:**\
H.264 no es el más eficiente, pero es el más compatible. H.265 es más eficiente, pero requiere más potencia de codificación y puede presentar más problemas de compatibilidad.

***

## 4.9 Comportamiento de FFmpeg si no se especifica el códec de audio

Cuando en un comando FFmpeg se especifica el códec de vídeo pero no se especifica nada sobre el audio, FFmpeg aplica su política automática para seleccionar o convertir streams.

El comportamiento puede variar según el caso, el formato de salida y los códecs compatibles.

### 4.9.1 Comportamiento por defecto

Si el códec de audio de entrada es compatible con el contenedor de salida, FFmpeg puede copiarlo o mantenerlo según la selección automática.

Si el códec original no es compatible con el contenedor de destino, FFmpeg puede recodificar automáticamente el audio al códec predeterminado del contenedor.

Cuando hay recodificación de vídeo activa, también puede producirse recodificación de audio si no se indica lo contrario.

### 4.9.2 Caso concreto del ejercicio

En el comando:

```bash
ffmpeg -i metal-violin-f616.mp4 -vcodec libx264 metal-violin-f616.mkv
```

El vídeo se recodifica con `libx264`. El audio no tiene una instrucción explícita, por lo que FFmpeg decide automáticamente qué hacer.

En la práctica, si el audio original es compatible con MKV, puede conservarlo o copiarlo. Si no lo es, puede recodificarlo a otro códec compatible.

### 4.9.3 Cómo controlar explícitamente el audio

Para evitar sorpresas, conviene especificar siempre el comportamiento del audio.

Copiar el audio sin modificarlo:

```bash
ffmpeg -i archivo.mp4 -vcodec libx264 -acodec copy salida.mkv
```

Recodificar el audio a AAC con bitrate de 192 kbps:

```bash
ffmpeg -i archivo.mp4 -vcodec libx264 -acodec aac -b:a 192k salida.mkv
```

**Observación de evaluación:**\
En documentación técnica es mejor no depender de comportamientos automáticos si se quiere reproducibilidad. Es recomendable indicar explícitamente los códecs de vídeo y audio.

***

## 4.10 Qué es yt-dlp y diferencias con youtube-dl

yt-dlp es un descargador de vídeo de código abierto para línea de comandos. Permite descargar vídeos, listas de reproducción y streams desde una gran cantidad de sitios web, como YouTube, Vimeo, Twitch, TikTok, Instagram o Dailymotion.

Es un fork de youtube-dlc, que a su vez era un fork de youtube-dl. Surgió cuando youtube-dl empezó a tener problemas de mantenimiento y actualizaciones poco frecuentes.

### 4.10.1 Diferencias principales

| Característica            | youtube-dl                                   | yt-dlp                                                           |
| ------------------------- | -------------------------------------------- | ---------------------------------------------------------------- |
| Estado del proyecto       | Mantenimiento lento o prácticamente inactivo | Activo, con actualizaciones frecuentes                           |
| Velocidad de descarga     | Estándar                                     | Mejorada mediante fragmentación paralela y herramientas externas |
| Formatos y calidad        | Selección básica                             | Selección avanzada mediante filtros complejos                    |
| Integración con FFmpeg    | Básica                                       | Avanzada, con merge, postprocesamiento y capítulos               |
| SponsorBlock              | No                                           | Sí                                                               |
| Cookies                   | Limitado                                     | Más avanzado, con importación desde navegador                    |
| Capítulos                 | Limitado                                     | Completo                                                         |
| Compatibilidad con sitios | Menor                                        | Mayor cantidad de extractores actualizados                       |
| Configuración             | Principalmente flags CLI                     | CLI y fichero de configuración                                   |

En resumen, yt-dlp puede considerarse una evolución más activa, rápida y completa de youtube-dl.

**Observación de evaluación:**\
yt-dlp no sustituye a FFmpeg. yt-dlp descarga y selecciona formatos; FFmpeg procesa, convierte y fusiona streams.

***

## 4.11 Instalación de yt-dlp en distintos sistemas operativos

### 4.11.1 Linux Ubuntu / Debian

El método recomendado es descargar el binario directamente desde GitHub para obtener una versión actualizada:

```bash
sudo wget https://github.com/yt-dlp/yt-dlp/releases/latest/download/yt-dlp -O /usr/local/bin/yt-dlp
sudo chmod a+rx /usr/local/bin/yt-dlp
```

### 4.11.2 Dependencias importantes

yt-dlp requiere Python 3.9 o superior. Ubuntu 22.04 ya incluye Python 3.10 por defecto.

También es recomendable tener FFmpeg instalado, ya que yt-dlp lo utiliza internamente para fusionar vídeo y audio cuando los descarga por separado.

```bash
sudo apt install ffmpeg -y
```

### 4.11.3 macOS

En macOS puede instalarse mediante Homebrew:

```bash
brew install yt-dlp
```

También puede instalarse mediante pip:

```bash
python3 -m pip install -U yt-dlp
```

### 4.11.4 Windows

En Windows se puede descargar el ejecutable `yt-dlp.exe` desde GitHub, colocarlo en una carpeta como `C:\yt-dlp\` y añadir esa ruta al `PATH`.

También puede instalarse con Winget:

```powershell
winget install yt-dlp
```

***

## 4.12 Características fundamentales de yt-dlp

yt-dlp tiene muchas funcionalidades. Las más importantes son las siguientes.

### 4.12.1 Selección de formato y calidad

Permite elegir exactamente qué calidad de vídeo y audio se quiere descargar. Se pueden usar filtros para seleccionar resolución, códec, extensión o bitrate.

Por ejemplo, se puede descargar solo vídeo en 1080p con códec AV1 y combinarlo con el mejor audio disponible.

### 4.12.2 Subtítulos

Puede descargar subtítulos automáticos y manuales en varios idiomas y formatos, como `.vtt`, `.srt` o `.ass`. También puede incrustarlos en el archivo final mediante FFmpeg.

### 4.12.3 Metadatos

Puede escribir metadatos del vídeo, como título, descripción, canal, fecha de subida o miniatura, dentro del archivo descargado o en ficheros separados.

### 4.12.4 Integración con FFmpeg

yt-dlp usa FFmpeg para:

* Fusionar vídeo y audio descargados por separado.
* Convertir archivos a otros formatos.
* Extraer audio en MP3 u otros formatos.
* Insertar metadatos, capítulos y subtítulos.

### 4.12.5 Descarga de listas de reproducción y canales

Puede descargar playlists completas o canales enteros, con control sobre orden, límites y archivos ya descargados mediante `--download-archive`.

### 4.12.6 SponsorBlock

Integra soporte para SponsorBlock, lo que permite eliminar segmentos de patrocinio, intros u outros en vídeos de YouTube.

### 4.12.7 Cookies del navegador

Permite importar cookies desde navegadores como Chrome o Firefox para descargar contenido restringido por edad, login o región.

**Observación de evaluación:**\
La integración con FFmpeg es una de las características más importantes de yt-dlp, porque muchas descargas modernas separan vídeo y audio en streams diferentes.

***

## 4.13 Comparativa de códecs disponibles en yt-dlp: AV1, VP9 y H.264

| Característica            | H.264 / AVC           | VP9                           | AV1                                                       |
| ------------------------- | --------------------- | ----------------------------- | --------------------------------------------------------- |
| Desarrollador             | ITU-T / ISO MPEG      | Google                        | Alliance for Open Media                                   |
| Año de publicación        | 2003                  | 2013                          | 2018                                                      |
| Patentes / licencias      | Patentado             | Libre de royalties            | Libre de royalties                                        |
| Eficiencia de compresión  | Referencia            | 30–40 % mejor que H.264       | Aproximadamente 30 % mejor que VP9 y 50 % mejor que H.264 |
| Calidad a mismo bitrate   | Buena                 | Mejor que H.264               | La mejor de las tres                                      |
| Velocidad de codificación | Muy rápida            | Moderada                      | Lenta, aunque mejora con hardware                         |
| Compatibilidad            | Universal             | Buena en navegadores modernos | Creciente, especialmente en hardware reciente             |
| Uso recomendado           | Máxima compatibilidad | Web de calidad media/alta     | Alta eficiencia y almacenamiento a largo plazo            |

YouTube puede servir AV1 a clientes compatibles, VP9 como alternativa y H.264 para dispositivos antiguos o situaciones donde se requiere máxima compatibilidad.

Comandos para forzar un códec concreto en yt-dlp:

```bash
# Descargar solo con AV1
yt-dlp -f "bestvideo[vcodec^=av01]+bestaudio" URL

# Descargar solo con VP9
yt-dlp -f "bestvideo[vcodec^=vp9]+bestaudio" URL

# Descargar solo con H.264
yt-dlp -f "bestvideo[vcodec^=avc1]+bestaudio" URL
```

**Observación de evaluación:**\
La elección del códec depende del equilibrio entre calidad, tamaño, velocidad de codificación y compatibilidad. No existe un único códec mejor para todos los casos.

***

## 4.14 Alternativas a FFmpeg y yt-dlp

### 4.14.1 Alternativas a FFmpeg

#### HandBrake

HandBrake es un transcodificador de vídeo de código abierto con interfaz gráfica y modo CLI.

Ventajas:

* Muy fácil de usar para usuarios no técnicos.
* Incluye presets para dispositivos como iPhone, Android o TV.
* Tiene buen soporte de aceleración hardware.

Inconvenientes:

* Es mucho más limitado que FFmpeg.
* No soporta tantos formatos y códecs.
* No está orientado a streaming avanzado.
* No es tan flexible para automatización.

Uso recomendado:

* Conversión puntual de vídeos sin necesidad de dominar la línea de comandos.

#### VLC Media Player

VLC es principalmente un reproductor multimedia, pero también permite conversión de vídeo desde interfaz gráfica o mediante CLI con `cvlc`.

Ventajas:

* Está instalado en muchos sistemas.
* Tiene interfaz gráfica.
* Puede trabajar con streaming.

Inconvenientes:

* Es menos potente y flexible que FFmpeg.
* La interfaz de conversión puede resultar poco intuitiva.
* No es ideal para uso profesional o automatizado.

#### GStreamer

GStreamer es un framework multimedia basado en plugins. Es muy utilizado en Linux, aplicaciones embebidas y desarrollo multimedia.

Ventajas:

* Muy modular.
* Potente para desarrolladores.
* Adecuado para sistemas embebidos y pipelines personalizados.

Inconvenientes:

* Tiene una curva de aprendizaje elevada.
* No está pensado como herramienta simple de usuario final.

Uso recomendado:

* Desarrollo de aplicaciones multimedia, sistemas embebidos e integración de pipelines en software.

### 4.14.2 Alternativas a yt-dlp

#### gallery-dl

gallery-dl es un descargador especializado en imágenes y galerías de sitios como Instagram, Flickr, Pixiv o Reddit.

Ventajas:

* Mejor que yt-dlp para sitios centrados en imágenes.
* Proyecto activo.

Inconvenientes:

* No está orientado a descargar vídeos de plataformas como YouTube.

#### Cobalt.tools

Cobalt.tools es una herramienta web que permite descargar vídeos desde el navegador, sin instalación local.

Ventajas:

* Fácil de usar.
* No requiere instalación.
* Funciona desde navegador.

Inconvenientes:

* Menor control sobre formatos.
* No permite la misma precisión que yt-dlp.
* Depende de la disponibilidad del servicio web.

#### Stacher

Stacher es una interfaz gráfica para yt-dlp. No es una alternativa completa, sino una forma visual de usar yt-dlp.

Ventajas:

* Más cómodo para usuarios sin experiencia en terminal.
* Disponible para Windows, macOS y Linux.

Inconvenientes:

* Menor control y flexibilidad que usar yt-dlp directamente desde CLI.

**Observación de evaluación:**\
FFmpeg es más potente que sus alternativas porque funciona como una caja de herramientas completa. HandBrake o VLC son más fáciles, pero mucho menos flexibles.

***

## 5. Parte práctica: instalación y uso básico de FFmpeg

## 5.1 Paso 1: actualizar el sistema

Antes de instalar cualquier herramienta, es buena práctica actualizar la información de los repositorios y los paquetes instalados. Esto evita conflictos de versiones y garantiza que el sistema utiliza las versiones disponibles más recientes en los repositorios configurados.

```bash
sudo apt update && sudo apt upgrade -y
```

Output esperado:

```
Hit:1 http://archive.ubuntu.com/ubuntu jammy InRelease
Get:2 http://archive.ubuntu.com/ubuntu jammy-updates InRelease [119 kB]
...
Reading package lists... Done
Building dependency tree... Done
...
0 upgraded, 0 newly installed, 0 to remove and 0 not changed.
```

El mensaje final puede variar. Si hay actualizaciones disponibles, el sistema mostrará cuántos paquetes se actualizarán. Si el sistema ya está actualizado, puede aparecer `0 upgraded`.

### 5.1.1 Posible error

```
E: Could not get lock /var/lib/dpkg/lock-frontend
```

Este error indica que otro proceso está usando `apt`, por ejemplo una actualización automática. La solución habitual es esperar unos minutos y volver a intentarlo. También se puede reiniciar la máquina virtual si el bloqueo persiste.

**Observación de evaluación:**\
Actualizar antes de instalar muestra una metodología correcta de administración de sistemas.

***

## 5.2 Paso 2: instalar FFmpeg

```bash
sudo apt install ffmpeg -y
```

Output esperado:

```
Reading package lists... Done
Building dependency tree... Done
The following additional packages will be installed:
  ffmpeg libavcodec58 libavdevice58 libavfilter7 ...
...
Setting up ffmpeg (7:4.4.2-0ubuntu0.22.04.1) amd64 ...
Processing triggers for man-db (2.10.2-1) ...
```

### 5.2.1 Posible error

```
E: Package 'ffmpeg' has no installation candidate
```

Este error puede aparecer si el repositorio `universe` no está activado. Para solucionarlo:

```bash
sudo add-apt-repository universe
sudo apt update
sudo apt install ffmpeg -y
```

### 5.2.2 Screenshot recomendado

Captura la terminal mostrando el proceso de instalación y el mensaje final `Setting up ffmpeg`.

Esta captura demuestra que FFmpeg se ha instalado correctamente junto con sus dependencias, como `libavcodec`, `libavfilter`y otras bibliotecas que contienen los códecs y filtros que FFmpeg puede utilizar.

***

## 5.3 Paso 3: verificar la instalación

```bash
ffmpeg -version
```

Output esperado:

```
ffmpeg version 4.4.2-0ubuntu0.22.04.1 Copyright (c) 2000-2021 the FFmpeg developers
built with gcc 11 (Ubuntu 11.2.0-19ubuntu1)
configuration: --prefix=/usr --extra-version=0ubuntu0.22.04.1 ...
libavutil      56. 70.100 / 56. 70.100
libavcodec     58.134.100 / 58.134.100
libavformat    58. 76.100 / 58. 76.100
libavdevice    58. 13.100 / 58. 13.100
libavfilter     7.110.100 /  7.110.100
libswscale      5.  9.100 /  5.  9.100
libswresample   3.  9.100 /  3.  9.100
libpostproc    55.  9.100 / 55.  9.100
```

Este output confirma que FFmpeg está instalado y funcional. La información de las librerías muestra los módulos disponibles:

* `libavcodec`: gestión de códecs.
* `libavformat`: gestión de contenedores y formatos.
* `libavfilter`: filtros de vídeo y audio.
* `libswscale`: escalado y conversión de formatos de píxel.
* `libswresample`: remuestreo de audio.

### 5.3.1 Screenshot recomendado

Captura el output completo del comando:

```bash
ffmpeg -version
```

**Observación de evaluación:**\
No basta con decir que FFmpeg está instalado. Es mejor explicar qué significan las librerías mostradas en el output.

***

## 5.4 Paso 4: explorar la ayuda de FFmpeg

```bash
ffmpeg -h
```

Output esperado:

```
ffmpeg version 4.4.2 Copyright (c) 2000-2021 the FFmpeg developers
usage: ffmpeg [options] [[infile options] -i infile]... {[outfile options] outfile}...

Getting help:
    -h      -- print basic options
    -h long -- print more options
    -h full -- print all options (including all format and codec specific options)
...
```

El comando `ffmpeg -h` muestra las opciones básicas. Para ver opciones más avanzadas se pueden usar:

```bash
ffmpeg -h long
ffmpeg -h full
```

Para consultar las opciones específicas de un encoder concreto, por ejemplo `libx264`, se puede usar:

```bash
ffmpeg -h encoder=libx264
```

**Observación de evaluación:**\
Consultar la ayuda demuestra capacidad de autonomía. En entornos reales no se memorizan todos los parámetros; se sabe consultar la documentación y la ayuda del propio comando.

***

## 5.5 Paso 5: obtener un vídeo de prueba

Para trabajar con FFmpeg se necesita un archivo de vídeo. Se puede descargar uno de muestra con `wget`:

```bash
wget https://www.learningcontainer.com/wp-content/uploads/2020/05/sample-mp4-file.mp4 -O coldplay.mp4
```

También se puede utilizar un vídeo propio.

Si ya está instalado yt-dlp, se puede descargar un vídeo con una calidad limitada para no trabajar con archivos demasiado grandes:

```bash
yt-dlp -f "bestvideo[height<=480]+bestaudio" "https://www.youtube.com/watch?v=zrnCBt2q-dY" -o coldplay.mp4
```

**Observación de evaluación:**\
Es recomendable utilizar un vídeo corto o de resolución moderada para que las conversiones no tarden demasiado dentro de la máquina virtual.

***

## 5.6 Paso 6: ver información detallada de un vídeo

El flag `-i`, seguido de un archivo, permite que FFmpeg muestre información técnica del archivo multimedia:

```bash
ffmpeg -i coldplay.mp4
```

Output esperado:

```
ffmpeg version 4.4.2 ...
Input #0, mov,mp4,m4a,3gp,3g2,mj2, from 'coldplay.mp4':
  Metadata:
    major_brand     : isom
    minor_version   : 512
    compatible_brands: isomiso2avc1mp41
    encoder         : Lavf58.76.100
  Duration: 00:03:52.47, start: 0.000000, bitrate: 2000 kb/s
    Stream #0:0(und): Video: h264 (High), yuv420p, 1280x720, 1870 kb/s, 25 fps, 25 tbr, 12800 tbn, 50 tbc (default)
    Stream #0:1(und): Audio: aac (LC), 44100 Hz, stereo, fltp, 128 kb/s (default)
```

Este output muestra:

* El formato contenedor del archivo.
* La duración total.
* El bitrate aproximado.
* El códec de vídeo.
* La resolución.
* Los FPS.
* El códec de audio.
* La frecuencia de muestreo del audio.
* Si el audio es mono o estéreo.

### 5.6.1 Screenshot recomendado

Captura el output completo de:

```bash
ffmpeg -i coldplay.mp4
```

Esta captura funciona como ficha técnica del vídeo.

### 5.6.2 Posible error

```
coldplay.mp4: No such file or directory
```

Este error indica que el archivo no existe en el directorio actual. Para comprobarlo:

```bash
ls
pwd
```

`ls` muestra los archivos del directorio actual y `pwd` indica en qué ruta se está trabajando.

***

## 5.7 Paso 7: convertir vídeo a MKV recodificando con H.264 y H.265

### 5.7.1 Conversión con H.264

```bash
ffmpeg -i coldplay.mp4 -vcodec libx264 coldplay_264.mkv
```

Output esperado:

```
frame=  100 fps= 45 q=23.0 size=    2048kB time=00:00:04.00 bitrate=4194.3kbits/s speed=1.8x
...
video:50000kB audio:3000kB subtitle:0kB other streams:0kB global headers:0kB muxing overhead: 0.050%
```

El output en tiempo real muestra:

* `frame`: número de fotogramas procesados.
* `fps`: velocidad de procesamiento.
* `size`: tamaño parcial del archivo generado.
* `time`: tiempo procesado del vídeo.
* `bitrate`: tasa de bits aproximada.
* `speed`: velocidad relativa respecto al tiempo real.

Si `speed=1.8x`, significa que FFmpeg está convirtiendo el vídeo 1,8 veces más rápido que la duración real del vídeo.

### 5.7.2 Screenshot recomendado para H.264

Captura la terminal mientras procesa y también el resumen final.

### 5.7.3 Conversión con H.265

```bash
ffmpeg -i coldplay.mp4 -vcodec libx265 coldplay_265.mkv
```

H.265 suele producir archivos más pequeños que H.264 manteniendo una calidad visual similar, pero el proceso de codificación suele ser más lento.

La velocidad `speed` puede ser menor, por ejemplo `0.3x` o `0.5x`, porque H.265 realiza cálculos más complejos para mejorar la compresión.

### 5.7.4 Screenshot recomendado para H.265

Captura el output del proceso de codificación con H.265 y observa especialmente la velocidad de procesamiento.

### 5.7.5 Comparar tamaños

```bash
ls -lh coldplay.mp4 coldplay_264.mkv coldplay_265.mkv
```

Output esperado:

```
-rw-r--r-- 1 user user  58M May 10 10:00 coldplay.mp4
-rw-r--r-- 1 user user  45M May 10 10:02 coldplay_264.mkv
-rw-r--r-- 1 user user  28M May 10 10:08 coldplay_265.mkv
```

Esta comparación permite observar que H.265 puede generar un archivo más pequeño que H.264 con una calidad visual similar.

### 5.7.6 Screenshot recomendado para tamaños

Captura el output de:

```bash
ls -lh coldplay.mp4 coldplay_264.mkv coldplay_265.mkv
```

### 5.7.7 Posible error con H.265

```
Unknown encoder 'libx265'
```

Este error indica que la versión de FFmpeg instalada no incluye soporte para `libx265`.

Posible solución:

```bash
sudo apt install ffmpeg libx265-dev -y
```

Si el error persiste, puede que el binario de FFmpeg no esté compilado con soporte para `libx265`.

**Observación de evaluación:**\
H.265 no siempre estará disponible según cómo se haya compilado FFmpeg. Esto muestra que no todos los binarios de FFmpeg tienen exactamente las mismas capacidades.

***

## 5.8 Paso 8: modificar el códec de audio

Los siguientes comandos convierten el archivo manteniendo el vídeo original sin recodificar, gracias a `-vcodec copy`, pero cambiando el códec de audio:

```bash
ffmpeg -i coldplay.mp4 -vcodec copy -acodec mp3 coldplay_mp3.mkv
ffmpeg -i coldplay.mp4 -vcodec copy -acodec aac coldplay_aac.mkv
ffmpeg -i coldplay.mp4 -vcodec copy -acodec libvorbis coldplay_vorbis.mkv
```

El flag `-vcodec copy` indica que FFmpeg copia el stream de vídeo sin modificarlo. Por tanto, solo se procesa el audio.

Estos comandos son más rápidos que una conversión de vídeo porque no se recodifican los fotogramas.

### 5.8.1 Comparativa de códecs de audio

| Códec  | Calidad                        | Compatibilidad                      | Tipo                       |
| ------ | ------------------------------ | ----------------------------------- | -------------------------- |
| MP3    | Buena con bitrate alto         | Universal                           | Patentes expiradas en 2017 |
| AAC    | Mejor que MP3 al mismo bitrate | Muy alta, estándar en iOS y YouTube | Patentado                  |
| Vorbis | Comparable a AAC               | Buena en web moderna                | Libre de patentes          |

### 5.8.2 Comparar tamaños

```bash
ls -lh coldplay_mp3.mkv coldplay_aac.mkv coldplay_vorbis.mkv
```

### 5.8.3 Screenshot recomendado

Captura la ejecución de los tres comandos y el output final. También es recomendable capturar la comparación de tamaños.

**Observación de evaluación:**\
Este paso demuestra que vídeo y audio son streams independientes. Es posible copiar uno y recodificar el otro.

***

## 5.9 Paso 9: personalizar el bitrate de vídeo y audio

```bash
ffmpeg -i coldplay.mp4 -b:v 2500k -b:a 192k coldplay_custom.mp4
```

Donde:

* `-b:v 2500k` define el bitrate de vídeo en 2500 kilobits por segundo.
* `-b:a 192k` define el bitrate de audio en 192 kilobits por segundo.

Output esperado:

```
frame=  ...  bitrate=2691.2kbits/s speed=2.5x
...
```

El bitrate determina cuántos datos por segundo se utilizan para representar el contenido. Un bitrate alto suele producir mejor calidad y mayor tamaño de archivo. Un bitrate bajo reduce el tamaño, pero puede degradar la calidad.

### 5.9.1 Screenshot recomendado

Captura el output durante la codificación y el mensaje final.

**Observación de evaluación:**\
El bitrate es una de las variables más importantes en compresión multimedia porque representa el equilibrio entre calidad, tamaño y ancho de banda.

***

## 5.10 Paso 10: extraer audio de un vídeo en MP3

```bash
ffmpeg -i coldplay.mp4 -vn coldplay_audio.mp3
```

El parámetro `-vn` significa “no video”. Indica que el archivo de salida no debe incluir ningún stream de vídeo.

Output esperado:

```
Stream mapping:
  Stream #0:1 -> #0:0 (aac (native) -> mp3 (libmp3lame))
Press [q] to quit, [?] for help
size=    3456kB time=00:03:52.00 bitrate= 121.8kbits/s speed=  62x
...
```

La extracción de audio suele ser mucho más rápida que la conversión de vídeo porque el audio requiere menos cálculo que procesar fotogramas.

### 5.10.1 Verificar que el archivo no tiene vídeo

```bash
ffmpeg -i coldplay_audio.mp3
```

El output debe mostrar únicamente un stream de audio, por ejemplo:

```
Stream #0:0: Audio:
```

No debe aparecer ningún stream de vídeo.

### 5.10.2 Screenshot recomendado

Captura el output de la extracción y, si es posible, también la verificación posterior.

***

## 5.11 Paso 11: recortar un fragmento de vídeo

### 5.11.1 Método 1: inicio y duración

Desde el segundo 35 con una duración de 30 segundos:

```bash
ffmpeg -i coldplay.mp4 -ss 35 -t 30 coldplay_frag.mp4
```

### 5.11.2 Método 2: inicio y final

Desde el segundo 35 hasta el segundo 65:

```bash
ffmpeg -i coldplay.mp4 -ss 00:00:35 -to 00:01:05 coldplay_frag.mp4
```

Donde:

* `-ss 35` indica el punto de inicio.
* `-t 30` indica la duración del fragmento.
* `-to 00:01:05` indica el punto final.

### 5.11.3 Verificar duración

```bash
ffprobe coldplay_frag.mp4
```

También se puede abrir el archivo en un reproductor para comprobar visualmente que el fragmento corresponde al intervalo elegido.

### 5.11.4 Screenshot recomendado

Captura el output del comando de recorte y, después, la verificación de duración con `ffprobe`.

### 5.11.5 Precisión de `-ss`

Colocar `-ss` antes de `-i` suele ser más rápido, porque FFmpeg busca el punto de inicio antes de decodificar. Sin embargo, puede ser menos preciso.

Colocar `-ss` después de `-i` suele ser más preciso, pero puede ser más lento, porque FFmpeg analiza más contenido antes de cortar.

**Observación de evaluación:**\
Este detalle demuestra comprensión avanzada del funcionamiento interno de FFmpeg.

***

## 6. Parte práctica: instalación y uso básico de yt-dlp

## 6.1 Paso 1: verificar Python

yt-dlp requiere Python 3.9 o superior. Para comprobar la versión instalada:

```bash
python3 --version
```

Output esperado:

```
Python 3.10.12
```

### 6.1.1 Posible error

```
bash: python3: command not found
```

Solución:

```bash
sudo apt install python3 -y
```

**Observación de evaluación:**\
Verificar dependencias antes de instalar evita errores posteriores y forma parte de una metodología correcta.

***

## 6.2 Paso 2: instalar yt-dlp

```bash
sudo wget https://github.com/yt-dlp/yt-dlp/releases/latest/download/yt-dlp -O /usr/local/bin/yt-dlp
```

Output esperado:

```
--2024-05-10 10:00:00--  https://github.com/yt-dlp/yt-dlp/releases/latest/download/yt-dlp
Resolving github.com... 140.82.121.4
Connecting to github.com|140.82.121.4|:443... connected.
HTTP request sent, awaiting response... 302 Found
Location: https://objects.githubusercontent.com/github-production-release-asset-... [following]
...
2024-05-10 10:00:05 (3.45 MB/s) - '/usr/local/bin/yt-dlp' saved [12345678/12345678]
```

El binario se descarga desde el repositorio oficial de GitHub y se guarda en `/usr/local/bin/`, que es un directorio estándar para ejecutables disponibles globalmente en el sistema.

### 6.2.1 Screenshot recomendado

Captura el output de `wget` mostrando el mensaje final de descarga completada.

### 6.2.2 Posible error

```
wget: unable to resolve host address 'github.com'
```

Este error indica que no hay resolución DNS o conexión a internet. En UTM conviene revisar que la red de la VM esté en modo Shared Network o Bridged.

***

## 6.3 Paso 3: dar permisos de ejecución

```bash
sudo chmod a+rx /usr/local/bin/yt-dlp
```

En Linux, si un comando se ejecuta correctamente y no tiene nada que mostrar, puede no producir output.

El comando `chmod a+rx` da permisos de lectura y ejecución a todos los usuarios del sistema.

Para verificar permisos:

```bash
ls -l /usr/local/bin/yt-dlp
```

### 6.3.1 Screenshot recomendado

Captura el comando `chmod` y, preferiblemente, el resultado de:

```bash
ls -l /usr/local/bin/yt-dlp
```

**Observación de evaluación:**\
El binario descargado no sirve si no tiene permiso de ejecución. Este paso demuestra conocimiento básico de permisos en Linux.

***

## 6.4 Paso 4: verificar la instalación

```bash
yt-dlp --version
```

Output esperado:

```
2024.04.09
```

La versión puede variar según la fecha de instalación.

***

## 6.5 Paso 5: consultar la ayuda

```bash
yt-dlp -h
```

Output esperado:

```
Usage: yt-dlp [OPTIONS] URL [URL...]

Options:
  General Options:
    -h, --help                      Print this help text and exit
    --version                       Print program version and exit
    -U, --update                    Update this program to the latest version
...
```

La lista de opciones de yt-dlp es extensa. Se puede filtrar con `grep`:

```bash
yt-dlp -h | grep "format"
```

O, si el sistema está en español y se busca por otra palabra:

```bash
yt-dlp -h | grep "audio"
```

**Observación de evaluación:**\
yt-dlp tiene muchas opciones. Saber filtrar la ayuda con `grep` es útil para encontrar parámetros concretos rápidamente.

***

## 6.6 Paso 6: ver formatos disponibles de un vídeo

Antes de descargar un vídeo, se pueden listar todos los formatos y calidades disponibles:

```bash
yt-dlp -F "https://www.youtube.com/watch?v=zrnCBt2q-dY"
```

Output esperado:

```
[youtube] Extracting URL: https://www.youtube.com/watch?v=zrnCBt2q-dY
[youtube] zrnCBt2q-dY: Downloading webpage
[youtube] zrnCBt2q-dY: Downloading ios player API JSON
[youtube] zrnCBt2q-dY: Downloading m3u8 information
[info] Available formats for zrnCBt2q-dY:
ID    EXT   RESOLUTION FPS │  FILESIZE   TBR PROTO │ VCODEC          VBR ACODEC      ABR ASR MORE INFO
──────────────────────────────────────────────────────────────────────────────────────────────────
sb3   mhtml 48x27          │                 mhtml │ images                                   storyboard
...
251   webm  audio only     │   3.50MiB   65k https │ audio only          opus        65k 48k medium, DRC
140   m4a   audio only     │   5.45MiB  128k https │ audio only          mp4a.40.2  128k 44k medium
...
160   mp4   256x144     25 │   2.10MiB   36k https │ avc1.4d400b     36k video only          144p
...
137   mp4   1920x1080   25 │  78.55MiB 1882k https │ avc1.4d4028   1882k video only          1080p
...
248   webm  1920x1080   25 │  51.43MiB 1234k https │ vp9            1234k video only          1080p
...
```

La tabla muestra:

* ID del formato.
* Extensión.
* Resolución.
* FPS.
* Tamaño aproximado.
* Protocolo.
* Códec de vídeo.
* Códec de audio.
* Bitrate.

En YouTube, los formatos de alta resolución suelen estar separados en vídeo only y audio only. Por eso yt-dlp descarga ambos streams y después usa FFmpeg para fusionarlos.

### 6.6.1 Screenshot recomendado

Captura la tabla completa de formatos disponibles.

### 6.6.2 Posible error

```
ERROR: [youtube] zrnCBt2q-dY: Sign in to confirm your age
```

Este error aparece cuando el vídeo tiene restricción de edad. Para solucionarlo puede ser necesario usar cookies del navegador con sesión iniciada mediante opciones como `--cookies-from-browser`.

**Observación de evaluación:**\
Saber leer la tabla de formatos es más importante que descargar automáticamente. Permite justificar técnicamente la calidad y el códec elegido.

***

## 6.7 Paso 7: descargar un formato específico por ID

Una vez conocidos los IDs de formato, se puede descargar uno concreto:

```bash
yt-dlp -f 137 "https://www.youtube.com/watch?v=zrnCBt2q-dY"
```

El ID `137` debe sustituirse por el formato que se quiera descargar según la tabla anterior.

Output esperado:

```
[youtube] Extracting URL: https://www.youtube.com/watch?v=zrnCBt2q-dY
[youtube] zrnCBt2q-dY: Downloading webpage
...
[download]  100% of   78.55MiB in 00:00:20 at 3.85MiB/s
```

**Observación de evaluación:**\
Descargar por ID demuestra control manual sobre el formato. Sin embargo, si el formato elegido es solo vídeo, puede descargarse sin audio.

***

## 6.8 Paso 8: descargar la mejor calidad disponible, vídeo y audio

```bash
yt-dlp -f "bv*+ba" "https://www.youtube.com/watch?v=bH3NFlkui4Y"
```

Donde:

* `bv*` significa best video, el mejor vídeo disponible.
* `ba` significa best audio, el mejor audio disponible.
* El símbolo `+` indica que deben combinarse ambos streams.

Output esperado:

```
[youtube] bH3NFlkui4Y: Downloading webpage
...
[download] Destination: AC_DC - ...webm
[download] 100% of 248.34MiB at 5.21MiB/s
...
[ffmpeg] Merging formats into "AC_DC - ....mkv"
Deleting original file ...webm (pass -k to keep)
```

Aquí se observa la integración entre yt-dlp y FFmpeg. yt-dlp descarga el mejor vídeo y el mejor audio por separado, y después llama a FFmpeg para fusionarlos en un único archivo.

Sin FFmpeg instalado, este paso no podría completarse correctamente.

### 6.8.1 Screenshot recomendado

Captura el proceso de descarga y especialmente la parte donde aparece:

```
[ffmpeg] Merging formats into ...
```

**Observación de evaluación:**\
Esta es una de las capturas más importantes porque demuestra que yt-dlp y FFmpeg trabajan conjuntamente.

***

## 6.9 Paso 9: otros comandos útiles de yt-dlp

### 6.9.1 Descargar con nombre y extensión personalizados

```bash
yt-dlp -o "%(title)s.%(ext)s" "https://www.youtube.com/watch?v=zrnCBt2q-dY"
```

La opción `-o` permite personalizar el nombre del archivo de salida. En este caso, el archivo se guardará usando el título del vídeo y su extensión correspondiente.

### 6.9.2 Descargar solo audio en MP3

```bash
yt-dlp -x --audio-format mp3 "https://www.youtube.com/watch?v=zrnCBt2q-dY"
```

El flag `-x` indica que se debe extraer solo el audio. La opción `--audio-format mp3` define el formato final del audio.

Internamente, yt-dlp usa FFmpeg para convertir el audio al formato solicitado.

### 6.9.3 Screenshot recomendado

Captura la descarga con `-x` mostrando la conversión a MP3.

### 6.9.4 Descargar calidad específica, por ejemplo 1080p

```bash
yt-dlp -f "bv*[height=1080]+ba" "https://www.youtube.com/watch?v=zrnCBt2q-dY"
```

Este comando selecciona el mejor vídeo con altura de 1080 píxeles y lo combina con el mejor audio disponible.

### 6.9.5 Descargar una playlist completa

```bash
yt-dlp "https://www.youtube.com/playlist?list=PLxxxxxx"
```

**Observación de evaluación:**\
La potencia de yt-dlp está en la selección de formato. No se limita a descargar “lo mejor”, sino que permite definir criterios técnicos exactos.

***

## 7. Ejemplo propio: FFmpeg + yt-dlp

### 7.1 Escenario

Eres administrador de sistemas de una pequeña radio online. Quieres descargar la versión oficial del último concierto de una banda desde YouTube, extraer solo el audio en alta calidad y dividirlo en fragmentos de 10 minutos para distribuirlos como podcast.

Este caso combina las dos herramientas:

* yt-dlp se encarga de descargar el contenido.
* FFmpeg se encarga de procesar y dividir el audio.

### 7.2 Paso 1: descargar el concierto con la mejor calidad de audio

```bash
yt-dlp -f "bestaudio" -o "concierto_original.%(ext)s" "https://www.youtube.com/watch?v=XXXXXXXXX"
```

Este comando descarga únicamente la mejor pista de audio disponible y la guarda con el nombre `concierto_original`manteniendo la extensión original.

### 7.3 Paso 2: convertir el audio a MP3 de alta calidad

```bash
ffmpeg -i concierto_original.webm -acodec libmp3lame -b:a 320k concierto.mp3
```

Este comando convierte el audio descargado a MP3 usando el códec `libmp3lame` y un bitrate de 320 kbps.

### 7.4 Paso 3: dividir el audio en fragmentos de 10 minutos

```bash
ffmpeg -i concierto.mp3 -f segment -segment_time 600 -c copy "podcast_parte_%03d.mp3"
```

Donde:

* `-f segment` indica que se usará el muxer de segmentación.
* `-segment_time 600` define fragmentos de 600 segundos, es decir, 10 minutos.
* `-c copy` copia el audio sin recodificarlo de nuevo.
* `%03d` genera números de tres dígitos: `000`, `001`, `002`, etc.

### 7.5 Resultado

Se obtienen archivos como:

```
podcast_parte_000.mp3
podcast_parte_001.mp3
podcast_parte_002.mp3
```

Estos fragmentos estarían listos para subirse a una plataforma de podcast.

### 7.6 Screenshot recomendado

Captura los tres pasos ejecutados y el listado final:

```bash
ls -lh podcast_parte_*.mp3
```

**Observación de evaluación:**\
Este ejemplo demuestra un flujo real de trabajo: descarga, conversión y segmentación. Además, deja claro que yt-dlp y FFmpeg tienen funciones complementarias.

***

## 8. Comandos para screenshots

Esta sección resume qué comandos ejecutar para obtener cada captura necesaria para la documentación.

### 8.1 Captura de actualización del sistema

```bash
sudo apt update && sudo apt upgrade -y
```

Captura recomendada: terminal mostrando la actualización de repositorios y paquetes.

### 8.2 Captura de instalación de FFmpeg

```bash
sudo apt install ffmpeg -y
```

Captura recomendada: terminal mostrando dependencias y mensaje final `Setting up ffmpeg`.

### 8.3 Captura de verificación de FFmpeg

```bash
ffmpeg -version
```

Captura recomendada: output completo con versión y librerías.

### 8.4 Captura de ayuda de FFmpeg

```bash
ffmpeg -h
```

Captura recomendada: primeras líneas de uso y opciones básicas.

### 8.5 Captura de descarga de vídeo de prueba

```bash
wget https://www.learningcontainer.com/wp-content/uploads/2020/05/sample-mp4-file.mp4 -O coldplay.mp4
```

Captura recomendada: descarga completada y archivo guardado.

### 8.6 Captura de información del vídeo

```bash
ffmpeg -i coldplay.mp4
```

Captura recomendada: sección con duración, streams, códec de vídeo y códec de audio.

### 8.7 Captura de conversión con H.264

```bash
ffmpeg -i coldplay.mp4 -vcodec libx264 coldplay_264.mkv
```

Captura recomendada: progreso y resumen final.

### 8.8 Captura de conversión con H.265

```bash
ffmpeg -i coldplay.mp4 -vcodec libx265 coldplay_265.mkv
```

Captura recomendada: progreso, velocidad y resumen final.

### 8.9 Captura de comparación de tamaños

```bash
ls -lh coldplay.mp4 coldplay_264.mkv coldplay_265.mkv
```

Captura recomendada: listado mostrando los tres tamaños.

### 8.10 Captura de cambio de códec de audio

```bash
ffmpeg -i coldplay.mp4 -vcodec copy -acodec mp3 coldplay_mp3.mkv
ffmpeg -i coldplay.mp4 -vcodec copy -acodec aac coldplay_aac.mkv
ffmpeg -i coldplay.mp4 -vcodec copy -acodec libvorbis coldplay_vorbis.mkv
```

Captura recomendada: ejecución de los comandos y salida final.

### 8.11 Captura de comparación de audios

```bash
ls -lh coldplay_mp3.mkv coldplay_aac.mkv coldplay_vorbis.mkv
```

Captura recomendada: listado comparando los tamaños generados.

### 8.12 Captura de bitrate personalizado

```bash
ffmpeg -i coldplay.mp4 -b:v 2500k -b:a 192k coldplay_custom.mp4
```

Captura recomendada: progreso y bitrate aproximado.

### 8.13 Captura de extracción de audio

```bash
ffmpeg -i coldplay.mp4 -vn coldplay_audio.mp3
```

Captura recomendada: stream mapping y velocidad de procesamiento.

### 8.14 Captura de verificación del audio extraído

```bash
ffmpeg -i coldplay_audio.mp3
```

Captura recomendada: output mostrando solo stream de audio.

### 8.15 Captura de recorte de vídeo

```bash
ffmpeg -i coldplay.mp4 -ss 35 -t 30 coldplay_frag.mp4
```

Captura recomendada: output del recorte.

### 8.16 Captura de verificación del fragmento

```bash
ffprobe coldplay_frag.mp4
```

Captura recomendada: duración aproximada de 30 segundos.

### 8.17 Captura de verificación de Python

```bash
python3 --version
```

Captura recomendada: versión de Python instalada.

### 8.18 Captura de instalación de yt-dlp

```bash
sudo wget https://github.com/yt-dlp/yt-dlp/releases/latest/download/yt-dlp -O /usr/local/bin/yt-dlp
```

Captura recomendada: descarga completada.

### 8.19 Captura de permisos de yt-dlp

```bash
sudo chmod a+rx /usr/local/bin/yt-dlp
ls -l /usr/local/bin/yt-dlp
```

Captura recomendada: permisos de ejecución visibles.

### 8.20 Captura de versión de yt-dlp

```bash
yt-dlp --version
```

Captura recomendada: número de versión.

### 8.21 Captura de ayuda de yt-dlp

```bash
yt-dlp -h
```

Captura recomendada: primeras líneas de opciones.

### 8.22 Captura de formatos disponibles

```bash
yt-dlp -F "https://www.youtube.com/watch?v=zrnCBt2q-dY"
```

Captura recomendada: tabla de formatos con ID, extensión, resolución y códecs.

### 8.23 Captura de descarga por ID

```bash
yt-dlp -f 137 "https://www.youtube.com/watch?v=zrnCBt2q-dY"
```

Captura recomendada: progreso hasta 100 %.

### 8.24 Captura de descarga mejor calidad con fusión

```bash
yt-dlp -f "bv*+ba" "https://www.youtube.com/watch?v=bH3NFlkui4Y"
```

Captura recomendada: proceso de descarga y línea `[ffmpeg] Merging formats into ...`.

### 8.25 Captura de extracción de audio con yt-dlp

```bash
yt-dlp -x --audio-format mp3 "https://www.youtube.com/watch?v=zrnCBt2q-dY"
```

Captura recomendada: descarga y conversión a MP3.

### 8.26 Captura del ejemplo propio

```bash
yt-dlp -f "bestaudio" -o "concierto_original.%(ext)s" "https://www.youtube.com/watch?v=XXXXXXXXX"
ffmpeg -i concierto_original.webm -acodec libmp3lame -b:a 320k concierto.mp3
ffmpeg -i concierto.mp3 -f segment -segment_time 600 -c copy "podcast_parte_%03d.mp3"
ls -lh podcast_parte_*.mp3
```

Captura recomendada: comandos ejecutados y listado final de fragmentos generados.

***

## 9. Referencias y fuentes

* FFmpeg Official Site: [https://ffmpeg.org](https://ffmpeg.org/)
* FFmpeg Documentation: [https://ffmpeg.org/documentation.html](https://ffmpeg.org/documentation.html)
* FFmpeg Wiki: [https://trac.ffmpeg.org/wiki](https://trac.ffmpeg.org/wiki)
* FFmpeg About: [https://ffmpeg.org/about.html](https://ffmpeg.org/about.html)
* FFmpeg Manual: [https://ffmpeg.org/ffmpeg.html](https://ffmpeg.org/ffmpeg.html)
* FFmpeg Codecs Documentation: [https://ffmpeg.org/ffmpeg-codecs.html](https://ffmpeg.org/ffmpeg-codecs.html)
* FFmpeg Formats Documentation: [https://ffmpeg.org/ffmpeg-formats.html](https://ffmpeg.org/ffmpeg-formats.html)
* FFmpeg Filters Documentation: [https://ffmpeg.org/ffmpeg-filters.html](https://ffmpeg.org/ffmpeg-filters.html)
* yt-dlp GitHub Repository: [https://github.com/yt-dlp/yt-dlp](https://github.com/yt-dlp/yt-dlp)
* yt-dlp Installation Wiki: [https://github.com/yt-dlp/yt-dlp-wiki/blob/master/Installation.md](https://github.com/yt-dlp/yt-dlp-wiki/blob/master/Installation.md)
* yt-dlp README: [https://github.com/yt-dlp/yt-dlp#readme](https://github.com/yt-dlp/yt-dlp#readme)
* RTMP Open Specification: [https://rtmp.veriskope.com/docs/spec/](https://rtmp.veriskope.com/docs/spec/)
* Apple HLS Documentation: [https://developer.apple.com/documentation/http-live-streaming](https://developer.apple.com/documentation/http-live-streaming)
* RTSP RFC 2326: [https://www.rfc-editor.org/rfc/rfc2326](https://www.rfc-editor.org/rfc/rfc2326)
* SRT Alliance: [https://www.srtalliance.org/](https://www.srtalliance.org/)
* ITU-T H.264 Recommendation: [https://www.itu.int/rec/T-REC-H.264/en](https://www.itu.int/rec/T-REC-H.264/en)
* ITU-T H.265 Recommendation: [https://www.itu.int/rec/T-REC-H.265/en](https://www.itu.int/rec/T-REC-H.265/en)
* Matroska Specification: [https://www.matroska.org/technical/specs/index.html](https://www.matroska.org/technical/specs/index.html)
* ISO MP4 Standard: [https://www.iso.org/standard/68960.html](https://www.iso.org/standard/68960.html)
* HandBrake Official: [https://handbrake.fr/](https://handbrake.fr/)
* GStreamer Official: [https://gstreamer.freedesktop.org/](https://gstreamer.freedesktop.org/)
* AOM AV1 Codec: [https://aomedia.org/av1/](https://aomedia.org/av1/)
* Google VP9 Documentation: [https://developers.google.com/media/vp9](https://developers.google.com/media/vp9)

***

Documento elaborado para MP0375 – Servicios | CFGS ASIX.
