---
layout: lecture
title: El entorno de línea de comandos
date: '2019-02-12'
ready: 'true'
video:
  aspect: '56.25'
  id: e8BO_dYxk5c
---

En esta clase repasaremos algunas formas de mejorar el trabajo en el shell. Ya estuvimos trabajando con el intérprete de comandos, pero fundamentalmente centrados en ejecutar distintos comandos. Ahora veremos cómo ejecutar varios procesos al mismo tiempo sin perder rastro de ellos, cómo pausar o detener un proceso específico y cómo hacer que un proceso se ejecute en segundo plano.

También veremos diferentes formas de mejorar el shell y otras herramientas definiendo nuevos comandos mediante "alias", y cómo personalizar las aplicaciones mediante archivos de configuración llamados _dotfiles_. Ambas estrategias nos permitirán ahorrar tiempo, por ejemplo, al utilizar configuraciones similares en todos los equipos donde trabajamos y sin tener que escribir líneas de comandos extensos. Veremos también cómo trabajar con máquinas remotas utilizando SSH.

# Control de trabajos

En ciertas ocasiones necesitarás interrumpir un proceso que aún se está ejecutando, por ejemplo, si un comando tarda demasiado en completarse (como un `find` con una estructura de directorios muy grande para buscar). La mayoría de las veces, podés pulsar `Ctrl-C` y el comando se detendrá. Pero, ¿cómo funciona esto realmente y por qué a veces no alcanza para detener el proceso?

## Matar un proceso

El intérprete de comandos utiliza un mecanismo de comunicación UNIX llamado *señal* para comunicar información a los procesos. Cuando un proceso recibe una señal, pausa su ejecución, atiende la señal recibida y es posible que cambie el flujo de ejecución en función de la información que llega con la señal. Por esta razón, las señales son *interrupciones de software* .

En nuestro caso, pulsar `Ctrl-C` hace que el shell envíe una señal `SIGINT` al proceso que se está ejecutando.

El siguiente es un ejemplo mínimo de un programa Python que captura `SIGINT` y lo ignora, con lo cual no es posible detenerlo. Para matar este programa deberemos enviar una señal llamada `SIGQUIT`. Esto se realiza pulsando `Ctrl-El siguiente es un ejemplo mínimo de un programa Python que captura `SIGINT` y lo ignora, con lo cual no es posible detenerlo. Para matar este programa deberemos enviar una señal llamada `SIGQUIT`. Esto se realiza pulsando  .

```python
#!/usr/bin/env python
import signal, time

def handler(signum, time):
    print("\nHe recibido SIGINT, pero no me voy a detener")

signal.signal(signal.SIGINT, handler)
i = 0
while True:
    time.sleep(.1)
    print("\r{}".format(i), end="")
    i += 1
