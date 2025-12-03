# EXAM

### 1. Comandos de control básico

#### `@echo off`

* **Qué hace en general**: desactiva el “eco” de los comandos; es decir, que la consola deje de mostrar cada línea que se ejecuta.
*   **En el script**:\
    Al principio:

    ```bat
    @echo off
    ```

    Hace que solo se vea la interfaz del juego (textos, opciones), no las líneas reales de BATCH.

***

#### `REM`

* **Qué hace**: introduce un comentario en BATCH. No se ejecuta nada, sirve solo como documentación.
*   **En el script**:\
    Ejemplo:

    ```bat
    REM - Empezamos por la escena 1
    ```

    El comando `REM` no cambia nada en la ejecución, es solo una explicación para el humano.

***

#### `chcp 65001 >nul`

* **Qué hace `chcp`**: cambia la página de códigos de la consola (codificación de caracteres).
* `65001` es UTF-8.
* `>nul` redirige la salida estándar a `NUL`, es decir, oculta el mensaje de “Página de códigos activa: 65001”.
*   **En el script**:

    ```bat
    chcp 65001 >nul
    ```

    Permite mostrar bien ñ, tildes y caracteres especiales sin ensuciar la pantalla con mensajes técnicos.

***

#### `cls`

* **Qué hace**: limpia completamente la pantalla de la consola.
*   **En el script**:

    ```bat
    call :color_ambiente "%AMBIENTE%"
    cls
    ```

    Después de aplicar el color, limpia lo anterior para que el jugador vea solo la escena actual.

***

#### `title`

* **Qué hace**: cambia el título de la ventana de CMD.
*   **En el script**:

    ```bat
    :titulo
        title %~1
        goto :eof
    ```

    Se llama como:

    ```bat
    call :titulo "Inicio"
    ```

    El texto `"Inicio"` se pasa como parámetro y se usa para cambiar el título a “Inicio”.

***

### 2. Variables y parámetros

#### `set` (asignación básica)

* **Qué hace**: crea o modifica una variable de entorno.
*   **Sintaxis**:

    ```bat
    set NOMBRE=valor
    set "NOMBRE=valor con espacios"
    ```
*   **En el script**:

    ```bat
    set "BASE=%~dp0"
    set "escena=1"
    set "AMBIENTE="
    set "i=0"
    ```

    Cada una define una variable que se usará después:

    * `BASE`: directorio donde está el .bat.
    * `escena`: número de escena actual.
    * `AMBIENTE`: ambiente leído del fichero.
    * `i`: contador de opciones.

***

#### `set /A`

* **Qué hace**: realiza operaciones aritméticas con variables.
*   **Sintaxis**:

    ```bat
    set /A contador+=1
    ```
*   **En el script**:

    ```bat
    set /a i+=1
    ```

    Incrementa `i` en 1 cada vez que se lee una opción de `opciones.txt`.

***

#### `setlocal enabledelayedexpansion`

* **Qué hace**:
  * `setlocal`: inicia un ámbito local de variables (las que se modifiquen dentro no afectan fuera al terminar).
  * `enabledelayedexpansion`: permite usar `!VAR!` para ver el valor actualizado dentro de bucles.
*   **En el script**:

    ```bat
    setlocal enabledelayedexpansion
    ```

    Esto es imprescindible para:

    ```bat
    echo !i!. %%O
    set "destino[!i!]=%%N"
    ```

    Sin delayed expansion, `i` no se actualizaría correctamente dentro del `for`.

***

#### `%variable%` y `!variable!`

* `%variable%`: se expande una vez al principio de la línea (modo normal).
* `!variable!`: se expande dinámicamente cada vez dentro de bucles cuando está activo `enabledelayedexpansion`.
*   **En el script**:

    *   Fuera de bucles:

        ```bat
        echo La escena %escena% no existe.
        ```
    *   Dentro de bucles:

        ```bat
        echo !i!. %%O
        set "destino[!i!]=%%N"
        set "proxima=!destino[%op%]!"
        ```

    En estos últimos casos se usa `!` porque `i` y `destino[...]` cambian dentro del `for`.

***

#### Parámetros `%~1`, `%~dp0`, etc.

* `%1`, `%2`, etc.: parámetros que recibe una subrutina o un .bat.
* Modificadores como `~d`, `~p`, `~dp`, etc.:
  * `%~dp0`: unidad (d) + ruta (p) del script actual.
  * `%~1`: primer parámetro sin comillas.
*   **En el script**:

    ```bat
    set "BASE=%~dp0"       :: ruta del juego
    set "escena=%~1"       :: número de escena pasado a :escena
    title %~1              :: título pasado a :titulo
    ```

***

### 3. Estructuras de control: etiquetas, GOTO, CALL, :EOF

#### `:label` (etiquetas)

* **Qué hace**: define un punto al que se puede saltar con `goto` o llamar con `call`.
*   **En el script**:

    ```bat
    :inicio
    :escena
    :ambiente_leido
    :color_ambiente
    :titulo
    ```

    Cada etiqueta marca una “subrutina” o sección lógica.

***

#### `goto`

* **Qué hace**: salta a una etiqueta dentro del mismo archivo.
*   **En el script**:

    ```bat
    goto :inicio
    goto ambiente_leido
    goto :eof
    ```

    * `goto :inicio`: vuelve al bucle principal.
    * `goto ambiente_leido`: salta a la etiqueta para continuar el flujo.
    * `goto :eof`: sale de una subrutina (equivalente a `return`).

***

#### `:eof`

* **Qué es**: etiqueta especial que representa “End Of File”.\
  `goto :eof` hace que el flujo vuelva a quien hizo `call` de esa subrutina.
*   **En el script**:\
    Al final de casi todas las subrutinas:

    ```bat
    goto :eof
    ```

    Eso hace que el control regrese al punto desde donde se llamó con `call`.

***

#### `call`

* **Qué hace**:
  * Llama a otra etiqueta como si fuera una función.
  * Al finalizar, se vuelve al punto siguiente al `call`.
*   **En el script**:

    ```bat
    call :escena "%escena%"
    call :color_ambiente "%AMBIENTE%"
    call :titulo "Inicio"
    ```

    Cada `call` transfiere el control a una subrutina, pasando parámetros.

***

### 4. Entrada y salida: `echo`, `type`, `choice`, `pause`

#### `echo`

* **Qué hace**: escribe texto en la consola.
*   **En el script**:\
    Se usa para mensajes, menús, errores:

    ```bat
    echo La escena %escena% no existe.
    echo Gracias por jugar EGO.
    echo !i!. %%O
    ```

    No altera la lógica, solo lo que ve el jugador.

***

#### `type`

* **Qué hace**: muestra el contenido de un archivo de texto.
*   **En el script**:

    ```bat
    if exist "escenario.txt" (
        type "escenario.txt"
    )
    ```

    Muestra el texto narrativo de la escena.

***

#### `choice`

* **Qué hace**: permite al usuario elegir una opción de un conjunto de teclas.\
  Devuelve el número de la opción en `errorlevel`.
*   **Sintaxis usada**:

    ```bat
    choice /c 123456789 /n /m "Elige: "
    ```

    * `/c 123456789`: teclas válidas (1 a 9).
    * `/n`: no muestra `[123456789]?` al lado.
    * `/m "Elige: "`: mensaje mostrado.
*   **En el script**:

    ```bat
    choice /c 123456789 /n /m "Elige: "
    set "op=%errorlevel%"
    ```

    Si el usuario pulsa 2, `errorlevel` = 2, y `op` se convierte en 2.

***

#### `pause`

* **Qué hace**: detiene la ejecución hasta que el usuario pulse una tecla.
*   **En el script**:\
    Se usa en mensajes de error o escenas sin opciones:

    ```bat
    pause
    pause >nul
    ```

    Con `>nul` se oculta el “Presione una tecla para continuar . . .”.

