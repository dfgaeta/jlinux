JLinux

THIS IS ONLY AN EXPERIMENTAL CODE!

A simulated "Linux kernel" in a single Java 17 file (JLinux.java, no package).

It is an SSH server, has a directory tree stored in SQLite3, a complete ANSI/xterm terminal (capable of running full-screen applications), a bash-like shell, approximately 115 programs (approximately 175 command names), and supports multiple VMs (JLinux instances) in the same JVM, acting as a hypervisor.

Every program is a Java class that extends JLinux.
ssh -p 2222 root@localhost       # password: root
ssh -p 2222 user@localhost       # password: user
Build and run
Requirements: JDK 17 or later (the javac inside the VMs uses javax.tools, so a JDK is required, not just a JRE) and the sqlite-jdbc driver (the only dependency, located in lib/).
./run.sh                                  # compiles if needed and starts (port 2222, jlinux.db image)
./run.sh -n vm1 -p 2200 --vm vm2:2201 --vm vm3:2202:vm3.db   # multiple VMs
Or manually:
javac --release 17 -cp lib/sqlite-jdbc.jar -d classes JLinux.java
java -cp classes:lib/sqlite-jdbc.jar JLinux --port 2222
Create a .jar
./mkjar.sh            # jlinux.jar (thin): keep lib/sqlite-jdbc.jar alongside it
./mkjar.sh --fat      # self-contained jlinux.jar (~14 MB, with sqlite-jdbc bundled)
./mkjar.sh -o x.jar   # use a different name
java -jar jlinux.jar --port 2222
Options: -n/--name, -p/--port, -d/--db, --root-password, --listen ADDR, --vm NAME:PORT[:DB] (repeatable), -q, --no-console.

The VM disk is the SQLite file (NAME.db): everything written inside the VM persists across executions and reboots.

Delete the .db file to start from scratch (mkfs runs automatically on the first boot).
Hypervisor console (the terminal where Java was started)
list, create NAME PORT [DB], start|stop|reboot NAME, destroy NAME [purge], info NAME, log NAME [N], exec NAME command..., quit.