```

Esto es lo que sucede si enviamos `SIGINT` dos veces a este programa, seguido de `SIGQUIT`. En la terminal, el símbolo `^` suele representar una pulsación de la tecla `Ctrl`.

```
$ python sigint.py
24^C
He recibido SIGINT, pero no me voy a detener
26^C
He recibido SIGINT, pero no me voy a detener
30^\[1]    39913 quit       python sigint.py
```

Si bien las señales `SIGINT` y `SIGQUIT` están asociadas generalmente a la finalización de un proceso, una señal más genérica para solicitar que un proceso finalice "correctamente" es la señal `SIGTERM`. Para enviar esta señal podemos usar el comando [`kill`](http://man7.org/linux/man-pages/man1/kill.1.html), con la sintaxis `kill -TERM <PID>` .

## Pausar procesos y ejecutarlos en segundo plano

Las señales pueden hacer otras cosas más allá de matar un proceso. Por ejemplo, `SIGSTOP` pone un proceso en pausa. Si presionás `Ctrl-Z` en el terminal, el intérprete de comandos enviará una señal `SIGTSTP`, abreviatura de "Terminal Stop" (es decir, la versión del terminal de `SIGSTOP`).

Para reanudar el proceso que está pausado en primer plano o en segundo plano, hay que escribir el comando [`fg`](http://man7.org/linux/man-pages/man1/fg.1p.html) o [`bg`](http://man7.org/linux/man-pages/man1/bg.1p.html), respectivamente.

El comando [`jobs`](http://man7.org/linux/man-pages/man1/jobs.1p.html) muestra todos los trabajos que aún no han finalizado en la terminal actual. Cuando sea necesario hacer referencia a un proceso en particular, podés indicarlo mediante su "process id" o _pid_ (el comando [`pgrep`](http://man7.org/linux/man-pages/man1/pgrep.1.html) devuelve el pid de un proceso dado su nombre). De manera más intuitiva, también es posible hacer referencia a un proceso en curso mediante el símbolo de porcentaje seguido del número de trabajo (que se muestra en los `jobs`). Para indicar que se desea operar sobre el último trabajo en segundo plano, puede utilizarse la variable de entorno `$!`.

Otro dato importante es que el sufijo `&` en una línea de comando hará que el mismo se ejecute en segundo plano, volviendo al _prompt_ y permitiéndote continuar con las tareas. Hay que tener en cuenta que la salida estándar del proceso iniciado continuará siendo la terminal, por lo que es posible que aparezcan líneas de salida entre nuestros comandos (para evitarlo, redireccioná la salida hacia un archivo).

Para poner en segundo plano un programa que ya se está ejecutando, pulsá `Ctrl-Z` y luego escribí el comando `bg`. Tené en cuenta que los procesos en segundo plano siguen perteneciendo a la terminal actual, y por ello morirán si cerrás la ventana de la terminal (el shell envía la señal `SIGHUP` a cada proceso hijo). Para evitar que eso suceda, cuando lances un proceso que querés que se mantenga en fondo incluso luego de cerrar la terminal, escribí la palabra clave {a4}`nohup`{/a4} antes del comando que deseas ejecutar (el proceso `nohup` captura la señal `SIGHUP` y la ignora), o bien podes ejecutar el comando {code7}disown{/code7} para evitar que un proceso existente muera al cerrar la terminal. Alternativamente, puede usar un multiplexor de terminal como veremos en la siguiente sección.

En la siguiente sesión de ejemplo pueden verse algunos de estos conceptos.

```
$ sleep 1000
^Z
[1]+ 18653 Detenido  sleep 1000

$ nohup sleep 2000 &
[2] 18745
nohup: se descarta la entrada y se añade la salida a 'nohup.out'

$ jobs
[1]+  Detenido      sleep 1000
[2]-  Ejecutando    nohup sleep 2000

$ bg %1
[1]- 18653 Continuando  sleep 1000

$ jobs
[1]-  Ejecutando    sleep 1000
[2]+  Ejecutando    nohup sleep 2000

$ kill -STOP %1
[1]+ 18653 Detenido  sleep 1000

$ jobs
[1]+  Detenido      sleep 1000
[2]-  Ejecutando    nohup sleep 2000

$ kill -SIGHUP %1
[1]+ 18653 Colgar (hangup)     sleep 1000

$ jobs
[2]+  Ejecutando    nohup sleep 2000

$ kill -SIGHUP %2

$ jobs
[2]+  Ejecutando    nohup sleep 2000

$ kill %2
[2]+ 18745 Terminado  nohup sleep 2000