***

### 5. Manejo de directorios y archivos

#### `cd /d`

* **Qué hace**: cambia el directorio actual.\
  El modificador `/d` permite cambiar de unidad (por ejemplo, de C: a D:).
*   **En el script**:

    ```bat
    cd /d "%BASE%\%escena%" 2>nul
    cd /d "%BASE%"
    ```

    * Entra a la carpeta de la escena (por número).
    * Vuelve a la carpeta base del juego.

El `2>nul` redirige la salida de error (canal 2) a NUL para ocultar posibles mensajes si la carpeta no existe.

***

#### `if exist`

* **Qué hace**: comprueba si existe un archivo o carpeta.
*   **En el script**:

    ```bat
    if exist "ambiente.txt" ( ... )
    if exist "escenario.txt" ( ... ) else ( ... )
    if not exist "opciones.txt" ( ... )
    ```

    Determina si se pueden leer los ficheros necesarios para la escena.

***

#### `if errorlevel`

* **Qué hace**: comprueba el código de salida del último comando.\
  En BATCH tradicional, `if errorlevel N` significa “si `errorlevel` es mayor o igual que N”.
*   **En el script**:

    ```bat
    cd /d "%BASE%\%escena%" 2>nul
    if errorlevel 1 (
        echo La escena %escena% no existe.
        ...
    )
    ```

    Si `cd` falla (por ejemplo, la carpeta no existe), `errorlevel` será 1 y se entra en el bloque de error.

***

### 6. Bucles y lectura de ficheros

#### `for /f`

* **Qué hace**: recorre líneas de un archivo o salida de comando y divide en tokens.
* **Sintaxis usada**:

1.  Leer una sola línea (`ambiente.txt`):

    ```bat
    for /f "usebackq delims=" %%A in ("ambiente.txt") do (
        set "AMBIENTE=%%A"
        goto ambiente_leido
    )
    ```

    * `usebackq`: permite usar comillas normales `"archivo.txt"`.
    * `delims=`: no usar separadores; se coge la línea entera.
    * `%%A`: variable que representa toda la línea.
    * Se hace `goto` para solo leer la primera línea.
2.  Leer opciones en `opciones.txt`:

    ```bat
    for /f "usebackq tokens=1* delims= " %%N in ("opciones.txt") do (
        set /a i+=1
        echo !i!. %%O
        set "destino[!i!]=%%N"
    )
    ```

    * `tokens=1*`: el primer campo se va a `%%N` (DESTINO), el resto de la línea a `%%O` (texto de la opción).
    * `delims=` : separador es el espacio.
    * Dentro del bucle:
      * `i` se incrementa.
      * Se imprime el número de opción + texto.
      * Se guarda el destino en un “array” `destino[i]`.

***

#### `for /l`

* **Qué hace**: bucle numérico: inicio, incremento, fin.
*   **En el script**:

    ```bat
    for /l %%K in (1,1,9) do set "destino[%%K]="
    ```

    Borra del 1 al 9 todas las posiciones del array `destino[...]` para dejarlo limpio antes de la siguiente escena.

***

### 7. Operadores de comparación y opciones de IF

#### `if /I`

* **Qué hace**: hace la comparación de cadenas ignorando mayúsculas/minúsculas.
*   **En el script**:

    ```bat
    if /I "!proxima!"=="SALIR" ( ... )
    if /I "!proxima!"=="REINICIO" ( ... )
    if /I "%A%"=="INICIO" color 07 & call :titulo "Inicio" & goto :eof
    ```

    Compara textos sin importar si están en mayúsculas o minúsculas.

***

#### `not defined`

* **Qué hace**: comprueba si una variable no existe o está vacía.
*   **En el script**:

    ```bat
    if not defined proxima (
        echo Opcion invalida.
        ...
    )
    ```

    Si el usuario ha elegido un número para el que no existe `destino[...]`, se indica que la opción es inválida.

***

#### `EQU`

* **Qué hace**: comparación numérica “igual que” en `if`.
*   **En el script**:

    ```bat
    if !i! EQU 0 (
        echo Esta escena no tiene opciones.
        ...
    )
    ```

    Si `i` sigue valiendo 0 después del `for`, significa que no se leyó ninguna opción.

***

### 8. Comando `color`

* **Qué hace**: cambia el color de fondo y de texto de la consola según un código hexadecimal:\
  `color XY` → `X` fondo, `Y` texto.
*   **En el script**:

    ```bat
    if /I "%A%"=="INICIO"       color 07 & call :titulo "Inicio" & goto :eof
    if /I "%A%"=="TEIRM"        color 08 & call :titulo "Teirm" & goto :eof
    ...
    color 07
    call :titulo "Escena"
    ```

    Cada ambiente (INICIO, TEIRM, CICLOPE, BRUJA, etc.) define un color distinto para dar atmósfera.

El operador `&` encadena varios comandos en la misma línea:\
`color 07 & call :titulo "Inicio" & goto :eof` ejecuta los tres en orden.

***

### 9. `exit /b`

* **Qué hace**: sale del archivo por lotes o subrutina actual y devuelve el control a la consola o a quien lo llamó.
*   **En el script**:

    ```bat
    if /I "!proxima!"=="SALIR" (
        cd /d "%BASE%"
        echo.
        echo Gracias por jugar EGO.
        echo Pulsa una tecla para salir...
        pause >nul
        exit /b
    )
    ```

    Cuando el destino es `SALIR`, se sale del programa después del mensaje de despedida.

Perfeito, agora vou te montar um capítulo só de **Linux/Bash**, em espanhol, organizado para GitBook e baseado no PDF de Bash que você enviou. Nada de emojis, só conteúdo claro.

Vou considerar que este é um capítulo tipo:

> `Bloque 1 · Scripting en Linux (Bash)`

E você cola isso no GitBook.

***

## 1. Introducción: la shell y Bash

En GNU/Linux trabajamos con un **intérprete de comandos** (shell). Hay varios tipos: `sh`, `csh`, `ksh`, `bash`, `zsh`… En este módulo se usa principalmente **Bash**.&#x20;

Aspectos clave:

* El sistema es **case sensitive**: `ECHO` no es lo mismo que `echo`.
*   La sintaxis general de un comando es:

    ```bash
    comando arg1 arg2 arg3 ...
    ```
* Si un programa no está en el `PATH`, hay que ejecutarlo con `./programa`.&#x20;
* Prompt típico:
  * `$` → usuario normal
  * `#` → usuario root

***

## 2. Comandos básicos de usuario

Listado de comandos básicos que se suelen usar antes de escribir scripts:&#x20;

* `ls`: listar ficheros y directorios.
* `man`: mostrar el manual de un comando.
* `pwd`: mostrar el directorio actual.
* `cd`: cambiar de directorio.
* `echo`: escribir texto en la salida estándar.
* `cat`: mostrar contenido de un fichero.
* `more`: paginar un fichero.
* `file`: mostrar el tipo de fichero.
* `touch`: crear o actualizar la fecha de un fichero.
* `rm`: borrar ficheros.
* `mkdir`: crear directorios.
* `rmdir`: borrar directorios vacíos.
* `cp`: copiar ficheros.
* `mv`: mover/renombrar.
* `ln`: crear enlaces (hard/simbólicos).
* `date`: mostrar la fecha actual.

***

## 3. Redirecciones y tuberías

En Linux, todo se maneja como flujos:

* `0`: STDIN (entrada estándar, teclado por defecto)
* `1`: STDOUT (salida estándar, pantalla por defecto)
* `2`: STDERR (salida de error)&#x20;

### 3.1 Redirecciones de salida

