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

`tmux` expects you to know its keybindings, and they all have the form `<C-b> x` where that means press `Ctrl+b` release, and the press `x`. `tmux` has the following hierarchy of objects:

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

# Dotfiles

Many programs are configured using plain-text files known as *dotfiles* (because the file names begin with a `.`, e.g. `~/.vimrc`, so that they are hidden in the directory listing `ls` by default).

Shells are one example of programs configured with such files. On startup, your shell will read many files to load its configuration. Depending of the shell, whether you are starting a login and/or interactive the entire process can be quite complex. [Here](https://blog.flowblok.id.au/2013-02/shell-startup-scripts.html) is an excellent resource on the topic.

For `bash`, editing your `.bashrc` or `.bash_profile` will work in most systems. Here you can include commands that you want to run on startup, like the alias we just described or modifications to your `PATH` environment variable. In fact, many programs will ask you to include a line like `export PATH="$PATH:/path/to/program/bin"` in your shell configuration file so their binaries can be found.

Some other examples of tools that can be configured through dotfiles are:

- `bash` - `~/.bashrc`, `~/.bash_profile`
- `git` - `~/.gitconfig`
- `vim` - `~/.vimrc` and the `~/.vim` folder
- `ssh` - `~/.ssh/config`
- `tmux` - `~/.tmux.conf`

How should you organize your dotfiles? They should be in their own folder, under version control, and **symlinked** into place using a script. This has the benefits of:

- **Easy installation**: if you log in to a new machine, applying your customizations will only take a minute.
- **Portability**: your tools will work the same way everywhere.
- **Synchronization**: you can update your dotfiles anywhere and keep them all in sync.
- **Change tracking**: you're probably going to be maintaining your dotfiles for your entire programming career, and version history is nice to have for long-lived projects.

What should you put in your dotfiles? You can learn about your tool's settings by reading online documentation or [man pages](https://en.wikipedia.org/wiki/Man_page). Another great way is to search the internet for blog posts about specific programs, where authors will tell you about their preferred customizations. Yet another way to learn about customizations is to look through other people's dotfiles: you can find tons of [dotfiles repositories](https://github.com/search?o=desc&q=dotfiles&s=stars&type=Repositories) on --- see the most popular one [here](https://github.com/mathiasbynens/dotfiles) (we advise you not to blindly copy configurations though). [Here](https://dotfiles.github.io/) is another good resource on the topic.

All of the class instructors have their dotfiles publicly accessible on GitHub: [Anish](https://github.com/anishathalye/dotfiles), [Jon](https://github.com/jonhoo/configs), [Jose](https://github.com/jjgo/dotfiles).

## Portability

A common pain with dotfiles is that the configurations might not work when working with several machines, e.g. if they have different operating systems or shells. Sometimes you also want some configuration to be applied only in a given machine.

There are some tricks for making this easier. If the configuration file supports it, use the equivalent of if-statements to apply machine specific customizations. For example, your shell could have something like:

```bash
if [[ "$(uname)" == "Linux" ]]; then {do_something}; fi

# Check before using shell-specific features
if [[ "$SHELL" == "zsh" ]]; then {do_something}; fi

# You can also make it machine-specific
if [[ "$(hostname)" == "myServer" ]]; then {do_something}; fi
```

If the configuration file supports it, make use of includes. For example, a `~/.gitconfig` can have a setting:

```
[include]
    path = ~/.gitconfig_local
```

And then on each machine, `~/.gitconfig_local` can contain machine-specific settings. You could even track these in a separate repository for machine-specific settings.

This idea is also useful if you want different programs to share some configurations. For instance, if you want both `bash` and `zsh` to share the same set of aliases you can write them under `.aliases` and have the following block in both:

```bash
# Test if ~/.aliases exists and source it
if [ -f ~/.aliases ]; then
    source ~/.aliases
fi
```

# Remote Machines

It has become more and more common for programmers to use remote servers in their everyday work. If you need to use remote servers in order to deploy backend software or you need a server with higher computational capabilities, you will end up using a Secure Shell (SSH). As with most tools covered, SSH is highly configurable so it is worth learning about it.

To `ssh` into a server you execute a command as follows

```bash
ssh foo@bar.mit.edu
```

Here we are trying to ssh as user `foo` in server `bar.mit.edu`. The server can be specified with a URL (like `bar.mit.edu`) or an IP (something like `foobar@192.168.1.42`). Later we will shee that if we modify ssh config file you can access just using something like `ssh bar`.

## Executing commands

An often overlooked feature of `ssh` is the ability to run commands directly. `ssh foobar@server ls` will execute `ls` in the home folder of foobar. It works with pipes, so `ssh foobar@server ls | grep PATTERN` will grep locally the remote output of `ls` and `ls | ssh foobar@server grep PATTERN` will grep remotely the local output of `ls`.

## SSH Keys

Key-based authentication exploits public-key cryptography to prove to the server that the client owns the secret private key without revealing the key. This way you do not need to reenter your password every time. Nevertheless, the private key (often `~/.ssh/id_rsa` and more recently `~/.ssh/id_ed25519`) is effectively your password, so treat it like so.

### Key generation

To generate a pair you can run [`ssh-keygen`](http://man7.org/linux/man-pages/man1/ssh-keygen.1.html).

```bash
ssh-keygen -o -a 100 -t ed25519 -f ~/.ssh/id_ed25519
```

You should choose a passphrase, to avoid someone who gets ahold of your private key to access authorized servers. Use [`ssh-agent`](http://man7.org/linux/man-pages/man1/ssh-agent.1.html) or [`gpg-agent`](https://linux.die.net/man/1/gpg-agent) so you do not have to type your passphrase every time.

If you have ever configured pushing to GitHub using SSH keys, then you have probably done the steps outlined [here](https://help.github.com/articles/connecting-to-github-with-ssh/) and have a valid key pair already. To check if you have a passphrase and validate it you can run `ssh-keygen -y -f /path/to/key`.

### Key based authentication

`ssh` will look into `.ssh/authorized_keys` to determine which clients it should let in. To copy a public key over you can use:

```bash
cat .ssh/id_ed25519.pub | ssh foobar@remote 'cat >> ~/.ssh/authorized_keys'
```

A simpler solution can be achieved with `ssh-copy-id` where available:

```bash
ssh-copy-id -i .ssh/id_ed25519.pub foobar@remote
```

## Copying files over SSH

There are many ways to copy files over ssh:

- `ssh+tee`, the simplest is to use `ssh` command execution and STDIN input by doing `cat localfile | ssh remote_server tee serverfile`. Recall that [`tee`](http://man7.org/linux/man-pages/man1/tee.1.html) writes the output from STDIN into a file.
- [`scp`](http://man7.org/linux/man-pages/man1/scp.1.html) when copying large amounts of files/directories, the secure copy `scp` command is more convenient since it can easily recurse over paths. The syntax is `scp path/to/local_file remote_host:path/to/remote_file`
- [`rsync`](http://man7.org/linux/man-pages/man1/rsync.1.html) improves upon `scp` by detecting identical files in local and remote, and preventing copying them again. It also provides more fine grained control over symlinks, permissions and has extra features like the `--partial` flag that can resume from a previously interrupted copy. `rsync` has a similar syntax to `scp`.

## Port Forwarding

In many scenarios you will run into software that listens to specific ports in the machine. When this happens in your local machine you can type `localhost:PORT` or `127.0.0.1:PORT`, but what do you do with a remote server that does not have its ports directly available through the network/internet?.

This is called *port forwarding* and it comes in two flavors: Local Port Forwarding and Remote Port Forwarding (see the pictures for more details, credit of the pictures from [this StackOverflow post](https://unix.stackexchange.com/questions/115897/whats-ssh-port-forwarding-and-whats-the-difference-between-ssh-local-and-remot)).

**Local Port Forwarding**![Local Port Forwarding](https://i.stack.imgur.com/a28N8.png%C2%A0 "Local Port Forwarding")

**Remote Port Forwarding**![Remote Port Forwarding](https://i.stack.imgur.com/4iK3b.png%C2%A0 "Remote Port Forwarding")

The most common scenario is local port forwarding, where a service in the remote machine listens in a port and you want to link a port in your local machine to forward to the remote port. For example, if we execute  `jupyter notebook` in the remote server that listens to the port `8888`. Thus, to forward that to the local port `9999`, we would do `ssh -L 9999:localhost:8888 foobar@remote_server` and then navigate to `locahost:9999` in our local machine.

## SSH Configuration

We have covered many many arguments that we can pass. A tempting alternative is to create shell aliases that look like

```bash
alias my_server="ssh -i ~/.id_ed25519 --port 2222 - L 9999:localhost:8888 foobar@remote_server
```

However, there is a better alternative using `~/.ssh/config`.

```bash
Host vm
    User foobar
    HostName 172.16.174.141
    Port 2222
    IdentityFile ~/.ssh/id_ed25519
    RemoteForward 9999 localhost:8888

# Configs can also take wildcards
Host *.mit.edu
    User foobaz
```

An additional advantage of using the `~/.ssh/config` file over aliases  is that other programs like `scp`, `rsync`, `mosh`, &amp;c are able to read it as well and convert the settings into the corresponding flags.

Note that the `~/.ssh/config` file can be considered a dotfile, and in general it is fine for it to be included with the rest of your dotfiles. However, if you make it public, think about the information that you are potentially providing strangers on the internet: addresses of your servers, users, open ports, &amp;c. This may facilitate some types of attacks so be thoughtful about sharing your SSH configuration.

Server side configuration is usually specified in `/etc/ssh/sshd_config`. Here you can make  changes like disabling password authentication, changing ssh ports, enabling X11 forwarding, &amp;c. You can specify config settings in a per user basis.

## Miscellaneous

A common pain when connecting to a remote server are disconnections due to shutting down/sleeping your computer or changing a network. Moreover if one has a connection with significant lag using ssh can become quite frustrating. [Mosh](https://mosh.org/), the mobile shell, improves upon ssh, allowing roaming connections, intermittent connectivity and providing intelligent local echo.

Sometimes it is convenient to mount a remote folder. [sshfs](https://github.com/libfuse/sshfs) can mount a folder on a remote server locally, and then you can use a local editor.

# Shells &amp; Frameworks

During shell tool and scripting we covered the `bash` shell because it is by far the most ubiquitous shell and most systems have it as the default option. Nevertheless, it is not the only option.

For example, the `zsh` shell is a superset of `bash` and provides many convenient features out of the box such as:

- Smarter globbing, `**`
- Inline globbing/wildcard expansion
- Spelling correction
- Better tab completion/selection
- Path expansion (`cd /u/lo/b` will expand as `/usr/local/bin`)

**Frameworks** can improve your shell as well. Some popular general frameworks are [prezto](https://github.com/sorin-ionescu/prezto) or [oh-my-zsh](https://github.com/robbyrussll/oh-my-zsh), and smaller ones that focus on specific features such as [zsh-syntax-highlighting](https://github.com/zsh-users/zsh-syntax-highlighting) or [zsh-history-substring-search](https://github.com/zsh-users/zsh-history-substring-search). Shells like [fish](https://fishshell.com/) include many of these user-friendly features by default. Some of these features include:

- Right prompt
- Command syntax highlighting
- History substring search
- manpage based flag completions
- Smarter autocompletion
- Prompt themes

One thing to note when using these frameworks is that they may slow down your shell, especially if the code they run is not properly optimized or it is too much code. You can always profile it and disable the features that you do not use often or value over speed.

# Terminal Emulators

Along with customizing your shell, it is worth spending some time figuring out your choice of **terminal emulator** and its settings. There are many many terminal emulators out there (here is a [comparison](https://anarc.at/blog/2018-04-12-terminal-emulators-1/)).

Since you might be spending hundreds to thousands of hours in your terminal it pays off to look into its settings. Some of the aspects that you may want to modify in your terminal include:

- Font choice
- Color Scheme
- Keyboard shortcuts
- Tab/Pane support
- Scrollback configuration
- Performance (some newer terminals like [Alacritty](https://github.com/jwilm/alacritty) or [kitty](https://sw.kovidgoyal.net/kitty/) offer GPU acceleration).

# Exercises

## Job control

1. From what we have seen, we can use some `ps aux | grep` commands to get our jobs' pids and then kill them, but there are better ways to do it. Start a `sleep 10000` job in a terminal, background it with `Ctrl-Z` and continue its execution with `bg`. Now use [`pgrep`](http://man7.org/linux/man-pages/man1/pgrep.1.html) to find its pid and [`pkill`](http://man7.org/linux/man-pages/man1/pgrep.1.html) to kill it without ever typing the pid itself. (Hint: use the `-af` flags).

2. Say you don't want to start a process until another completes, how you would go about it? In this exercise our limiting process will always be `sleep 60 &`. One way to achieve this is to use the [`wait`](http://man7.org/linux/man-pages/man1/wait.1p.html) command. Try launching the sleep command and having an `ls` wait until the background process finishes.

    However, this strategy will fail if we start in a different bash session, since `wait` only works for child processes. One feature we did not discuss in the notes is that the `kill` command's exit status will be zero on success and nonzero otherwise. `kill -0` does not send a signal but will give a nonzero exit status if the process does not exist. Write a bash function called `pidwait` that takes a pid and waits until said process completes. You should use `sleep` to avoid wasting CPU unnecessarily.

## Terminal multiplexer

1. Follow this `tmux` [tutorial](https://www.hamvocke.com/blog/a-quick-and-easy-guide-to-tmux/) and then learn how to do some basic customizations following [these steps](https://www.hamvocke.com/blog/a-quick-and-easy-guide-to-tmux/).

## Aliases

1. Create an alias `dc` that resolves to `cd` for when you type it wrongly.

2. Run `history | awk '{$1="";print substr($0,2)}' | sort | uniq -c | sort -n | tail -n 10`  to get your top 10 most used commands and consider writing shorter aliases for them. Note: this works for Bash; if you're using ZSH, use `history 1` instead of just `history`.

## Dotfiles

Let's get you up to speed with dotfiles.

1. Create a folder for your dotfiles and set up version control.
2. Add a configuration for at least one program, e.g. your shell, with some customization (to start off, it can be something as simple as customizing your shell prompt by setting `$PS1`).
3. Set up a method to install your dotfiles quickly (and without manual effort) on a new machine. This can be as simple as a shell script that calls `ln -s` for each file, or you could use a [specialized utility](https://dotfiles.github.io/utilities/).
4. Test your installation script on a fresh virtual machine.
5. Migrate all of your current tool configurations to your dotfiles repository.
6. Publish your dotfiles on GitHub.

## Remote Machines

Install a Linux virtual machine (or use an already existing one) for this exercise. If you are not familiar with virtual machines check out [this](https://hibbard.eu/install-ubuntu-virtual-box/) tutorial for installing one.

1. Go to `~/.ssh/` and check if you have a pair of SSH keys there. If not, generate them with `ssh-keygen -o -a 100 -t ed25519`. It is recommended that you use a password and use `ssh-agent` , more info [here](https://www.ssh.com/ssh/agents).
2. Edit `.ssh/config` to have an entry as follows

```bash
Host vm
    User username_goes_here
    HostName ip_goes_here
    IdentityFile ~/.ssh/id_ed25519
    RemoteForward 9999 localhost:8888
```

1. Use `ssh-copy-id vm` to copy your ssh key to the server.
2. Start a webserver in your VM by executing `python -m http.server 8888`. Access the VM webserver by navigating to `http://localhost:9999` in your machine.
3. Edit your SSH server config by doing  `sudo vim /etc/ssh/sshd_config` and disable password authentication by editing the value of `PasswordAuthentication`. Disable root login by editing the value of `PermitRootLogin`. Restart the `ssh` service with `sudo service sshd restart`. Try sshing in again.
4. (Challenge) Install [`mosh`](https://mosh.org/) in the VM and establish a connection. Then disconnect the network adapter of the server/VM. Can mosh properly recover from it?
5. (Challenge) Look into what the `-N` and `-f` flags do in `ssh` and figure out what a command to achieve background port forwarding.
