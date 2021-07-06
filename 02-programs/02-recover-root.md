# Recover root

## Wir bauen einen Server und vergessen das Paßwort

```bash
kris@server:~$ dbdeployer deploy single 8.0.25                                                                          Database installed in $HOME/sandboxes/msb_8_0_25
run 'dbdeployer usage single' for basic instructions'
. sandbox server started
kris@server:~$ cd sandboxes/msb_8_0_25/
kris@server:~/sandboxes/msb_8_0_25$ ./use --user=root mysql                                
mysql [localhost:8025] {root} (mysql) > set password='hab ich vergessen';
Query OK, 0 rows affected (0.01 sec)
kris@server:~/sandboxes/msb_8_0_25$ ./use --user=root mysql
ERROR 1045 (28000): Access denied for user 'root'@'localhost' (using password: YES)
```
## Wir versuchen, den Server zurück zu bekommen

Die sicherste Methode, den Server zurück zu bekommen, ist ihn zu stoppen und dann neu zu starten.
Dabei verwenden wir die `my.cnf` Direktive `init-file`.
Dort gibt man den Namen einer SQL-Datei an, die Kommandos enthält, die beim Start des Servers ausgeführt werden.
Die Datei muß natürlich für den Server beim Start lesbar sein (`mysql:mysql`).
Das wäre in `data-dir` der Fall.

Also:
```bash
kris@server:~/sandboxes/msb_8_0_25$ ./stop
stop /home/kris/sandboxes/msb_8_0_25
kris@server:~/sandboxes/msb_8_0_25$ echo 'alter user "root"@"localhost" identified by "geheim";'
alter user "root"@"localhost" identified by "geheim";
kris@server:~/sandboxes/msb_8_0_25$ echo 'alter user "root"@"localhost" identified by "geheim";' > data/startup.sql
kris@server:~/sandboxes/msb_8_0_25$ pwd
/home/kris/sandboxes/msb_8_0_25
kris@server:~/sandboxes/msb_8_0_25$ echo 'init-file=/home/kris/sandboxes/msb_8_0_25/data/startup.sql'
init-file=/home/kris/sandboxes/msb_8_0_25/data/startup.sql
kris@server:~/sandboxes/msb_8_0_25$ ./start
. sandbox server started
kris@server:~/sandboxes/msb_8_0_25$ ./status
msb_8_0_25 on
kris@server:~/sandboxes/msb_8_0_25$ ./use --user=root --password=geheim mysql
mysql [localhost:8025] {root} (mysql) > quit
Bye
```

Danach muss man natürlich die `startup.sql` löschen, denn sie enthält das Serverpasswort im Klartext.
Außerdem muß man die `my.cnf` zurück editieren, denn sonst startet der Server nicht, weil er die `startup.sql` nicht mehr findet.

Wenn es nicht gelingt, findet man die Fehlermeldungen im Server Log.
Bei `dbdeployer` also in `data/msandbox.err`.

```
2021-07-06T15:45:08.236290Z 0 [ERROR] [MY-010455] [Server] Failed to open the bootstrap file /tmp/recovery.sql
2021-07-06T15:45:08.236300Z 0 [ERROR] [MY-010119] [Server] Aborting
```

Hier wollte ich `/tmp/recovery.sql` verwenden, habe aber `/tmp/recovery.sql` konfiguriert.