Inside a VM, root has vmctl (the same commands) and vmattach NAME (connects your terminal to another VM's console; Ctrl-] disconnects).
SFTP
The sshd starts with the sftp subsystem enabled (boot log: sshd: subsystem sftp enabled). It works with any client: sftp -P 2222 user@localhost, WinSCP, FileZilla, modern scp, and paramiko.

It is implemented by the JLinux.SftpServer class (/usr/lib/sftp-server, SFTP v3 protocol plus OpenSSH posix-rename, hardlink, and fsync extensions), and runs with the credentials and permissions of the logged-in user. Supported operations include: ls, get/`put` (resuming from an offset), mkdir, rmdir, rm, rename, ln -s, chmod, chown, utime, and truncate.

Configure it in /etc/ssh/sshd_config: Subsystem sftp /usr/lib/sftp-server (default) or Subsystem sftp none to disable it.
What's included
Area	Contents
SSH	SSH-2 implemented from scratch on top of the JDK: key exchange (curve25519-sha256 and diffie-hellman-group14-sha256/sha1), ed25519 and RSA host keys (rsa-sha2-256/512), aes128/192/256-ctr, hmac-sha2-256/512, hmac-sha1 and -etm variants, password and public-key authentication (~/.ssh/authorized_keys), pty, window-change, exec, rekey, and flow-control windows
Terminal	Line discipline with a complete termios model (canonical/raw modes, VMIN/VTIME, ECHO/ECHOE/ECHOK/ECHOKE/ECHOCTL/ECHOPRT echo modes, ISIG, ^S/^Q, ^V, ^W, ^R, ICRNL/INLCR/IGNCR/IUCLC/ISTRIP, OPOST/OLCUC/OCRNL/ONLCR/TAB3, configurable special characters), and full stty; ANSI/xterm (16/256/truecolor, cursor, scrolling regions, alternate screen, SGR mouse, bracketed paste, resizing, UTF-8, wide characters)
File system	Inodes/dentries/4 KB blocks in SQLite; permissions, setuid/setgid/sticky bits, symbolic and hard links, FIFOs, devices (/dev/null, zero, random, urandom, tty, pts/N), virtual /proc (cpuinfo, meminfo, uptime, loadavg, mounts, <pid>/...)
Processes	Threads with PID/PGID/SID, signals, process groups, simulated fork/`exec`, /proc/<pid>, ps, top, kill
Users	/etc/passwd, shadow, and group (PBKDF2 passwords), su, sudo, passwd, useradd/userdel/usermod/groupadd/groupdel/chpasswd
Shell (sh, bash)	Pipelines, && || ;, &, subshells, if/while/until/for/case, functions, local, redirections and here-docs, $(...), backticks, $((...)), [[ ]], brace expansion, globbing, ${var:-x}, ${#x}, ${x#p}, ${x/a/b}…, indexed and associative arrays (a=(1 2), ${a[@]}, ${!a[@]}, declare -A, mapfile, read -a), trap, job control (jobs fg bg wait), history/`!!`, TAB completion, PS1


Utilities (all implemented as Java classes)
- Files: ls cp mv rm mkdir rmdir ln touch chmod chown chgrp stat cat tee more less head tail wc du df find tree file basename dirname realpath readlink mkfifo mknod mktemp truncate sync tar gzip gunzip zcat zip unzip dd md5sum sha*sum base64 od hexdump xxd cmp diff
- Text: grep egrep fgrep sed awk(gawk) sort uniq cut tr paste comm fold nl rev tac expand strings column shuf seq expr bc xargs yes
- System: ps top pgrep pkill killall kill sleep date cal uname hostname id whoami groups who w uptime free env printenv which clear reset tput stty tty dmesg nproc time timeout nohup watch getent mount lscpu man apropos
- Accounts / power: su sudo passwd useradd userdel usermod groupadd groupdel chpasswd halt reboot poweroff shutdown init
- Development: javac and java (compile and run inside the VM, for any user: the .class file is written to the current directory, or to -d DIR)
- Full-screen: usermgr (user and group management; see below), nano (pico), vi (vim: normal/insert/visual modes, operators and motions, ., undo/redo, :s, :g, /, :w :q :wq), less/`more`, top, snake, matrix, life, clock, colors (ANSI test)
- Hypervisor: vmctl, vmattach
stty
stty [-F DEVICE] [-a|-g] [CONFIG]... — compatible with GNU stty.
- With no arguments, displays settings that differ from sane; -a displays everything (speed, rows/columns, special characters, min/`time`, control/input/output/local flags); -g prints a reusable string (stty $(stty -g) restores the settings); size, speed; -F /dev/pts/N operates on another terminal (root or the same user).
- Flags (prefix with - to disable): parenb parodd cmspar hupcl cstopb cread clocal crtscts ignbrk brkint ignpar parmrk inpck istrip inlcr igncr icrnl ixon ixoff iuclc ixany imaxbel iutf8 opost olcuc ocrnl onlcr onocr onlret ofill ofdel isig icanon iexten echo echoe echok echonl noflsh xcase tostop echoprt echoctl echoke flusho extproc, cs5..cs8, nl0/1 cr0..3 tab0..3 bs0/1 vt0/1 ff0/1.
- Special characters: intr quit erase kill eof eol eol2 swtch start stop susp rprnt werase lnext discard, with values such as ^C, ^?, ^-/`undef`, a single character, or a number; min N, time N, rows N, cols N, line N, ispeed/ospeed N, baud rate.
- Combinations: sane raw cooked cbreak [-]lcase crt dec [-]decctlq ek [-]evenp [-]oddp [-]parity [-]nl [-]pass8 [-]litout hup tabs.
- Actual effects: echo, canonical mode, VMIN/VTIME, ISIG/NOFLSH, ^S/^Q (ixon/`ixany`), lnext/`werase`/`rprnt`, erase/kill/eof/eol, CR/NL and uppercase/lowercase conversions, TAB3, window size. The following are only stored (they have no effect on a virtual terminal): parity, ixoff, break, delays, xcase, tostop, flusho, extproc, crtscts, speed.
Java "Linux" programs (main) inside the VM
Classes compiled inside the VM (or in -cp) with public static void main run as VM processes without modification, provided they use the terminal as they would on Linux:
- System.in/out/err, new FileOutputStream(FileDescriptor.out), and new FileInputStream(FileDescriptor.in) are connected to the session terminal (the class loader rewrites these references).
- new ProcessBuilder("sh","-c","stty raw -echo </dev/tty").start() runs inside the VM (same terminal, user, and files), with redirectErrorStream, inheritIO, environment(), and output reading. </dev/tty and >/dev/tty provide access to the actual terminal (stty and isatty work).
- System.exit terminates only the program; terminal modes (termios) are restored when main finishes, including after an error or Ctrl-C.
- Limitations: Runtime.exec(...) and ProcessBuilder.directory(File) are not redirected; Runtime.addShutdownHook runs only when the host JVM terminates; new FileOutputStream("file") writes to the host disk, not the VFS (use k/`JLinux` for the VFS).
  
Writing programs (classes that extend JLinux)
Inside the VM:
cp /usr/share/jlinux/examples/Hello.java . ; javac Hello.java ; java Hello a b     # classpath: -cp, $CLASSPATH, or the current directory
mkdir -p ~/bin ; echo "#!java -cp $PWD Hello" > ~/bin/hello ; chmod +x ~/bin/hello ; ~/bin/hello
# root, system-wide: javac -d /usr/local/lib/jlinux Hello.java ; echo '#!java Hello' > /usr/local/bin/hello
public class Hello extends JLinux {
    public int run(String[] args) throws Exception {
        out.println("pid " + proc.pid + " user " + proc.user + " cwd " + cwd());
        for (String l : linesOf("/etc/hostname")) out.println(l);   // inherited utilities
        return 0;
    }
}
The VM's java command also runs regular classes with public static void main(String[]) that do not extend JLinux (for example, your own standalone class): System.in/out/err are connected to the session terminal, args works, and System.exit(n) only terminates the program (returns n to the shell; it does not shut down the JVM). Stubs such as #!java -cp DIR ClassName also work.

Limitations: no Swing/AWT and no ProcessBuilder for real operating-system commands.

Inherited fields: k (kernel), proc, in, out, err. Useful methods: readFile, writeFile, list, stat, exec(...), env(), cwd(), sleep, tty(), isatty(fd).

It also works on the host: any class on the Java classpath that extends JLinux can be installed using a #!java ClassName stub.
User management (usermgr, alias cadusr)
A full-screen application (class JLinux.Usermgr extends JLinux.App, file 82_app_users) that manages /etc/passwd, /etc/shadow, and /etc/group through the kernel itself:
- Users and Groups tabs (Tab/F9), scrollable list, details panel (groups, open sessions, password status), search (/), sorting (s), hide system accounts (v), mouse support (click, double-click, wheel, F-key bar).
- F3 adds a user (login, automatic UID, primary group or private group, name, home, shell, supplementary groups, password, creates the home directory using /etc/skel); F2/Enter edits (including renaming the login); F4 deletes (with or without the home directory); F5 changes the password; F6 locks/unlocks the account.
- Validation (login, duplicate UID/GID, nonexistent groups, password mismatch), protection for root and the current account.
- Only root can make changes; other users have read-only access. usermgr is located in /usr/sbin.
External program (examples/Hello.java)
examples/Hello.java is a separate class, outside JLinux.java, that writes to stdout and reads from stdin:
javac --release 17 -cp classes -d examples examples/Hello.java
java -cp classes:lib/sqlite-jdbc.jar:examples JLinux          # examples is added to the host classpath
# inside the VM:
echo '#!java Hello' > /usr/local/bin/hello ; chmod +x /usr/local/bin/hello
hello                         # interactive
printf 'Ana\nBob\n' | hello    # through a pipe
Full-screen applications
public class Demo extends JLinux.App {            // raw mode, alternate screen, resize, restoration on exit
    public int run(String[] a) {
        begin();
        try {
            while (true) {
                scr.clear();
                scr.put(2, 1, "Hello", S(10, -1, Screen.BOLD));   // x, y, text, style (fg, bg, attributes)
                scr.box(0, 0, W, H, S(8, -1, 0));
                flush();                                           // sends only what changed
                int k = key(200);                                  // Keys.UP, Keys.F1, 'q', Keys.RESIZE, ...
                if (k == 'q') return 0;
            }
        } finally { end(); }
    }
}
Ansi, Keys, KeyReader, Screen (differential buffer with 256/RGB colors and wide characters), and LineEditor are available to any program.
Multiple VMs
./run.sh -n alpha -p 2200 --vm beta:2201 --vm gamma:2202
ssh -p 2201 root@localhost
Each VM has its own kernel, process table, users, .db, and SSH port. From inside alpha: vmctl create delta 2203, vmctl exec beta uname -n, vmattach beta.
Tests performed
I ran the VM and tested it over SSH using paramiko (an OpenSSH client was not installed) and a terminal emulator (pyte) to verify the full-screen applications: the kex/cipher/MAC/host-key combinations that paramiko can negotiate (curve25519-sha256@libssh.org, diffie-hellman-group14-sha256, all 3 ciphers, 5 MACs, and 3 host-key algorithms; the curve25519-sha256 key exchange under the RFC 8731 name and group14-sha1, which paramiko does not offer, were not tested), RSA public-key authentication, rekey during an active session, pty resizing, 3 MB output, interactive sessions (job control, Ctrl-C, here-doc, TAB, history), less, vi, nano, top, snake, life, clock, colors, tar/zip/gzip, sed/awk/grep/sort, javac/`java` inside the VM, two VMs running in parallel and vmattach, reboot with SQLite persistence, and a clean build using --release 17.

PuTTY and OpenSSH were not tested directly. The protocol follows RFC 4253/4252/4254 and paramiko is a strict client, but testing with your PuTTY client is still recommended.
Known limitations
- No SCP subsystem (modern scp uses SFTP and works), no port forwarding, and no simulated network (ping, ip, and netstat are unavailable).
- Processes are Java threads: kill -STOP/TSTP (suspend) and real fork() are not available; Ctrl-Z does not suspend a process.
- awk, sed, vi, the shell, and other tools are faithful subsets, but are not 100% identical to their originals (for example, vi has no macros/named registers; bash has no coproc, select, or process substitution <(...)).
- local does not restore arrays when a function exits.
- A .db image can be opened by only one VM at a time.