* `>` redirige la salida estándar sobrescribiendo el fichero.
* `>>` añade al final del fichero.
* `2>` redirige la salida de error.
* `2>>` añade la salida de error al final.

Ejemplos:

```bash
ls > listado.txt          # Guarda la salida en listado.txt
ls >> listado.txt         # Añade al final
ls 2> errores.txt         # Mensajes de error a errores.txt
ls 2>> errores.txt        # Añade errores
```

### 3.2 Redirecciones de entrada

* `<` redirige el contenido de un fichero a la entrada estándar del comando.

Ejemplo:

```bash
tr a A < fichero.txt
```

Equivale a “leer fichero.txt” y pasar su contenido a `tr`.&#x20;

### 3.3 Tuberías (pipes)

Las tuberías conectan la salida de un comando con la entrada del siguiente:

```bash
comando1 | comando2
```

Ejemplo:

```bash
cat fichero.txt | tr a A
```

***

## 4. Filtros de texto importantes

Son comandos que trabajan sobre texto y son muy útiles en scripting:&#x20;

* `sort`: ordena líneas de texto.
* `tr`: sustituye caracteres (`tr a A`).
* `head`: muestra las primeras N líneas (`head -n 10`).
* `tail`: muestra las últimas N líneas (`tail -n 20`).
* `wc`: cuenta líneas, palabras, caracteres (`wc -l`, `wc -w`, `wc -c`).
* `cut`: corta columnas según un delimitador (`cut -d: -f1`).

***

## 5. Búsqueda y expresiones regulares: grep, awk, find

### 5.1 grep

`grep` busca una cadena o patrón en uno o varios ficheros y muestra las líneas que lo contienen.&#x20;

Sintaxis:

```bash
grep [opciones] patron [fich1 fich2 ...]
```

Metacaracteres básicos que se pueden usar en el patrón:

* `.` → cualquier carácter
* `*` → cero o más repeticiones del carácter anterior
* `^` → principio de línea
* `$` → final de línea
* `[a-f]` → cualquier carácter entre a y f
* `[^56]` → cualquier carácter excepto 5 y 6

Opciones típicas:&#x20;

* `-c` → solo cuenta ocurrencias.
* `-i` → ignora mayúsculas/minúsculas.
* `-l` → muestra solo nombres de ficheros donde se encuentra el patrón.
* `-s` → modo silencioso (solo errores).
* `-v` → muestra líneas que NO contienen el patrón.
* `-w` → trata el patrón como palabra completa.

Uso típico en combinación con otros comandos:

```bash
comando | grep ...
```

### 5.2 awk (visión general)

`awk` es un lenguaje para procesar texto por campos. Cada línea es un registro y se divide en campos `$1`, `$2`, …, `$0` es la línea completa.&#x20;

Sintaxis general:

```bash
awk 'patron { accion }' fichero
```

Ejemplos útiles:

```bash
awk '{print $1}' fichero          # imprime el primer campo
awk '{print $1,$3}' fichero       # imprime campo 1 y 3
awk '$3 > 100 {print $1,$3}' fich # solo líneas donde el campo 3 > 100
awk '/error/ {print $0}' fich     # líneas que contienen "error"
```

Se puede usar sin fichero, leyendo de stdin:

```bash
echo "a b c" | awk '{print $2}'   # imprime "b"
```

### 5.3 find

`find` sirve para buscar ficheros en directorios según criterios varios: nombre, tipo, fechas, etc.&#x20;

Sintaxis:

```bash
find [path] opciones
```

Opciones típicas:

* `-name "patrón"` → nombre del fichero.
* `-type f` / `-type d` → ficheros / directorios.
* `-maxdepth n` / `-mindepth n` → profundidad máxima/mínima.
* `-mtime n` → ficheros modificados hace n días.
* `-exec comando {} \;` → ejecutar un comando para cada fichero encontrado.

Ejemplos:

```bash
find /etc -name "*.conf"
find . -type f -maxdepth 2
find . -name "*.log" -exec rm {} \;
```

***

## 6. Usuarios, grupos, permisos y procesos

### 6.1 Gestión de usuarios y grupos

Comandos típicos que aparecen en temario:&#x20;

* `useradd`: añade un usuario (bajo nivel).
* `userdel`: elimina un usuario.
* `usermod`: modifica un usuario.
* `groupadd`: añade un grupo.
* `groupdel`: elimina un grupo.
* `groupmod`: modifica un grupo.
* `passwd`: cambia la contraseña de un usuario.
* `gpasswd`: gestiona pertenencia a grupos.

(En Debian/Ubuntu, `adduser` es un script de alto nivel que llama a utilidades como `useradd`, copia `/etc/skel`, crea `/home`, etc., mientras `useradd` es más básico y no interactivo.)

### 6.2 Propietarios y permisos

Comandos clave:&#x20;

* `chmod`: cambia los permisos de acceso (lectura, escritura, ejecución).
* `chown`: cambia el propietario de un fichero.
* `chgrp`: cambia el grupo propietario.
* `who`: muestra usuarios conectados.
* `whoami`: indica el usuario actual.
* `id`: muestra UID, GID y grupos.
* `su`: cambia de usuario.

### 6.3 Gestión de procesos

Comandos do PDF:&#x20;

* `top`: monitor de procesos en tiempo real.
* `ps`: lista de procesos en ejecución.
* `pstree`: muestra procesos en forma de árbol.
* `pgrep`: busca PIDs por nombre de proceso.
* `pidof`: da el PID de un programa.
* `kill`: envía una señal a un proceso (por PID).
* `killall`: envía señal a todos los procesos con un nombre.

Control de trabajos:

* `&` → lanzar un proceso en segundo plano.
* `bg` → enviar un trabajo al background.
* `fg` → traer un trabajo al foreground.
* `ctrl+z` → suspender un trabajo.
* `ctrl+c` → terminar un proceso (SIGINT).
* `jobs` → listar trabajos en segundo plano.
* `nohup`: hace que el proceso siga aunque se cierre la sesión, redirigiendo la salida a `nohup.out`.
* `disown`: “desengancha” un proceso de la shell actual.&#x20;

***

## 7. Compresión y backups

Comandos de backup más comunes:&#x20;

* `gzip`: compresor estándar.
* `bzip2`: compresor más potente pero más lento.
* `tar`: empaqueta múltiples ficheros/directorios en uno.
* `zcat`, `zmore`, `zgrep`: versiones para trabajar sobre `.gz` sin descomprimir explícitamente.

Ejemplo de combinación:

```bash
tar czf backup.tar.gz /etc
zgrep cadena fichero.gz
gzip -dc fichero.gz | grep cadena
```

***

## 8. Primeros pasos con shell scripts

### 8.1 Primer script y shebang

Un script es un archivo de texto con comandos que la shell ejecuta.&#x20;

Ejemplo básico `hola.sh`:

```bash
#!/bin/bash
echo hola
```

* La primera línea `#!/bin/bash` se llama _shebang_ e indica qué intérprete ejecutará el script.
*   Hay que darle permisos de ejecución:

    ```bash
    chmod +x hola.sh
    ./hola.sh
    ```

### 8.2 Variables

Declaración de variables:

```bash
FECHA="15/07/2004"
echo "Hoy es $FECHA"
```

Puntos importantes:

* No se usan espacios en torno a `=`:
  * Correcto: `VAR=valor`
  * Incorrecto: `VAR = valor`

### 8.3 Variables de entorno y `export`

Al iniciar la shell, já existem muitas variáveis definidas (PATH, HOME, USER…). Podemos vê-las com:

```bash
env
```

Una variable definida en una shell solo vive en esa shell, a menos que se haga `export`:

```bash
USUARIO="AlumnoIFP"
export USUARIO
```

Cualquier script lanzado desde esa shell podrá ver `USUARIO`.&#x20;

