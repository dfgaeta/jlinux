# JLinux

Um "kernel Linux" simulado em **um único arquivo Java 17** (`JLinux.java`, sem pacote).
Ele é um **servidor SSH**, tem uma **árvore de diretórios guardada em SQLite3**, um **terminal ANSI/xterm**
completo (dá para fazer aplicações full-screen), um shell tipo `bash`, ~115 programas (~175 nomes de comando) e permite
**várias VMs (instâncias de JLinux) na mesma JVM**, como um hypervisor.

Todo programa é uma **classe Java que estende `JLinux`**.

```
ssh -p 2222 root@localhost       # senha: root
ssh -p 2222 user@localhost       # senha: user
```

## Compilar e executar

Requisitos: JDK 17 ou superior (o `javac` dentro das VMs usa `javax.tools`, por isso precisa ser JDK e não só JRE) e o driver `sqlite-jdbc` (única dependência, em `lib/`).

```sh
./run.sh                                  # compila se preciso e inicia (porta 2222, imagem jlinux.db)
./run.sh -n vm1 -p 2200 --vm vm2:2201 --vm vm3:2202:vm3.db   # várias VMs
```

Ou à mão:

```sh
javac --release 17 -cp lib/sqlite-jdbc.jar -d classes JLinux.java
java -cp classes:lib/sqlite-jdbc.jar JLinux --port 2222
```

### Gerar um .jar

```sh
./mkjar.sh            # jlinux.jar (fino): mantenha lib/sqlite-jdbc.jar ao lado
./mkjar.sh --fat      # jlinux.jar autossuficiente (~14 MB, com o sqlite-jdbc embutido)
./mkjar.sh -o x.jar   # outro nome
java -jar jlinux.jar --port 2222
```

Opções: `-n/--name`, `-p/--port`, `-d/--db`, `--root-password`, `--listen ADDR`, `--vm NOME:PORTA[:DB]` (repetível), `-q`, `--no-console`.

O disco da VM é o arquivo SQLite (`NOME.db`): tudo o que você grava dentro da VM persiste entre execuções e `reboot`s.
Apague o `.db` para recomeçar do zero (o `mkfs` roda sozinho no primeiro boot).

### Console do hypervisor (terminal onde o java foi iniciado)

`list`, `create NOME PORTA [DB]`, `start|stop|reboot NOME`, `destroy NOME [purge]`, `info NOME`, `log NOME [N]`, `exec NOME comando...`, `quit`.

Dentro de uma VM, o root tem `vmctl` (mesmos comandos) e `vmattach NOME` (liga o seu terminal ao console de outra VM; `Ctrl-]` desconecta).

## SFTP

O sshd já sobe com o subsistema **sftp** ativo (log de boot: `sshd: subsystem sftp enabled`). Funciona com qualquer cliente: `sftp -P 2222 user@localhost`, WinSCP, FileZilla, `scp` moderno, paramiko.
Ele é a classe `JLinux.SftpServer` (`/usr/lib/sftp-server`, protocolo SFTP v3 + `posix-rename`, `hardlink` e `fsync` do OpenSSH), executada com as credenciais e permissões do usuário que logou: ls, get/put (retomada com offset), mkdir, rmdir, rm, rename, ln -s, chmod, chown, utime, truncate. Controle em `/etc/ssh/sshd_config`: `Subsystem sftp /usr/lib/sftp-server` (padrão) ou `Subsystem sftp none` para desligar.

## O que tem dentro

