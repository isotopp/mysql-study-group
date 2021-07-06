
# .bashrc

Welche Files werden von Deiner Shell beim Start gelesen und wo kannst Du `PATH` Definitionen hin tun?

[Das ist kompliziert](https://blog.flowblok.id.au/2013-02/shell-startup-scripts.html) und nach Distro wird es weiter verkompliziert.

[Anderswo](https://medium.com/@rajsek/zsh-bash-startup-files-loading-order-bashrc-zshrc-etc-e30045652f2e) erklärt man

> Moral:
> For bash, put stuff in ~/.bashrc, and make ~/.bash_profile source it.
> For zsh, put stuff in ~/.zshrc, which is always executed.

# ./use --user=root tut nicht

> Kevin:  ok, dbdeployer funktioniert scheinbar aus irgendwelchen Gründen nicht in meinem Ubuntu in der WSL, später mal schauen was da genau los ist

> $ ./use --user=root mysql
> ERROR 1045 (28000): Access denied for user 'root'@'localhost' (using password: YES)

Aus der wilden Debug-Session sind zwei Dinge mitzunehmen:

1. `strace -oout -ff -efile ./use`: Dieses `strace` läuft mit `-ff` und geht also auch auf geforkte Prozesse.
   Es verwendet die Option `-oout`, um die Tracedaten in Daten der Form `out.$PID` zu schreiben.
   Wir tracen mit `-efile` alle File-Operationen mit.
   Dabei sehen wir später, daß statt des normalen `open(2)` Systemaufrufes das modernere, containerfähige `openat(F_CWD, ...)` verwendet wird.
   Ein `grep openat out* | grep cnf` findet aber keine Auffälligkeiten.
1. Jedoch findet ein `grep pass *cnf` in der `dbdeployer` Sandbox in der Datei `my.sandbox.cnf` die Zeile `password = msandbox`.
   In der `grants.mysql` findet man die Statements, die die User anlegen.
   Dabei wird klar, daß ein User `msandbox` mit dem Paßwort `msandbox` angelegt wird.
   Bei einem `./use --user=root` wird der User aus der `my.sandbox.cnf` überschreiben, das Paßwort jedoch nicht.
   Die Meldung oben enthält den Satzteil `(using password: YES)`.
   Das bezieht sich genau auf die Zeile mit dem Paßwort aus der `my.sandbox.cnf` und das Paßwort `msandbox`.