### 8.4 Interactividad: `read`

Podemos pedir datos al usuario:

```bash
#!/bin/bash
echo "Buenas tardes, dime tu nombre:"
read NOMBRE
echo "Hola $NOMBRE, encantado de conocerte"
```

### 8.5 Argumentos de un script

Un script puede recibir parámetros:

```bash
./nombre.sh Jorge Martinez Peña
./nombre.sh "Maria Dolores" Perez Belloch
```

Dentro do script:

* `$1`, `$2`, `$3`… → argumentos.
* `$0` → nombre del script.
* `$#` → número de argumentos.
* `$*` / `$@` → todos los argumentos.&#x20;

Ejemplo:

```bash
#!/bin/bash
echo "Nombre: $1"
echo "Primer Apellido: $2"
echo "Segundo Apellido: $3"
```

Otras utilidades:

* `basename "$0"` → solo el nombre del script.
* `dirname "$0"` → ruta del script.
* `shift` → desplaza los argumentos: lo que estaba en `$2` pasa a `$1`, etc.&#x20;

### 8.6 Sustitución de comandos

Podemos guardar el resultado de un comando en una variable:

```bash
LISTADO=$(ls)
```

Esta forma `$(...)` es preferida porque permite anidar otras sustituciones.&#x20;

***

## 9. Operaciones aritméticas con `expr`

`expr` permite hacer operaciones aritméticas básicas:&#x20;

```bash
SUMA=$(expr 7 + 5)
expr 7 \> 5          # comparación (ojo: hay que escapar >)
expr \( 7 + 5 \) \* 2
```

En scripts modernos, también se usa `(( ))`:

```bash
n=0
((n = n + 1))
```

***

## 10. Control de flujo en Bash

### 10.1 Test y condición

Se usa `test` o la sintaxis `[ ]` para comprobar condiciones; modifican `$?` (0 = verdadero, distinto de 0 = falso).&#x20;

Ejemplos:

*   Comparar strings:

    ```bash
    [ "$NOMBRE" = "Juan" ]
    [ "$NOMBRE" != "Juan" ]
    ```
*   Comparar números:

    ```bash
    [ "$DINERO" -eq 1000 ]
    [ "$DINERO" -gt 500 ]
    ```
*   Ficheros:

    ```bash
    [ -f /etc/passwd ]    # fichero regular
    [ -d /home/usuario ]  # directorio
    [ -r archivo ]        # legible
    [ -w archivo ]        # escribible
    [ -x script.sh ]      # ejecutable
    ```

### 10.2 Estructura `if`

Sintaxis completa:&#x20;

```bash
if condicion
then
  comandos_then
elif otra_condicion
then
  comandos_elif
else
  comandos_else
fi
```

Ejemplo:

```bash
if [ "$NOMBRE" = "Juan" ]; then
  echo "Hola Juanin, ¿qué tal?"
elif [ "$NOMBRE" = "Pedro" ]; then
  echo "Pedrete, ¡cuánto tiempo!"
else
  echo "No te conozco"
fi
```

### 10.3 `case`

Para evitar muchos `if` anidados:&#x20;

```bash
case $VARIABLE in
  "VALOR1") comandos_valor1 ;;
  "VALOR2") comandos_valor2 ;;
  *)        comandos_default ;;
esac
```

***

## 11. Bucles: while, until, for, select

### 11.1 while

Ejecuta de 0 a N veces mientras la condición sea verdadera:&#x20;

```bash
N=1
while [ "$N" -lt 100 ]; do
  echo "Repito, ya llevo $N veces"
  N=$(expr $N + 1)
  sleep 1
done
```

### 11.2 until

Ejecuta de 0 a N veces mientras la condición sea falsa (es decir, hasta que se cumpla).&#x20;

```bash
N=1
until [ "$N" -ge 100 ]; do
  echo "Repito, ya llevo $N veces"
  N=$(expr $N + 1)
done
```

### 11.3 for

Permite iterar sobre una lista de valores:&#x20;

```bash
for N in "Ensalada" "Pasta" "Queso"; do
  echo "Hoy comemos $N"
done
```

Se puede usar `IFS` para cambiar el separador (por defecto es espacio, tabulación y salto de línea). Ejemplo clásico con `$PATH`:&#x20;

```bash
IFS=":"
echo "Directorios en el PATH..."
for DIR in $PATH; do
  echo "$DIR"
done
```

Ejemplo numérico con `seq`:&#x20;

```bash
for N in $(seq 10); do
  echo "N ahora vale $N"
done
```

### 11.4 select

Genera un menú interactivo:&#x20;

```bash
PS3="Elige un personaje: "
select PERS in "Superman" "Batman" "Spiderman"; do
  echo "Has elegido a $PERS"
  echo "El número es $REPLY"
done
```

* `PS3` define el mensaje del prompt.
* `PERS` es el valor elegido.
* `REPLY` es el número tecleado.

***

## 12. Funciones en Bash y carga con `source`

### 12.1 Definición de funciones

O PDF mostra vários formatos equivalentes:&#x20;

```bash
function suma {
  echo $(expr $1 + $2)
}
```

ou

```bash
suma() {
  echo $(expr $1 + $2)
}
```

Chamada:

```bash
suma 14 26
suma 5 23
```

Notas importantes:&#x20;

* O corpo da função fica entre `{` e `}`.
* As funções devem estar definidas antes de serem chamadas.
* Dentro da função, `$1`, `$2`, … são os parâmetros da função, não do script principal.

### 12.2 Alcance de variables (`local`)

Exemplo do PDF:&#x20;

```bash
var1='A'
var2='B'

mi_funcion () {
  local var1='C'
  var2='D'
  echo "Dentro de la función: var1: $var1, var2: $var2"
}

echo "Antes: var1: $var1, var2: $var2"
mi_funcion
echo "Después: var1: $var1, var2: $var2"
```

* `local var1='C'` faz com que a alteração de `var1` só valha dentro da função.
* `var2` é alterada globalmente.

### 12.3 Carga de funciones con `source` (librería en memoria)

Podemos definir funções num ficheiro e “carregá-las” noutro script com `source` ou `.`:&#x20;

```bash
#!/bin/bash
source funciones.sh
suma 34 56
resta 67 77
```

Isto é precisamente “cargar una librería en memoria”: o script com as funções não é executado como programa separado, mas sim importado na shell atual.

É a mesma ideia usada na atividade `descifra_secreto.sh`: o script só define a função, e depois no terminal fazemos:

```bash
source descifra_secreto.sh
descifra_secreto quijote.txt secreto.txt
```

***

Bora, vamos linha por linha então 😎\
Vou explicar **em espanhol**, bem simples, e focar tanto em **o que faz** como **na lógica**.

Pra não virar um muro de texto impossível de ler, vou:

* Ir **script por script**
* Colar o código em blocos
* Embaixo de cada linha (ou pequeno grupo) explicar o que faz

***

### 1) `impares_entre_dos_numeros.bat`

```bat
@echo off
```

* Desactiva la visualización de los comandos en pantalla.
* Solo veremos los mensajes `echo`, no cada línea que se ejecuta.

```bat
chcp 65001 >nul
```

* Cambia la página de códigos a UTF-8 (65001) para que salgan bien las tildes y caracteres especiales.
* `>nul` oculta el mensaje que normalmente muestra `chcp`.

```bat
REM AUTOR: Willoom.
REM FECHA: 5 de noviembre de 2025.
REM RESUMEN:
REM    Pide dos números enteros positivos sin signo.
REM    Comprueba que el primero es menor que el segundo.
REM    Muestra los impares entre ellos (exclusivos).
```

* Comentarios, no se ejecutan.
* Sirven para documentar quién hizo el script, cuándo y qué hace.

```bat
:pedir_primero
```