| Área | Conteúdo |
|---|---|
| **SSH** | SSH-2 escrito do zero sobre o JDK: kex `curve25519-sha256` e `diffie-hellman-group14-sha256/sha1`, host keys ed25519 e RSA (`rsa-sha2-256/512`), `aes128/192/256-ctr`, `hmac-sha2-256/512`, `hmac-sha1` e variantes `-etm`, senha e **chave pública** (`~/.ssh/authorized_keys`), pty, `window-change`, exec, rekey, janelas de fluxo |
| **Terminal** | disciplina de linha com modelo termios completo (canônico/raw, VMIN/VTIME, eco ECHOE/ECHOK/ECHOKE/ECHOCTL/ECHOPRT, ISIG, ^S/^Q, ^V, ^W, ^R, ICRNL/INLCR/IGNCR/IUCLC/ISTRIP, OPOST/OLCUC/OCRNL/ONLCR/TAB3, caracteres especiais configuráveis) e `stty` completo, ANSI/xterm (cores 16/256/truecolor, cursor, regiões de rolagem, tela alternativa, mouse SGR, bracketed paste, redimensionamento, UTF‑8, caracteres largos) |
| **Sistema de arquivos** | inodes/dentries/blocos de 4 KB no SQLite; permissões, setuid/setgid/sticky, links simbólicos e duros, fifos, dispositivos (`/dev/null zero random urandom tty pts/N`), `/proc` virtual (`cpuinfo meminfo uptime loadavg mounts <pid>/...`) |
| **Processos** | threads com PID/PGID/SID, sinais, grupos de processos, `fork`/`exec` simulados, `/proc/<pid>`, `ps`, `top`, `kill` |
| **Usuários** | `/etc/passwd shadow group` (senhas PBKDF2), `su`, `sudo`, `passwd`, `useradd/userdel/usermod/groupadd/groupdel/chpasswd` |
| **Shell** (`sh`, `bash`) | pipelines, `&& \|\| ;`, `&`, subshells, `if/while/until/for/case`, funções, `local`, redirecionamentos e here-docs, `$(...)`, crases, `$((...))`, `[[ ]]`, expansão de chaves, globbing, `${var:-x}`, `${#x}`, `${x#p}`, `${x/a/b}`…, **arrays indexados e associativos** (`a=(1 2)`, `${a[@]}`, `${!a[@]}`, `declare -A`, `mapfile`, `read -a`), `trap`, job control (`jobs fg bg wait`), `history`/`!!`, completar com TAB, PS1 |

### Utilitários (todos classes Java)

* **Arquivos:** `ls cp mv rm mkdir rmdir ln touch chmod chown chgrp stat cat tee more less head tail wc du df find tree file basename dirname realpath readlink mkfifo mknod mktemp truncate sync tar gzip gunzip zcat zip unzip dd md5sum sha*sum base64 od hexdump xxd cmp diff`
* **Texto:** `grep egrep fgrep sed awk(gawk) sort uniq cut tr paste comm fold nl rev tac expand strings column shuf seq expr bc xargs yes`
* **Sistema:** `ps top pgrep pkill killall kill sleep date cal uname hostname id whoami groups who w uptime free env printenv which clear reset tput stty tty dmesg nproc time timeout nohup watch getent mount lscpu man apropos`
* **Contas / energia:** `su sudo passwd useradd userdel usermod groupadd groupdel chpasswd halt reboot poweroff shutdown init`
* **Desenvolvimento:** `javac` e `java` (compilam e rodam **dentro da VM**, para qualquer usuário: o `.class` vai para o diretório atual, ou `-d DIR`)
* **Full-screen:** `usermgr` (cadastro de usuários e grupos, veja abaixo), `nano` (pico), `vi` (vim: modos normal/insert/visual, operadores+movimentos, `.`, undo/redo, `:s`, `:g`, `/`, `:w :q :wq`), `less`/`more`, `top`, `snake`, `matrix`, `life`, `clock`, `colors` (teste ANSI)
* **Hypervisor:** `vmctl`, `vmattach`

## stty

`stty [-F DISPOSITIVO] [-a|-g] [CONFIG]...` — compatível com o GNU stty.

* Sem argumentos mostra o que difere do `sane`; `-a` mostra tudo (velocidade, linhas/colunas, caracteres especiais, `min`/`time`, flags de controle/entrada/saída/local); `-g` imprime uma string reutilizável (`stty $(stty -g)` restaura); `size`, `speed`; `-F /dev/pts/N` age em outro terminal (root ou mesmo usuário).
* Flags (com `-` para desligar): `parenb parodd cmspar hupcl cstopb cread clocal crtscts ignbrk brkint ignpar parmrk inpck istrip inlcr igncr icrnl ixon ixoff iuclc ixany imaxbel iutf8 opost olcuc ocrnl onlcr onocr onlret ofill ofdel isig icanon iexten echo echoe echok echonl noflsh xcase tostop echoprt echoctl echoke flusho extproc`, `cs5..cs8`, `nl0/1 cr0..3 tab0..3 bs0/1 vt0/1 ff0/1`.
* Caracteres: `intr quit erase kill eof eol eol2 swtch start stop susp rprnt werase lnext discard` com `^C`, `^?`, `^-`/`undef`, um caractere ou número; `min N`, `time N`, `rows N`, `cols N`, `line N`, `ispeed/ospeed N`, velocidade em baud.
* Combinações: `sane raw cooked cbreak [-]lcase crt dec [-]decctlq ek [-]evenp [-]oddp [-]parity [-]nl [-]pass8 [-]litout hup tabs`.
* Efeito real: eco, canônico, VMIN/VTIME, ISIG/NOFLSH, ^S/^Q (ixon/ixany), lnext/werase/rprnt, erase/kill/eof/eol, conversões CR/NL e maiúsculas/minúsculas, TAB3, tamanho da janela. Apenas armazenados (sem efeito num terminal virtual): paridade, `ixoff`, break, atrasos, `xcase`, `tostop`, `flusho`, `extproc`, `crtscts`, velocidade.

