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

Man braucht dabei gar keine Start-/Stop-Scripte, sondern kann auch einfach auf der Kommandozeile den Server freihändig starten:

```bash
$ mysqld --defaults-file=./my.sandbox.cnf
```

Oder gar die Init-File Option auch freifliegend mit angeben und sie nicht in die `my.cnf` editieren.

```bash
$ mysqld --defaults-file=./my.unchanged.sandbox.cnf --init-file=data/startup.sql
```

Danach muss man natürlich die `startup.sql` löschen, denn sie enthält das Serverpasswort im Klartext.
Außerdem muß man die `my.cnf` zurück editieren, denn sonst startet der Server nicht, weil er die `startup.sql` nicht mehr findet.

Wenn es nicht gelingt, findet man die Fehlermeldungen im Server Log.
Bei `dbdeployer` also in `data/msandbox.err`.

```
2021-07-06T15:45:08.236290Z 0 [ERROR] [MY-010455] [Server] Failed to open the bootstrap file /tmp/recovery.sql
2021-07-06T15:45:08.236300Z 0 [ERROR] [MY-010119] [Server] Aborting
```

Hier wollte ich `data/startup.sql` verwenden, habe aber `/tmp/recovery.sql` konfiguriert.

## skip-grant-tables

In vielen alten Anweisungen findet man den Hinweis, den Server stattdessen mit der Option `skip-grant-tables` zu starten.
Das ist nicht empfehlenswert, denn der Server läuft ja und lauscht auch auf dem Netzwerk-Port, während er ohne Authentisierung läuft. 
Keine gute Idee.

### ALTER USER geht nicht?

Ab 8.0.16 ist es so, daß `ALTER USER` nicht geht, wenn `skip-grant-tables` aktiv ist.
Das ist so, weil der Server dann gar keine Grant Tables hat, die `ALTER USER` bearbeiten könnte.
Erst nach `FLUSH PRIVILEGES` geht es.

Man kann also den Server mit `skip-grant-tables` starten, sich ohne Paßwort verbinden.
Danach kann man mit `FLUSH PRIVILEGES` die Grant Tables laden und dann erst mit `ALTER USER "root"@"localhost" IDENTIFIED BY ...` ein Kennwort neu festlegen.

## Keine PASSWORD() Funktion?

Alte Versionen von MySQL haben eine Funktion `PASSWORD()` gehabt, die ein Klartextkennwort in einer für MySQL Tabellen geeigneten Weise verschlüsselt.
Was die Funktion `PASSWORD()` gemacht hat, war je nach Version unterschiedlich, weil sich die Art und Weise wie MySQL Paßworte speichert über die Zeit geändert hat.

Man hat dann `SET PASSWORD=PASSWORD("geheim");` gemacht, um das Paßwort zu setzen.
Diese Funktion wird nun vom `ALTER USER`-Statement übernommen.

Mit 8.0.16 hat sich die Paßwortspeicherung erneut geändert, und die Funktion hätte angepaßt werden müssen.
Da es nun aber `ALTER USER ... IDENTIFIED BY ...` gibt, wird das nicht mehr gebraucht.

Weil es Leute gegeben haben, die die Funktion verwendet haben, um Paßworte für ihre Anwendung (nicht dem MySQL Server) zu verschlüsseln und weil diese Leute bei jeder Anpassung protestiert habenm, daß das ihre Anwendung zerbricht, wurde die Funktion also entfernt.