* Define una **etiqueta**.
* Podemos hacer `goto pedir_primero` para saltar aquí.

```bat
set "num1="
```

* Inicializa (vacía) la variable `num1`.

```bat
set /p "num1=Introduzca el primer número entero positivo: "
```

* Muestra el texto y guarda lo que escriba el usuario en `num1`.
* `/p` significa “leer entrada” (prompt).

```bat
call :es_entero_positivo "%num1%" valido1
```

* Llama a la “función” `:es_entero_positivo`.
* Le pasa el valor de `num1` y el nombre de la variable donde queremos el resultado (`valido1`).
* La función pondrá `valido1=1` si es válido, `valido1=0` si no.

```bat
if "%valido1%"=="0" (
    echo El valor debe ser un entero positivo sin signo.
    goto pedir_primero
)
```

* Si `valido1` es `0`, el valor no es correcto.
* Muestra mensaje de error.
* `goto pedir_primero` vuelve a pedir el primer número.

```bat
:pedir_segundo
```

* Etiqueta para pedir el segundo número.

```bat
set "num2="
set /p "num2=Introduzca el segundo número entero positivo (mayor que el primero): "
```

* Vacía `num2` y pide el segundo número al usuario.

```bat
call :es_entero_positivo "%num2%" valido2
```

* Igual que antes, pero con el segundo número.

```bat
if "%valido2%"=="0" (
    echo El valor debe ser un entero positivo sin signo.
    goto pedir_segundo
)
```

* Si el segundo número no es un entero positivo, vuelve a pedirlo.

```bat
if %num1% GEQ %num2% (
    echo El segundo número debe ser mayor que el primero.
    goto pedir_primero
)
```

* Compara `num1` y `num2`.
* `GEQ` = “greater or equal than” (mayor o igual).
* Si `num1` es mayor o igual que `num2`, no cumple la condición, así que se vuelve al principio a pedir de nuevo el primero.

```bat
echo Números impares entre %num1% y %num2%:
```

* Mensaje informativo.

```bat
set /a i=num1+1
```

* `set /a` hace una operación aritmética.
* Empieza `i` en el número siguiente a `num1`, porque queremos los que están **entre** los dos.

```bat
:loop_impares
```

* Etiqueta del bucle que recorre los números entre `num1` y `num2`.

```bat
if %i% GEQ %num2% goto fin
```

* Si `i` ya es mayor o igual que `num2`, terminamos el bucle y saltamos a `:fin`.

```bat
set /a resto=i%%2
```

* Calcula el resto de `i` al dividir entre 2 (`i % 2`).
* En Batch hay que escribir `%%` para obtener `%`.

```bat
if %resto% NEQ 0 echo %i%
```

* Si el resto **no** es 0 (`NEQ` = not equal), el número es impar.
* Entonces lo muestra.

```bat
set /a i=i+1
```

* Incrementa `i` para pasar al siguiente número.

```bat
goto loop_impares
```

* Vuelve al inicio del bucle para procesar el siguiente valor.

```bat
:fin
echo Fin.
goto :eof
```

* Etiqueta de final.
* Muestra “Fin.” y `goto :eof` termina el script.

***

#### Función `:es_entero_positivo`

```bat
REM --------------------------------------------------
REM Función: es_entero_positivo
REM %1 = cadena a comprobar
REM %2 = nombre de variable donde devolvemos 1 (OK) o 0 (NO)
REM --------------------------------------------------
:es_entero_positivo
setlocal
set "cadena=%~1"
```

* Comentarios explicando qué hace la función.
* `:es_entero_positivo` define la función.
* `setlocal` crea un ámbito local de variables.
* `cadena` guarda el primer parámetro (`%~1`).

```bat
if "%cadena%"=="" (
    endlocal & set "%2=0" & goto :eof
)
```

* Si la cadena está vacía, directamente devolvemos “no válido”.
* `endlocal` sale del ámbito local.
* `set "%2=0"` pone la variable del “llamador” (ej.: `valido1`).
* `goto :eof` vuelve al punto desde donde se llamó.

```bat
for /f "delims=0123456789" %%A in ("%cadena%") do (
    endlocal & set "%2=0" & goto :eof
)
```

* Este truco dice: “si hay algún carácter que **no** sea 0-9, ejecuta este `for`”.
* Si entra en el `for`, significa que hay algo raro → no es número → devuelve 0 (no válido).

```bat
endlocal & set "%2=1"
goto :eof
```

* Si llegamos aquí, el `for` no se ejecutó, así que todos los caracteres son dígitos.
* `set "%2=1"` → número válido.
* `goto :eof` termina la función.

**Lógica geral do script 1:**

1. Pede dois números.
2. Valida que são inteiros positivos.
3. Garante que o primeiro é menor que o segundo.
4. Faz um loop do primeiro+1 até o segundo-1.
5. Mostra só aqueles com resto 1 na divisão por 2 (ímpares).

***

### 2) `monkey_island_insult.bat`

```bat
@echo off
chcp 65001 >nul
```

* Igual que antes: oculta comandos y pone UTF-8.

```bat
REM AUTOR: Willoom.
REM FECHA: 5 de noviembre de 2025.
REM RESUMEN:
REM    ...
```

* Comentarios de documentación.

```bat
set "pregunta=He oído que eres un cobarde, ¡luchas como una vaca!"
```

* Guarda el texto del insulto en la variable `pregunta`.

```bat
echo Insulto de Monkey Island:
echo %pregunta%
echo.
```

* Muestra el título.
* Muestra el insulto guardado.
* `echo.` imprime una línea en blanco.

```bat
echo Elige la respuesta correcta:
echo   1^) Qué apropiado, tú luchas como una vaca.
echo   2^) Yo soy goma, tú eres pegamento.
echo   3^) ¡Qué miedo me das!
echo   4^) He conocido lechugas con más agallas.
echo   5^) Vuelve con tu madre.
```

* Presenta las 5 posibles respuestas.
* `1^)` usa `^` para escapar el paréntesis y que no dé problemas en el `echo`.

```bat
choice /C 12345 /N /M "Opción (1-5): "
```

* `choice` obliga al usuario a pulsar solo una de las teclas indicadas.
* `/C 12345` → teclas válidas: 1,2,3,4,5.
* `/N` → no repetir las opciones en pantalla.
* `/M "texto"` → texto que se muestra como mensaje.

```bat
if errorlevel 5 goto op5
if errorlevel 4 goto op4
if errorlevel 3 goto op3
if errorlevel 2 goto op2
if errorlevel 1 goto op1
```

* `errorlevel` devuelve el número de la opción escogida.
* Importante: se evalúa de mayor a menor, porque `errorlevel` es “>=”.
* Si eligió `5`, va a `op5`; si no, mira `4`, etc.

```bat
:op1
echo ¡Te vencí, mequetrefe!
goto fin
```

* Etiqueta para la opción 1, que es la correcta.
* Muestra mensaje de victoria y salta a `fin`.

```bat
:op2
:op3
:op4
:op5
echo Me has vencido. Me uno a tu tripulación.
goto fin
```

* Todas las demás opciones comparten la misma lógica: mensaje de derrota.

```bat
:fin
```

* Etiqueta final; no hay más comandos, el script termina.

**Lógica geral:**

* Mostra insulto → mostra 5 respostas → `choice` garante opção válida → usa `errorlevel` para saber qual foi escolhida → se 1 = ganha, senão = perde.

***

### 3) `separar_nombre_apellidos.bat`

```bat
@echo off
chcp 65001 >nul
```

* Igual: oculta comandos, configura UTF-8.

```bat
REM AUTOR: Willoom
REM FECHA: 10 de diciembre de 2024
REM DESCRIPCIÓN:
REM    ...
```

* Comentarios explicativos.