## Programas Java "de Linux" (main) dentro da VM

Classes compiladas na VM (ou em `-cp`) com `public static void main` rodam como processos da VM, sem alteração, desde que usem o terminal como num Linux:

* `System.in/out/err`, `new FileOutputStream(FileDescriptor.out)` e `new FileInputStream(FileDescriptor.in)` vão para o terminal da sessão (o carregador de classes reescreve essas referências).
* `new ProcessBuilder("sh","-c","stty raw -echo </dev/tty").start()` executa dentro da VM (mesmo terminal, usuário e arquivos), com `redirectErrorStream`, `inheritIO`, `environment()` e leitura das saídas. `</dev/tty` e `>/dev/tty` entregam o terminal de verdade (`stty`, `isatty` funcionam).
* `System.exit` encerra só o programa; os modos do terminal (termios) são restaurados quando o `main` termina, mesmo após erro ou Ctrl-C.
* Limites: `Runtime.exec(...)` e `directory(File)` do ProcessBuilder não são redirecionados; `Runtime.addShutdownHook` roda só quando a JVM do host termina; `new FileOutputStream("arquivo")` grava no disco do host, não no VFS (use `k`/`JLinux` para o VFS).

## Escrevendo programas (classes que estendem `JLinux`)

Dentro da VM:

```sh
cp /usr/share/jlinux/examples/Hello.java . ; javac Hello.java ; java Hello a b     # classpath: -cp, $CLASSPATH ou o diretório atual
mkdir -p ~/bin ; echo "#!java -cp $PWD Hello" > ~/bin/hello ; chmod +x ~/bin/hello ; ~/bin/hello
# root, para todos: javac -d /usr/local/lib/jlinux Hello.java ; echo '#!java Hello' > /usr/local/bin/hello
```

```java
public class Hello extends JLinux {
    public int run(String[] args) throws Exception {
        out.println("pid " + proc.pid + " user " + proc.user + " cwd " + cwd());
        for (String l : linesOf("/etc/hostname")) out.println(l);   // utilitários herdados
        return 0;
    }
}
```

O `java` da VM também roda classes **comuns** com `public static void main(String[])`, que não estendem `JLinux` (por exemplo uma classe sua standalone): `System.in/out/err` ficam ligados ao terminal da sessão, `args` funciona e `System.exit(n)` só encerra o programa (devolve `n` ao shell, não derruba a JVM). Stubs `#!java -cp DIR Classe` também servem. Limites: sem Swing/AWT, sem `ProcessBuilder` para comandos do SO real.

Campos herdados: `k` (kernel), `proc`, `in`, `out`, `err`. Métodos úteis: `readFile`, `writeFile`, `list`, `stat`, `exec(...)`, `env()`, `cwd()`, `sleep`, `tty()`, `isatty(fd)`.
Também funciona no host: qualquer classe no *classpath* do Java que estenda `JLinux` pode ser instalada com um stub `#!java NomeDaClasse`.

### Cadastro de usuários (`usermgr`, apelido `cadusr`)

Aplicação full-screen (classe `JLinux.Usermgr extends JLinux.App`, arquivo `82_app_users`) que mantém `/etc/passwd`, `/etc/shadow` e `/etc/group` pelo próprio kernel:

* abas **Usuários** e **Grupos** (Tab/F9), lista com rolagem, painel de detalhes (grupos, sessões abertas, status da senha), busca (`/`), ordenação (`s`), ocultar contas de sistema (`v`), mouse (clique, duplo clique, roda, barra de F-keys);
* **F3** inclui (login, UID automático, grupo primário ou grupo privado, nome, home, shell, grupos suplementares, senha, cria o home com `/etc/skel`), **F2/Enter** altera (inclusive renomear o login), **F4** exclui (com ou sem o home), **F5** troca a senha, **F6** bloqueia/desbloqueia;
* validações (login, UID/GID duplicados, grupos inexistentes, senhas que não conferem), proteção do root e da própria conta;
* só o root altera; os demais usuários têm visão somente leitura. `usermgr` fica em `/usr/sbin`.

### Programa externo ao módulo (`examples/Hello.java`)

`examples/Hello.java` é uma classe separada, fora do `JLinux.java`, que escreve em `stdout` e lê de `stdin`:

```sh
javac --release 17 -cp classes -d examples examples/Hello.java
java -cp classes:lib/sqlite-jdbc.jar:examples JLinux          # examples entra no classpath do host
# dentro da VM:
echo '#!java Hello' > /usr/local/bin/hello ; chmod +x /usr/local/bin/hello
hello                         # interativo
printf 'Ana\nBob\n' | hello    # via pipe
```

### Aplicações full-screen

```java
public class Demo extends JLinux.App {            // raw mode, tela alternativa, resize, restauração ao sair
    public int run(String[] a) {
        begin();
        try {
            while (true) {
                scr.clear();
                scr.put(2, 1, "Hello", S(10, -1, Screen.BOLD));   // x, y, texto, estilo(fg, bg, atributos)
                scr.box(0, 0, W, H, S(8, -1, 0));
                flush();                                           // só envia o que mudou
                int k = key(200);                                  // Keys.UP, Keys.F1, 'q', Keys.RESIZE, ...
                if (k == 'q') return 0;
            }
        } finally { end(); }
    }
}
```

`Ansi`, `Keys`, `KeyReader`, `Screen` (buffer diferencial com cores 256/RGB, caracteres largos) e `LineEditor` estão disponíveis para qualquer programa.

## Várias VMs

```sh
./run.sh -n alpha -p 2200 --vm beta:2201 --vm gamma:2202
ssh -p 2201 root@localhost
```

Cada VM tem seu próprio kernel, tabela de processos, usuários, `.db` e porta SSH. De dentro de `alpha`: `vmctl create delta 2203`, `vmctl exec beta uname -n`, `vmattach beta`.

## Testes realizados

Executei a VM e a testei de verdade por SSH com **paramiko** (não havia cliente OpenSSH instalado) e com um emulador de terminal (**pyte**) para conferir a tela dos programas full-screen: as combinações de kex/cifra/MAC/host key que o paramiko consegue negociar (`curve25519-sha256@libssh.org`, `diffie-hellman-group14-sha256`, as 3 cifras, 5 MACs, 3 algoritmos de host key; os kex `curve25519-sha256` sob o nome RFC 8731 e `group14-sha1` o paramiko não oferece, ficaram sem teste), autenticação por chave RSA, rekey durante sessão ativa, resize do pty, saída de 3 MB, sessão interativa (job control, Ctrl‑C, here-doc, TAB, histórico), `less`, `vi`, `nano`, `top`, `snake`, `life`, `clock`, `colors`, `tar/zip/gzip`, `sed/awk/grep/sort`, `javac`/`java` dentro da VM, duas VMs em paralelo e `vmattach`, reboot com persistência no SQLite, compilação limpa com `--release 17`.
O PuTTY e o OpenSSH não foram testados diretamente; o protocolo segue o RFC 4253/4252/4254 e o paramiko é um cliente estrito, mas vale um teste com o seu PuTTY.

## Limitações conhecidas

* Sem SCP (o `scp` moderno usa SFTP e funciona), sem port forwarding, sem rede simulada (`ping`, `ip`, `netstat` não existem).
* Processos são threads Java: `kill -STOP/TSTP` (suspender) e `fork()` real não existem; `Ctrl-Z` não suspende.
* `awk`, `sed`, `vi`, o shell etc. são subconjuntos fiéis mas não 100 % idênticos aos originais (por exemplo, `vi` sem macros/registradores nomeados; `bash` sem `coproc`, `select`, substituição de processo `<(...)`).
* `local` não restaura arrays ao sair da função.
* Uma imagem `.db` é aberta por uma VM por vez.