$ jobs
```

La señal `SIGKILL` es especial, pues no puede ser capturada por el proceso y siempre hará que el mismo muera de inmediato. Sin embargo, puede tener efectos secundarios negativos, tal como dejar en ejecución aquellos procesos que fueron iniciados por el proceso padre (procesos huérfanos).

Podés obtener más información sobre éstas y otras señales [aquí](https://en.wikipedia.org/wiki/Signal_(IPC)) o bien escribiendo [`man signal`](http://man7.org/linux/man-pages/man7/signal.7.html) o `kill -t`.

# Multiplexores de terminal

Muchas veces la interfaz de línea de comandos resulta insuficiente para ejecutar más de un proceso a la vez. Por ejemplo, quizás quieras tener la pantalla dividida en dos, con un editor de texto a la izquierda y la ejecución de un proceso a la derecha. Si bien esto puede lograrse abriendo nuevas ventanas de terminal, el uso de un multiplexor de terminal es una solución más versátil.

Los multiplexores de terminal, tal como [`tmux`](http://man7.org/linux/man-pages/man1/tmux.1.html), permiten trabajar con varias ventanas de terminal, separándolas en paneles y pestañas, permitiendo interactuar con múltiples sesiones de shell a la vez. Además, permiten "desconectarse" de una sesión de terminal y volver a "conectarse" en algún momento posterior sin perder el estado de la pantalla ni detener los procesos en curso. Esto es sumamente útil cuando trabajamos con máquinas remotas, ya que anula la necesidad de usar `nohup` y trucos similares.

El multiplexor más popular en estos días es [`tmux`](http://man7.org/linux/man-pages/man1/tmux.1.html). `tmux` es altamente configurable y mediante distintas combinaciones de teclas es posible crear múltiples pestañas y paneles, y cambiar de una a otra rápidamente.

`tmux` espera que conozcas sus combinaciones de teclas, y todas tienen la forma `<Cb> x` donde eso significa presionar `Ctrl+b`, liberar y luego presionar `x`. `tmux` tiene la siguiente jerarquía de objetos:

- **Sesiones**: una sesión es un espacio de trabajo independiente con una o más ventanas

    - `tmux` inicia una nueva sesión.
    - `tmux new -s NOMBRE` inicia una nueva sesión con ese nombre.
    - `tmux ls` muestra las sesiones existentes
    - Dentro de `tmux`, pulsar `<Cb> d` nos desconecta de la sesión actual
    - `tmux a` reconecta a la última sesión. Podés usar el parámetro `-t` para conectarte a una sesión diferente (conociendo su nombre)

- **Ventanas**: equivalentes a las pestañas en editores o navegadores web, son partes visualmente separadas de la misma sesión

    - `<Cb> c` crea una nueva ventana. Para cerrarla, simplemente hay que pulsar `<Cd>`
    - `<C-b> N` ir a la ventana *N*. Las ventanas se numeran en orden de creación
    - `<C-b> p` va a la ventana anterior
    - `<C-b> n` va a la ventana siguiente
    - `<C-b> ,` cambia el nombre de la ventana actual
    - `<C-b> w` lista las ventanas abiertas

- **Paneles**: al igual que las divisiones en el editor _vim_, los paneles nos permiten tener múltiples shells en la misma pantalla.

    - `<C-b> "` divide horizontalmente al panel actual
    - `<C-b> %` divide verticalmente al panel actual
    - `<C-b> <direction>` mueve al panel en la *dirección* especificada. Dirección aquí significa teclas de flecha.
    - `<C-b> z` cambia el zoom del panel actual
    - `<C-b> [` inicia el desplazamiento hacia atrás. Luego puede presionar `<space>` para comenzar a seleccionar texto y `<enter>` para copiar esa selección.
    - `<C-b> <space>` Cicla entre distintas configuraciones de panel.