```bat
if "%~1"=="" (
    echo Debe indicar la ruta a un fichero de texto.
    goto fin
)
```

* Comprueba si el primer parámetro (`%~1`) está vacío.
* Si no se pasó ningún fichero al script, avisa y termina.

```bat
if not exist "%~1" (
    echo El fichero "%~1" NO existe.
    goto fin
)
```

* Verifica que el fichero exista.
* Si no existe, muestra mensaje y termina.

```bat
for /f "usebackq tokens=1,2,3" %%A in ("%~1") do (
    echo nombre:%%A apellido1:%%B apellido2:%%C
)
```

* `for /f` lee el fichero línea por línea.
* `usebackq` permite usar comillas para el nombre de archivo.
* `tokens=1,2,3` separa la línea en tres partes por defecto usando espacios.
* `%%A` = nombre, `%%B` = primer apellido, `%%C` = segundo apellido.
* Dentro del `do`, muestra cada línea en el formato pedido.

```bat
:fin
```

* Etiqueta final.

**Lógica geral:**\
Recebe um ficheiro, verifica que existe, e para cada linha `NOMBRE APELLIDO1 APELLIDO2` imprime “nombre:X apellido1:Y apellido2:Z”.

***

### 4) `decimal_a_binario.bat`

```bat
@echo off
chcp 65001 >nul
setlocal enabledelayedexpansion
```

* `setlocal` inicia un bloque local de variables.
* `enabledelayedexpansion` permite usar `!var!` para ver valores actualizados dentro de bucles.

```bat
REM Comentarios de autor y descripción...
```

* Solo documentación.

```bat
echo Introduzca el número a traducir a binario:
set /p "num="
```

* Pide el número decimal al usuario y lo guarda en `num`.

```bat
if "%num%"=="" (
    echo Debe introducir un número.
    goto fin
)
```

* Si el usuario no introduce nada, error y termina.

```bat
for /f "delims=0123456789" %%A in ("%num%") do (
    echo Debe ser un número entero positivo.
    goto fin
)
```

* Igual truco de antes: si hay caracteres que no sean 0-9, el `for` se ejecuta.
* Si entra, muestra error y termina.

```bat
set "n=%num%"
set "binario="
```

* Copia el número a `n`, que iremos dividiendo.
* Inicializa la cadena `binario` vacía.

```bat
:dividir
if !n! LEQ 1 (
    set "binario=!n!!binario!"
    goto mostrar
)
```

* Etiqueta del bucle de división.
* Si `n` es 0 o 1, este es el último bit; lo concatena delante de `binario` y pasa a mostrar el resultado.

```bat
set /a resto=n%%2
set /a n=n/2
set "binario=!resto!!binario!"
goto dividir
```

* Calcula `resto = n % 2`.
* Divide `n` entre 2 (división entera).
* Mete el resto al principio de la cadena `binario`.
* Repite el proceso.

```bat
:mostrar
echo El número decimal %num% es %binario% en binario.
```

* Muestra el resultado final.

```bat
:fin
endlocal
```

* Etiqueta final y fin del `setlocal`.

**Lógica geral:**

* Valida que a entrada é número.
* Vai dividindo por 2 e guardando os restos.
* Cada resto é um bit, colocado à esquerda da string.
* Quando `n` vira 0 ou 1, termina e mostra o binário completo.

***

Boa, agora ficou bem específico, gosto. Vamos por partes:

1. Te doy **ejemplos mejores** que puedes meter en tu capítulo (redirecciones, pipes, grep, find, bucles, funciones…).
2. Hago un **subcapítulo de `awk` bien explicado y fácil**, listo para GitBook.
3. Explico claro y práctico la **diferencia entre `useradd` y `adduser`**.

Todo en español, sin emojis.

***

### 1. Ejemplos mejores para tu capítulo

#### 1.1 Redirecciones: ejemplo “real” con logs

Imagina que quieres ver cuántas veces aparece la palabra `ERROR` en un log, pero quieres guardar la salida en un fichero y los errores en otro:

```bash
grep "ERROR" /var/log/syslog > errores.txt 2> errores_grep.txt
```

* `>` guarda las líneas que contienen `ERROR` en `errores.txt`.
* `2>` guarda posibles errores de `grep` (por ejemplo, permisos denegados) en `errores_grep.txt`.

Ejemplo de **añadir** sin machacar:

```bash
date >> historial_ejecuciones.log
ls /no_existe 2>> historial_errores.log
```

Cada vez que ejecutes eso:

* Se añade la fecha al final de `historial_ejecuciones.log`.
* Los errores de `ls` se acumulan en `historial_errores.log`.

***

#### 1.2 Pipes (tuberías): ejemplos útiles

**Ejemplo 1: ver los 10 comandos más usados**

```bash
history | awk '{print $2}' | sort | uniq -c | sort -nr | head
```

Explicación rápida:

* `history` → lista comandos ejecutados.
* `awk '{print $2}'` → coge la segunda columna (el comando).
* `sort | uniq -c` → agrupa y cuenta.
* `sort -nr` → ordena de mayor a menor.
* `head` → muestra los 10 primeros.

**Ejemplo 2: ver los 5 usuarios con más procesos**

```bash
ps aux | awk '{print $1}' | sort | uniq -c | sort -nr | head -n 5
```

* `ps aux` → lista todos los procesos.
* `awk '{print $1}'` → primer campo: usuario.
* El resto igual que antes.

***

#### 1.3 `grep`: ejemplos claros

**Buscar líneas que NO contienen una palabra**

```bash
grep -v "DEBUG" /var/log/syslog
```

Muestra todas las líneas del log que **no** tienen la palabra `DEBUG`.

**Contar cuántas veces aparece “ssh”**

```bash
grep -i "ssh" /var/log/auth.log | wc -l
```

* `-i` → ignora mayúsculas/minúsculas.
* `wc -l` → cuenta las líneas.

**Ver solo el nombre de los ficheros donde aparece una palabra**

```bash
grep -l "TODO" *.sh
```

Lista solo los scripts `.sh` que contienen la cadena `TODO`.

***

#### 1.4 `find`: ejemplos que sí se usan en la vida real

**Borrar logs antiguos (más de 7 días)**

```bash
find /var/log -type f -name "*.log" -mtime +7 -exec rm {} \;
```

* `-mtime +7` → modificados hace más de 7 días.
* `-exec rm {} \;` → borra cada archivo encontrado.

**Buscar ficheros grandes (más de 100 MB)**

```bash
find / -type f -size +100M 2>/dev/null
```

* `-size +100M` → tamaño mayor de 100 MB.
* `2>/dev/null` → oculta los mensajes de error (permisos, etc).

***

#### 1.5 Bucles con ejemplos mejorcitos

**Backup sencillo de varios directorios**

```bash
#!/bin/bash

DEST=/backup

for DIR in /etc /home /var/www; do
  NOMBRE=$(basename "$DIR")
  tar czf "$DEST/${NOMBRE}.tar.gz" "$DIR"
done
```

* Recorre una lista de directorios.
* Crea un `.tar.gz` por cada uno.

**`while`: leer fichero línea a línea**

```bash
#!/bin/bash

while read LINEA; do
  echo "Procesando: $LINEA"
done < lista_ficheros.txt
```

***

### 2. `awk` explicado bien y fácil (para tu GitBook)

Te dejo esto como si fuera una sección del capítulo:

***

#### 5.2 `awk` explicado de forma sencilla

Piensa en `awk` como una herramienta que:

* Lee **líneas** de texto.
* Divide cada línea en **campos/columnas**.
* Te deja decidir qué hacer con cada línea (imprimir algo, filtrar, sumar, etc).

Por defecto, el separador de campos es **espacio o tabulación**.

* `$0` → la línea completa.
* `$1` → primer campo.
* `$2` → segundo campo.
* `NF` → número de campos de la línea.
* `NR` → número de línea (1, 2, 3…).

***

**2.1 Ejemplo 1: imprimir solo una columna**

Fichero `alumnos.txt`:

```
Ana 8 9
Luis 5 7
Marta 10 9
```

Mostrar solo los nombres:

```bash
awk '{print $1}' alumnos.txt
```

Salida:

```
Ana
Luis
Marta
```

* `{print $1}` significa “para cada línea, imprime la columna 1”.

Mostrar nombre y segunda nota:

```bash
awk '{print $1, $3}' alumnos.txt
```

***

**2.2 Usar otro separador con `-F`**

Fichero `/etc/passwd` tiene líneas separadas por `:`:

```
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
```

Queremos solo el nombre de usuario y el directorio home:

```bash
awk -F: '{print $1, $6}' /etc/passwd
```

* `-F:` → indica que el separador es `:` (no espacio).
* `$1` → nombre de usuario.
* `$6` → directorio home.

***

**2.3 Filtrar líneas según una condición**

Mismo `alumnos.txt`:

```
Ana 8 9
Luis 5 7
Marta 10 9
```

Mostrar solo quienes tienen nota final (campo 2) mayor o igual que 8:

```bash
awk '$2 >= 8 {print $1, $2}' alumnos.txt
```

Explicación:

* `$2 >= 8` → condición.
* `{print $1, $2}` → acción cuando la condición se cumple.

Mostrar solo las líneas donde aparezca “Ana”:

```bash
awk '$1 == "Ana" {print $0}' alumnos.txt
```

* `$1 == "Ana"` → el primer campo es exactamente “Ana”.
* `$0` → línea completa.

***

**2.4 `awk` con patrones de texto (como mini grep)**

```bash
awk '/ERROR/ {print $0}' /var/log/syslog
```

* `/ERROR/` → patrón de texto (líneas que contienen “ERROR”).
* `{print $0}` → imprime la línea completa.

Es equivalente a:

```bash
grep "ERROR" /var/log/syslog
```

pero con `awk` luego puedes añadir más lógica.

***

**2.5 `BEGIN` y `END`: cabecera y resumen**

`BEGIN` se ejecuta **antes** de leer ninguna línea.\
`END` se ejecuta **después** de leer todas.

Ejemplo: calcular la media de la segunda columna:

```bash
awk 'BEGIN {suma=0; n=0}
     {suma += $2; n++}
     END {print "Media:", suma/n}' alumnos.txt
```

* En cada línea, suma el valor de `$2` y aumenta el contador.
* Al final, imprime la media.

***

**2.6 Variables especiales útiles**

* `NR` → número de línea (record).
* `NF` → número de campos en esa línea.

Ejemplo: mostrar solo líneas con más de 3 campos:

```bash
awk 'NF > 3 {print $0}' fichero.txt
```

Ejemplo: numerar líneas:

```bash
awk '{print NR ":", $0}' fichero.txt
```

***

**2.7 `awk` con pipes**

Puedes usar `awk` sin fichero, leyendo desde otro comando:

```bash
ps aux | awk '{print $1}' | sort | uniq -c | sort -nr | head
```

o:

```bash
ls -l | awk '{print $9}'      # imprime solo los nombres de fichero
```

***

Se quiser, isso já pode virar um subapartado do teu capítulo tipo:

> 5.2 Procesamiento de texto con `awk` (explicado paso a paso)

***

### 3. Diferencia entre `useradd` y `adduser`

Isto é muito importante e costuma cair em prova / exercício.

#### 3.1 Idea geral

Em **Debian/Ubuntu**:

* `useradd` → **comando de bajo nivel**, muy “seco”.
* `adduser` → **script de alto nivel**, más amigable e interactivo.

En muchas otras distros (CentOS, RHEL), `adduser` suele ser solo un enlace a `useradd` y se comportan casi igual. Pero en Debian/Ubuntu hay diferencia real.

***

#### 3.2 `useradd` (bajo nivel)

Características:

* No hace preguntas, no es interactivo.
* Si no le das opciones, puede **no crear el home**, no asignar shell correcta, etc.
* Es más “peligroso” para principiantes si no recuerdas las opciones.

Ejemplo “correcto” en Debian:

```bash
sudo useradd -m -s /bin/bash -G sudo,developers juan
sudo passwd juan
```

* `-m` → crea el directorio `/home/juan`.
* `-s /bin/bash` → shell de login.
* `-G` → grupos adicionales.
* Luego `passwd` para definir la contraseña.

Si haces solo:

```bash
sudo useradd juan
```

es probable que:

* no tenga directorio `/home/juan`,
* y venga con configuración mínima.

***

#### 3.3 `adduser` (alto nivel)

En Debian/Ubuntu, `adduser` es un **script en Perl** que:

* Te hace preguntas interactivamente (contraseña, nombre completo, etc.).
* **Siempre crea** el directorio home por defecto (`/home/usuario`).
* Copia los ficheros de `/etc/skel` (configuración por defecto).
* Asigna un shell razonable (por ejemplo `/bin/bash`).
* Suele ser la forma recomendada para crear usuarios “normales”.

Ejemplo:

```bash
sudo adduser juan
```

El sistema te preguntará:

* contraseña,
* nombre completo,
* etc.

Y al terminar ya tendrás:

* `/home/juan` creado,
* permisos correctos,
* ficheros iniciales (como `.bashrc`),
* shell configurada.

***

#### 3.4 Resumen que puedes poner en el GitBook

Puedes poner un recuadro de resumen así:

> **Resumen rápido**
>
> * Usa `adduser` cuando quieras **crear usuarios normales** en Debian/Ubuntu de forma fácil e interactiva.
> * Usa `useradd` cuando necesites un **control muy fino** (scripts automáticos, configuración personalizada), pero recuerda añadir opciones como `-m`, `-d`, `-s`, `-G`, etc.
> * En muchos sistemas tipo Red Hat, `adduser` es solo un alias de `useradd`, así que se comportan igual.

Perfeito. A seguir entrego o **capítulo inteiro revisado**, com **explicaciones mais claras**, **exemplos melhores**, `awk` refeito para iniciantes, e a **diferença entre `useradd` y `adduser`** integrada no capítulo.

Tudo está estruturado, limpo e pronto para colar no teu **GitBook**.

***

## Bloque 1 · Scripting en Linux (Bash)

## 1. Introducción: la shell y Bash

En GNU/Linux, la **shell** es el programa que interpreta los comandos que escribes y los envía al sistema. Existen varios tipos (`sh`, `csh`, `zsh`, `ksh`, `bash`), pero **Bash** es la más usada.

Características fundamentales:

* Linux es **case sensitive**: `LS` ≠ `ls`.
*   Estructura de comando:

    ```bash
    comando arg1 arg2 --opcion1 --opcion2
    ```
*   Si el programa no está en el `PATH`, hay que ejecutarlo indicando la ruta:

    ```bash
    ./programa
    ```
* Tipo de usuario según el prompt:
  * `$` usuario normal
  * `#` usuario root (administrador)

***

## 2. Comandos básicos de usuario

Comandos esenciales para moverse y consultar información:

* `ls` — Lista ficheros y directorios.\
  Ejemplo: `ls -l` muestra permisos, dueño y tamaño.
* `man` — Abre el manual de un comando.
* `pwd` — Indica el directorio actual.
* `cd` — Cambia de directorio.
* `echo` — Imprime texto.
* `cat` — Muestra contenido de un fichero completo.
* `more` — Muestra el contenido por páginas.
* `file` — Detecta el tipo de fichero.
* `touch` — Crea o actualiza un fichero.
* `rm` — Borra ficheros. Con `-r` borra directorios.
* `mkdir` — Crea un directorio.
* `rmdir` — Borra un directorio vacío.
* `cp` — Copia. Con `-r` copia directorios.
* `mv` — Mueve o renombra.
* `ln` — Crea enlaces duros o simbólicos.
* `date` — Muestra la fecha actual.