Para más información, [aquí hay](https://www.hamvocke.com/blog/a-quick-and-easy-guide-to-tmux/) un rápido tutorial sobre `tmux` y [este artículo](http://linuxcommand.org/lc3_adv_termmux.php) tiene una explicación más detallada que cubre el antiguo comando `screen`. Conviene familiarizarse también con el comando <a data-md-type="link" href="http://man7.org/linux/man-pages/man1/screen.1.html">`screen`</a>, pues es el que viene instalado por defecto en la mayoría de los sistemas UNIX.

# Atajos o "alias" de comandos

Escribir comandos largos que involucran muchas banderas u opciones detalladas suele tornarse aburrido. Por ello, la mayoría de los shells admiten crear *alias* de comandos. Un alias de shell es una forma abreviada de otro comando que el intérprete reemplazará automáticamente cada vez que se escribe. Por ejemplo, un alias en bash tiene la siguiente estructura:

```bash
alias alias_name="command_to_alias arg1 arg2"
```

Hay que tener en cuenta que no va espacio antes ni después del signo igual `=`. [`alias`](http://man7.org/linux/man-pages/man1/alias.1p.html) es un comando de shell que toma un solo argumento.

Los alias son convenientes por múltiples razones:

```bash
# para hacer atajos para parámetros que se usan frecuentemente
alias ll="ls -lh"

# para acortar comandos habituales
alias gs="git status"
alias gc="git commit"
alias v="vim"

# para cuando uno suele escribir mal los comandos
alias sl=ls

# para redefinir comandos existentes
alias mv="mv -i"           # -i pregunta antes de pisar archivos
alias mkdir="mkdir -p"     # -p crea directorios padre automáticamente
alias df="df -h"           # -h muestra los tamaños en formato mb/gb

# los alias pueden incluso reutilizarse
alias la="ls -A"
alias lla="la -l"

# para evitar utilizar un alias (y forzar la ejecución del comando)
# antecederlo con el símbolo \
\ls
# o bien desactivarlo con el comando unalias
unalias la

# para conocer cómo está definido un alias, usar el comando alias
alias ll
# mostrará ll='ls -lh'
```

Hay que advertir que los alias no se mantienen automáticamente. Para que un alias sea persistente, hay que incluirlo en los archivos de inicio del shell, por ejemplo en `.bashrc` o `.zshrc`, que es lo que veremos a continuación.

# Archivos de configuración o "Dotfiles"

Muchos programas se configuran utilizando archivos de texto plano conocido como *dotfiles*, debido a que los nombres de los archivos comienzan con un `.` (dot significa punto en inglés). Los archivos cuyo nombre comienza con un punto (por ejemplo `~/.vimrc`), de manera estándar no se muestran al listar un directorio con `ls`.

Los shells son un ejemplo de programas configurados con dichos archivos. Al iniciarse, tu shell leerá muchos archivos para cargar su configuración y todo el proceso puede ser bastante complejo. [Aquí](https://blog.flowblok.id.au/2013-02/shell-startup-scripts.html) encontrarás un excelente recurso sobre el tema.

Para `bash` , editar tu `.bashrc` o `.bash_profile` funcionará en la mayoría de los sistemas. Allí podés incluir comandos que querés que se ejecuten al inicio, como los alias que acabamos de describir o modificaciones a tu variable de entorno `PATH`. De hecho, muchos programas te pedirán que incluyas una línea como `export PATH="$PATH:/path/to/program/bin"` en el archivo de configuración de tu shell para que se puedan encontrar sus archivos binarios (ejecutables).

Algunos ejemplos de herramientas que se pueden configurar mediante dotfiles son:

- `bash` - `~/.bashrc`, `~/.bash_profile`
- `git` - `~/.gitconfig`
- `vim` - `~/.vimrc` y el directorio `~/.vim`
- `ssh` - `~/.ssh/config`
- `tmux` - `~/.tmux.conf`

¿Cómo debes organizar tus dotfiles? Deben estar en su propio directorio, bajo control de versiones, y **enlazados simbólicamente** a su lugar usando un script. Esto tiene como beneficios:

- **Instalación fácil**: si iniciás sesión en una nueva máquina, aplicar tus personalizaciones llevará solo un minuto.
- **Portabilidad**: tus herramientas funcionarán de la misma manera en todas partes.
- **Sincronización**: puedes actualizar tus archivos de configuración en cualquier lugar y mantenerlos todos sincronizados.
- **Seguimiento de cambios**: probablemente mantendrás tus archivos de configuración durante toda tu carrera programando, y tener un historial de versiones es bueno para proyectos de larga duración.

¿Qué deberías poner en tus dotfiles? Podés obtener información sobre la configuración de tus herramientas leyendo la documentación en línea o las [páginas de manual](https://en.wikipedia.org/wiki/Man_page) . Otra buena manera es buscar en Internet publicaciones en blogs sobre programas específicos, donde los autores cuentan sus personalizaciones preferidas. Otra forma mas de aprender acerca de las personalizaciones es mirar los dotfiles de otras personas: podés encontrar un muchísimos [repositorios de dotfiles](https://github.com/search?o=desc&q=dotfiles&s=stars&type=Repositories). podés ver el más popular [aquí](https://github.com/mathiasbynens/dotfiles) (no obstante te recomendamos no copiar ciegamente las configuraciones). [Aquí](https://dotfiles.github.io/) hay otro buen recurso sobre el tema.

Todos los instructores de la clase tienen sus dotfiles accesibles   públicamente en GitHub: [Anish](https://github.com/anishathalye/dotfiles) , [Jon](https://github.com/jonhoo/configs) , [Jose](https://github.com/jjgo/dotfiles) .

## Portabilidad

Un problema común con los archivos de configuración es que las configuraciones pueden no funcionar cuando se trabaja con varias máquinas, por ejemplo, si tienen diferentes sistemas operativos o shells. A veces, también podés querer que alguna configuración se aplique solo en una máquina determinada.

Hay algunos trucos para facilitar esto. Si el archivo de configuración lo soporta, usa el equivalente de sentencias if para aplicar personalizaciones específicas de la máquina. Por ejemplo, tu shell podría tener algo como:

```bash
if [[ "$(uname)" == "Linux" ]]; then {hacer_algo}; fi

# Chequear antes de utilizar características especiales del shell
if [[ "$SHELL" == "zsh" ]]; then {hacer_algo}; fi

# También puede ser específico por máquina
if [[ "$(hostname)" == "MiServidor" ]]; then {hacer_algo}; fi
```

Si el archivo de configuración lo soporta, podés usar "includes". Por ejemplo, `~/.gitconfig` podría contener:

```
[include]
    path = ~/.gitconfig_local
```

Y luego en cada máquina, `~/.gitconfig_local` puede contener configuraciones específicas para esa máquina. Incluso podrías utilizar repositorios distintos para realizar el seguimiento de las configuraciones específicas de cada máquina.

Esta idea es útil también si querés que diferentes programas compartan algunas configuraciones. Por ejemplo, si queŕes que `bash` y `zsh` compartan el mismo conjunto de alias, podés escribirlos en `.aliases` y tener el siguiente bloque en ambos:

```bash
# Verificar si existe ~/.aliases y ejecutar sus comandos
if [ -f ~/.aliases ]; then
    source ~/.aliases
fi
```

# Máquinas Remotas

Se ha vuelto cada vez más común que los programadores usen servidores remotos en su trabajo diario. Si necesitás usar servidores remotos para implementar software de back-end, o necesitás un servidor con mayos capacidad de cómputo, terminarás usando un Secure Shell (SSH). Como con la mayoría de las herramientas vistas, SSH es altamente configurable, por lo que vale la pena conocerlo.

Para conectarse remotamente a un servidor con `ssh`, ejecutás el siguiente comando

```bash
ssh foo@bar.mit.edu
```

Aquí estamos tratando de conectarnos mediante ssh  al servidor `bar.mit.edu`, como el usuario `foo`. El servidor se puede especificar tanto con una URL (en el ejemplo, `bar.mit.edu`), como con una dirección IP (por ej.`foobar@192.168.1.42`). Más adelante veremos que si modificamos el archivo de configuración de ssh, podés conectarte simplemente usando algo como `ssh bar` .

## Ejecutando comandos

Una característica de `ssh` que muchas veces se pasa por alto, es la capacidad de ejecutar comandos directamente. `ssh foobar@servidor ls` ejecutará `ls` en el directorio home del usuario foobar en la máquina remota. Además funciona con tuberías, por lo que `ssh foobar@servidor ls | grep PATTERN` ejecutará grep localmente sobre la salida del `ls` ejecutado en la máquina remota, y `ls | ssh foobar@servidor grep PATTERN` ejecutará grep en la máquina remota sobre la salida del `ls` ejecutado localmente.

## Claves SSH

La autenticación basada en claves utiliza la criptografía de clave pública para asegurar al servidor que el cliente posee la clave privada secreta, sin revelar dicha clave. De esta manera, no necesitás volver a ingresar tu contraseña cada vez. Sin embargo, la clave privada (a menudo `~/.ssh/id_rsa` y más recientemente `~/.ssh/id_ed25519` ) es efectivamente tu contraseña, así que cuidala como tal.

### Generación de Claves

Para generar el par de claves debés ejecutar [`ssh-keygen`](http://man7.org/linux/man-pages/man1/ssh-keygen.1.html).

```bash
ssh-keygen -o -a 100 -t ed25519 -f ~/.ssh/id_ed25519
```

Deberías elegir una frase de contraseña para evitar que si alguien obtiene tu clave privada, no pueda usarla para acceder a otros servidores que utilicen autenticación basada en claves. Podés usar [`ssh-agent`](http://man7.org/linux/man-pages/man1/ssh-agent.1.html) o [`gpg-agent`](https://linux.die.net/man/1/gpg-agent) para no tener que escribir cada vez la frase de contraseña.

Si alguna vez configuraste el envío a GitHub usando claves SSH, entonces probablemente hayas realizado los pasos descriptos [aquí](https://help.github.com/articles/connecting-to-github-with-ssh/) y ya tengas un par de claves válido. Para verificar si tenés una frase de contraseña y validarla, podés ejecutar `ssh-keygen -y -f /path/a/la/clave` .

### Autenticación basada en claves

`ssh` buscará en `.ssh/authorized_keys` para determinar qué clientes debe permitir el acceso. Para copiar la clave pública a otra máquina, podés usar:

```bash
cat .ssh/id_ed25519.pub | ssh foobar@maquina-remota 'cat >> ~/.ssh/authorized_keys'
```

Es mas sencillo si está disponible `ssh-copy-id`:

```bash
ssh-copy-id -i .ssh/id_ed25519.pub foobar@maquina_remota
```

## Copiando archivos con SSH

Hay varias formas de copiar archivos con ssh:

- `ssh+tee`, la más simple es ejecutar el comando `ssh` utilizando el flujo de entrada de la sigueinte manera:  `cat localfile | ssh remote_server tee serverfile` . Recuerda que [`tee`](http://man7.org/linux/man-pages/man1/tee.1.html) permite escribir su flujo de entrada en un archivo.
- [`scp`](http://man7.org/linux/man-pages/man1/scp.1.html), cuando hay que copiar gran cantidad de archivos o directorios, es mucho mas conveniente usar el comando `scp` (secure copy) ya que permite hacer copias  recursivas sobre las rutas o directorios. La sintaxis es `scp path/al/archivo_local máquina_remota:path/al/archivo_remoto`
- [`rsync`](http://man7.org/linux/man-pages/man1/rsync.1.html) mejora a `scp` verificando diferencias entre archivos locales y remotos, y evitar así copiar los que son idénticos. También proporciona un control más detallado sobre enlaces simbólicos, permisos, y tiene características adicionales como el parámetro `--partial` que permite reanudar una copia interrumpida previamente. `rsync` tiene una sintaxis similar a `scp` .

## Reenvío de puertos

En muchos casos te encontrarás con software que escucha en puertos específicos en la máquina. Cuando esto sucede en tu máquina local, podés escribir `localhost:PORT` o `127.0.0.1:PORT` , pero ¿que hacés en el caso de un servidor remoto que no tiene sus puertos accesibles directamente a través de la red / internet?

Esto se llama *reenvío de puertos* y puede ser de dos tipos: reenvío de puertos locales y reenvío de puertos remotos (mirá las imágenes para obtener más detalles, crédito de las imágenes a [esta publicación de StackOverflow](https://unix.stackexchange.com/questions/115897/whats-ssh-port-forwarding-and-whats-the-difference-between-ssh-local-and-remot) ).

**Reenvío de puerto local**![Local Port Forwarding](https://i.stack.imgur.com/a28N8.png%C2%A0 "Local Port Forwarding")

**Reenvío de puerto remoto**![Remote Port Forwarding](https://i.stack.imgur.com/4iK3b.png%C2%A0 "Remote Port Forwarding")

El escenario más común es el reenvío de puertos local, donde un servicio en la máquina remota escucha en un puerto y querés vincular un puerto en tu máquina local para reenviarlo al puerto remoto. Por ejemplo, si ejecutamos `jupyter notebook` en el servidor remoto que escucha el puerto `8888`, para reenviar desde dicho puerto al puerto local `9999`, haríamos `ssh -L 9999:localhost:8888 foobar@remote_server` y luego conectarnos a `locahost:9999` en nuestra máquina local.

## Configuración de SSH

Hemos cubierto muchos argumentos que podemos pasar. Una alternativa tentadora es crear alias de shell como

```bash
alias mi_servidor="ssh -i ~/.id_ed25519 --port 2222 - L 9999:localhost:8888 foobar@servidor_remoto"
```

Sin embargo, hay una mejor alternativa usando `~/.ssh/config` .

```bash
Host vm
    User foobar
    HostName 172.16.174.141
    Port 2222
    IdentityFile ~/.ssh/id_ed25519
    RemoteForward 9999 localhost:8888

# Las configuraciones también aceptan comodines
Host *.mit.edu
    User foobaz
```

Una ventaja adicional de usar el archivo `~/.ssh/config` sobre los alias es que otros programas como `scp` , `rsync` , `mosh` , etc, también pueden leerlo y convertir las configuraciones en los parámetros correspondientes.

Ten en cuenta que el archivo `~/.ssh/config` puede considerarse un dotfile, y en general está bien que lo incluyas con el resto de tus dotfiles. Sin embargo, si lo publicás, debés pensar en la información que potencialmente estás proporcionando a extraños en Internet: direcciones de tus servidores, nombres de usuarios, puertos abiertos, etc. Esto puede facilitar algunos tipos de ataques, así que ten cuidado al compartir tu configuración SSH.

La configuración del lado del servidor generalmente se especifica en `/etc/ssh/sshd_config` . Allí podés realizar cambios como deshabilitar la autenticación basada contraseñas, cambiar el puerto de escucha, habilitar el reenvío X11, etc. Además, las configuraciones pueden ser en base a cada usuario.

## Misceláneas

Un problema común cuando estamos trabajando en un servidor remoto son las desconexiones debidas a apagados, activación del modo hibernación o suspensión o cambios en la red. Además, si uno tiene una conexión con un retardo significativo, usar ssh puede ser bastante frustrante. [Mosh](https://mosh.org/) , el shell móvil, es una alternativa que puede mejorar la experiencia al usar ssh permitiendo conexiones móviles, conectividad intermitente y proporcionando eco local inteligente.

A veces, es conveniente montar un directorio remoto. [sshfs](https://github.com/libfuse/sshfs) puede montar localmente un directorio de un servidor remoto, y así luego podés usar un editor local.

# Shells &amp; Entornos de trabajo (Frameworks)

Durante las descripción de las herramienta de shell y las secuencias de comandos, hemos utilizado el shell `bash` porque es, con mucho, el shell más ubicuo y la mayoría de los sistemas lo tienen como opción predeterminada. Sin embargo, no es la única opción.

Por ejemplo, el shell `zsh` es un superconjunto de `bash` y proporciona muchas funciones prácticas listas para usar, como:

- Globbing más inteligente, `**`
- Expansión de comodines / globbing en línea
- Corrección de escritura
- Mejor autocompletado / selección
- Expansión de rutas ( `cd /u/lo/b` se expandirá como `/usr/local/bin` )

Los **frameworks** también pueden mejorar tu shell. Algunos frameworks populares son [prezto](https://github.com/sorin-ionescu/prezto) u [oh-my-zsh](https://github.com/robbyrussll/oh-my-zsh) , y otros más pequeños que se centran en características específicas como [zsh-syntax-highlighting](https://github.com/zsh-users/zsh-syntax-highlighting) o [zsh-history-substring-search](https://github.com/zsh-users/zsh-history-substring-search) . Shells como [fish](https://fishshell.com/) incluyen muchas de estas características fáciles de usar por defecto. Algunas de estas características incluyen:

- Indicador o Prompt a la derecha
- Resaltado de sintaxis de los comandos
- Búsqueda de subcadenas en el historial
- Completado de parámetros basado en páginas de manual
- Autocompletado más inteligente
- Temas de indicadores o prompt

Una cosa a tener en cuenta al usar estos frameworks es que pueden ralentizar su shell, especialmente si el código que ejecutan no está optimizado correctamente o es demasiado código. Siempre podés hacer un análisis de rendimiento y deshabilitar las funciones que no uses con frecuencia o que no valga la pena teniendo en cuenta la velocidad.

# Emuladores de terminal

Junto con la personalización de su shell, vale la pena pasar un tiempo pensando en que **emulador de terminal** elejir y su configuración. Hay muchos emuladores de terminal por ahí (aquí hay una [comparación](https://anarc.at/blog/2018-04-12-terminal-emulators-1/) ).

Dado que puedes pasar de cientos a miles de horas en tu terminal, vale la pena mirar su configuración. Algunos de los aspectos que podés modificar en tu terminal incluyen:

- Elección de la tipografía
- Esquema de colores
- Atajos de teclado
- Soporte de Pestañas / Paneles
- Configuración de desplazamiento hacia atrás
- Rendimiento (algunos terminales más nuevos como [Alacritty](https://github.com/jwilm/alacritty) o [kitty](https://sw.kovidgoyal.net/kitty/) aprovechan la  aceleración por GPU).

# Ejercicios

## Control de trabajos

1. Por lo que hemos visto, podemos usar los comandos `ps aux | grep` para obtener los pids de nuestros trabajos y luego matarlos, pero hay mejores maneras de hacerlo. Inicie un trabajo `sleep 10000` en una terminal, envíalo a segundo plano con `Ctrl-Z` y continúa su ejecución con `bg`. Ahora usa [`pgrep`](http://man7.org/linux/man-pages/man1/pgrep.1.html) para encontrar su pid y [`pkill`](http://man7.org/linux/man-pages/man1/pgrep.1.html) para matarlo sin tener que escribir el pid en sí. (Sugerencia: use los parámetros `-af`).

2. Digamos que no deseas comenzar un proceso hasta que se complete otro, ¿cómo lo harías? En este ejercicio, nuestro proceso limitante será `sleep 60 &` . Una forma de lograr esto es usar el comando [`wait`](http://man7.org/linux/man-pages/man1/wait.1p.html) . Intenta iniciar el comando sleep y `ls`, pero haciendo que este espere hasta que finalice el proceso en segundo plano.

    Sin embargo, esta estrategia no funcionará si comenzamos en una sesión bash diferente, ya que `wait` solo funciona para procesos hijos. Una característica que no discutimos en las notas es que el estado de salida del comando `kill` será cero en caso de éxito, y distinto de cero en caso contrario. `kill -0` no envía una señal pero tendrá un estado de salida distinto de cero si el proceso no existe. Escribe una función bash llamada `pidwait` que tome un pid y espere hasta que dicho proceso se complete. Deberás usar`sleep` para evitar desperdiciar CPU innecesariamente

## Multiplexor de Terminales

1. Seguí este [tutorial](https://www.hamvocke.com/blog/a-quick-and-easy-guide-to-tmux/) de `tmux`, y luego aprendé cómo hacer algunas personalizaciones básicas siguiendo [estos pasos](https://www.hamvocke.com/blog/a-quick-and-easy-guide-to-tmux/) .

## Atajos o Alias

1. Creá un alias `dc` que ejecute `cd` para cuando lo escribís incorrectamente.

2. Ejecutá `history | awk '{$1="";print substr($0,2)}' | sort | uniq -c | sort -n | tail -n 10` para obtener tus 10 comandos más utilizados y considerá escribir alias más cortos para ellos. Nota: esto funciona para Bash; si usás ZSH, utilizá `history 1` en lugar de solo `history` .

## Archivos de configuración o Dotfiles

Vamos a ponerte al día con los archivos de puntos.

1. Creá un directorio para tus archivos de configuración y configura el control de versiones.
2. Agregue una configuración para al menos un programa, por ejemplo tu shell, con algunas personalizaciones (para empezar, puede ser algo tan simple como personalizar tu indicador de shell redefiniendo `$PS1`).
3. Por a punto un método para instalar tus archivos de configuración rápidamente (y sin esfuerzo manual) en una máquina nueva. Esto puede ser tan simple como un script de shell que ejecute `ln -s` para cada archivo, o podés usar alguna [utilidad especializada](https://dotfiles.github.io/utilities/).
4. Prueba tu script de instalación en una máquina virtual nueva.
5. Migra todas las configuraciones actuales de tus herramientas  a tu repositorio de archivos de configuración.
6. Publica tus archivos de configuración en GitHub.

## Máquinas remotas

Instala una máquina virtual Linux (o usa alguna que ya exista) para este ejercicio. Si no estás familiarizado con las máquinas virtuales, podés chequear [este](https://hibbard.eu/install-ubuntu-virtual-box/) tutorial para instalar una.

1. Ve a `~/.ssh/` y verificá si ya tenés un par de claves SSH allí. Si no, generalo con `ssh-keygen -o -a 100 -t ed25519` . Es recomendable que uses una contraseña y `ssh-agent`, encontrarás más información [aquí](https://www.ssh.com/ssh/agents) .
2. Editá `.ssh/config` para tener una entrada como la siguiente

```bash
Host vm
    User el_nombre_del_usuario_va_aquí
    HostName la_dirección_ip_va_aquí
    IdentityFile ~/.ssh/id_ed25519
    RemoteForward 9999 localhost:8888
```

1. Usá `ssh-copy-id vm` para copiar la clave ssh al servidor.
2. Iniciá un servidor web en la VM ejecutando `python -m http.server 8888`. Accedá al servidor web de la VM escribiendo `http://localhost:9999` en el navegador de tu máquina.
3. Editá la configuración de tu servidor SSH ejecutando `sudo vim /etc/ssh/sshd_config` y deshabilitá la autenticación mediante contraseñas modificando el valor de `PasswordAuthentication`. Deshabilitá el inicio de sesión como usuario root modificando el valor de `PermitRootLogin`. Reiniciá el servicio `ssh` con `sudo service sshd restart`. Intenta ingresar con ssh nuevamente.
4. (Desafío) Instalá [`mosh`](https://mosh.org/) en la VM y establecé una conexión. Luego, desconectá el adaptador de red del servidor / VM. ¿Puede Mosh recuperarse adecuadamente?
5. (Desafío) Buscá que hacen los parámetros `-N` y `-f` en `ssh` y dilucidá el comando para lograr el reenvío de puertos en segundo plano.