***

## 3. Redirecciones y tuberías

En Linux, todo programa usa flujos:

* `STDIN` (0): entrada.
* `STDOUT` (1): salida.
* `STDERR` (2): errores.

Controlarlos permite combinar comandos de forma potente.

### 3.1 Redirecciones de salida

* `>` sobrescribe el archivo destino.
* `>>` añade al final.
* `2>` redirige errores.
* `2>>` añade errores.

Ejemplos:

```bash
ls /etc > listado.txt
grep ERROR /var/log/syslog > errores.log 2> errores_grep.log
date >> historial.log
```

### 3.2 Redirección de entrada

Un comando lee datos desde un fichero:

```bash
tr a A < texto.txt
```

### 3.3 Tuberías (pipes)

Conectan comandos como etapas de un proceso:

```bash
comando1 | comando2
```

Ejemplos útiles:

```bash
history | awk '{print $2}' | sort | uniq -c | sort -nr | head
ps aux | awk '{print $1}' | sort | uniq -c | sort -nr | head
```

***

## 4. Filtros de texto importantes

Comandos que procesan texto línea a línea:

* `sort`: ordenar.
* `tr`: transformar caracteres.
* `head`: primeras líneas.
* `tail`: últimas líneas.
* `wc`: contar (`wc -l`, `wc -w`, `wc -c`).
* `cut`: extraer columnas.

Ejemplo:

```bash
cut -d: -f1 /etc/passwd      # nombres de usuario del sistema
```

***

## 5. Búsqueda y expresiones regulares: grep, awk, find

### 5.1 grep

`grep` busca patrones usando expresiones regulares.

Patrones básicos:

* `.` → un carácter cualquiera
* `*` → repetición
* `^` → inicio de línea
* `$` → final
* `[a-z]` → rango
* `[^0-9]` → negación

Ejemplos útiles:

```bash
grep -i error /var/log/syslog
grep -v DEBUG fichero.log
grep -l "TODO" *.sh
grep "ssh" /var/log/auth.log | wc -l
```

***

### 5.2 awk — Explicación clara y fácil (versión mejorada)

`awk` es un **procesador de texto por columnas**.\
Cada línea se divide en campos:

* `$0`: línea completa
* `$1`: primer campo
* `$2`: segundo campo
* `NF`: número de campos
* `NR`: número de línea

#### 5.2.1 Ejemplo básico: imprimir columnas

`alumnos.txt`:

```
Ana 8 9
Luis 5 7
Marta 10 9
```

```bash
awk '{print $1}' alumnos.txt     # nombres
awk '{print $1,$3}' alumnos.txt  # nombre + nota final
```

#### 5.2.2 Usar otro delimitador

`/etc/passwd` usa `:`:

```bash
awk -F: '{print $1,$6}' /etc/passwd   # usuario y home
```

#### 5.2.3 Filtrar por condición

```bash
awk '$2 >= 8 {print $1,$2}' alumnos.txt
awk '$1 == "Ana" {print $0}' alumnos.txt
```

#### 5.2.4 Buscar como grep, pero con lógica extra

```bash
awk '/ERROR/ {print $0}' /var/log/syslog
```

#### 5.2.5 BEGIN y END

Media de la segunda columna:

```bash
awk 'BEGIN{suma=0;n=0} {suma+=$2;n++} END{print "Media:", suma/n}' alumnos.txt
```

#### 5.2.6 Numerar líneas e imprimir solo las más largas

```bash
awk '{print NR":"$0}' fichero.txt
awk 'NF > 3 {print $0}' fichero.txt
```

#### 5.2.7 awk con pipes

```bash
ls -l | awk '{print $9}'        # nombres de fichero
ps aux | awk '{print $1}'       # usuarios con procesos
```

***

### 5.3 find

Busca ficheros por nombre, tipo, tamaño o fecha.

Ejemplos prácticos:

```bash
find /var/log -type f -name "*.log"
find / -type f -size +100M 2>/dev/null
find /var/log -type f -mtime +7 -exec rm {} \;
```

***

## 6. Usuarios, grupos, permisos y procesos

### 6.1 Gestión de usuarios y grupos

Comandos:

* `useradd`, `userdel`, `usermod`
* `groupadd`, `groupdel`, `groupmod`
* `passwd`, `gpasswd`

#### Diferencia clara: useradd vs adduser

**useradd (bajo nivel)**

* No hace preguntas.
* No crea home si no se le pide.
* Usado en scripts.\
  Ejemplo correcto:

```bash
sudo useradd -m -s /bin/bash -G sudo juan
sudo passwd juan
```

**adduser (alto nivel, Debian/Ubuntu)**

* Script interactivo.
* Siempre crea `/home/usuario`.
* Copia ficheros de `/etc/skel`.
* Recomendado para usuarios normales.

```bash
sudo adduser juan
```

***

### 6.2 Permisos y propietarios

* `chmod`: cambia permisos.
* `chown`: propietario.
* `chgrp`: grupo.
* `who`, `whoami`, `id`: información sobre usuarios.

***

### 6.3 Procesos

* `ps`, `top`, `pstree`
* `pgrep`, `pidof`
* `kill`, `killall`

Control de trabajos:

* `&`, `bg`, `fg`
* `jobs`
* `ctrl+z` pausa
* `ctrl+c` interrumpe
* `nohup` mantiene procesos
* `disown` los desvincula

***

## 7. Compresión y backups

* `gzip`, `bzip2`
* `tar` empaqueta y comprime

Ejemplos útiles:

```bash
tar czf backup.tar.gz /etc
gzip -dc archivo.gz | grep cadena
zgrep error log.gz
```

***

## 8. Primeros pasos con shell scripts

### 8.1 Primer script

```bash
#!/bin/bash
echo "Hola mundo"
```

Permisos:

```bash
chmod +x script.sh
./script.sh
```

### 8.2 Variables

```bash
NOMBRE="Ana"
echo "Hola $NOMBRE"
```

### 8.3 Variables de entorno

```bash
export VARIABLE
```

### 8.4 Entrada del usuario

```bash
read NOMBRE
```

### 8.5 Argumentos

```bash
$1 $2 $3
$0 $# $@ $*
```

Ejemplo:

```bash
echo "Nombre: $1"
```

### 8.6 Sustitución de comandos

```bash
FECHA=$(date)
```

***

## 9. Operaciones aritméticas

Comando `expr`:

```bash
expr 7 + 5
expr \( 7 + 5 \) \* 2
```

Sintaxis moderna:

```bash
((n = n + 1))
```

***

## 10. Control de flujo

### If

```bash
if [ "$N" -gt 10 ]; then
  echo "Mayor que 10"
fi
```

### Case

```bash
case $OPCION in
  1) echo "Uno" ;;
  2) echo "Dos" ;;
  *) echo "Otra cosa" ;;
esac
```

***

## 11. Bucles

### While

```bash
while [ "$N" -lt 10 ]; do
  echo "$N"
  ((N++))
done
```

### Until

```bash
until [ "$N" -ge 10 ]; do
  echo "$N"
  ((N++))
done
```

### For

```bash
for F in *.txt; do
  echo "Procesando $F"
done
```

### Select

```bash
PS3="Elige opción: "
select OPC in A B C; do
  echo "Elegido: $OPC"
done
```

***

## 12. Funciones y carga con source

### Función

```bash
suma() {
  echo $(($1 + $2))
}
```

### Variables locales

```bash
mi_funcion() {
  local A=10
}
```

### Cargar funciones desde otro archivo

```bash
source funciones.sh
```

***

