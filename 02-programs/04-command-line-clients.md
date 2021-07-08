# Clientprogramme

Mit MySQL kommt eine Reihe von Client- und Hilfsprogrammen.
Einige von denen sind wichtiger als andere, und einige sind... kaputt oder veraltet.

Hier meine Anmerkungen.

## perror

`perror` druckt den Text zu Linux- und zu MySQL-Fehlernummern aus.
Im Betrieb begegnet man meistens den Linux-Errors 2 und 13:

```bash
kris@server:~/sandboxes/msb_8_0_25$ perror 2
OS error code   2:  No such file or directory
MySQL error code MY-000002: Error reading file '%s' (OS errno %d - %s)
kris@server:~/sandboxes/msb_8_0_25$ perror 13
OS error code  13:  Permission denied
MySQL error code MY-000013: Can't get stat of '%s' (OS errno %d - %s)
```

Das ist also "Datei existiert nicht" und "Zugriffsrechte falsch". 
Beide Fehler treten meistens dann auf, wenn Dinge in `datadir` dem Systemverwalter gehören statt `mysql:mysql`.
Das ist mit einem `chown -R mysql:mysql ...` dann meist schnell gefixt.
Ein `vi` als `root` auf Dateien in `datadir` ist eventuell weniger schlau als gedacht.

Auch immer wieder gerne genommen:

```bash
kris@server:~/sandboxes/msb_8_0_25$ perror 150
MySQL error code MY-000150 (handler): Foreign key constraint is incorrectly formed
```

Dieser Text sollte eigentlich lauten: "Als root im MySQL Client, `pasger less` und dann `SHOW ENGINE INNODB STATUS\G`".
Im Abschnitt über Foreign Key Constraint Violations recht weit oben in der Ausgabe findet man den wahren Grund detailliert angezeigt.
Der Abschnitt existiert nur, wenn es schon einen solchen Fehler gegeben hat. 

## mysql_tzinfo_to_sql

MySQL Datendateien sind portabel und das Datenformat ist vorwärtskompatibel über alle Versionen, wenn die "Sprungweite" des Updates nicht mehr als eine Major-Version ist.
Ein Betriebssystemupdate wird den Umgang mit Zeichensätzen und Zeitzonen nicht beeinflussen, weil MySQL diese Dinge intern selbst und betriebssystemunabhängig implementiert.

Das gilt auch für Replikation, und zwischen Maschinen mit 32-Bit und 64-Bit Wortbreite oder zwischen Maschinen mit Big Endian (Sun, Power, Motorola) und Little Endian (Intel).

Bei den Zeitzonen ist es so, daß man die Zeitzonendaten aus einer Referenzquelle nimmt (etwa: Deine Maschine mit `/usr/share/zoneinfo` aus Deinem `tzdata`-Paket) und nach SQL umwandelt.

```bash
kris@server:~/sandboxes/msb_8_0_25$ mysql_tzinfo_to_sql  /usr/share/zoneinfo | head -10
TRUNCATE TABLE time_zone;
TRUNCATE TABLE time_zone_name;
TRUNCATE TABLE time_zone_transition;
TRUNCATE TABLE time_zone_transition_type;
START TRANSACTION;
INSERT INTO time_zone (Use_leap_seconds) VALUES ('N');
SET @time_zone_id= LAST_INSERT_ID();
INSERT INTO time_zone_name (Name, Time_zone_id) VALUES ('Africa/Abidjan', @time_zone_id);
INSERT INTO time_zone_transition (Time_zone_id, Transition_time, Transition_type_id) VALUES
 (@time_zone_id, -1830383032, 1)
```

Diese Datei kann dann in *alle* Server einer Replikationshierarchie nach `mysql.*` geladen werden.
So wird sichergestellt, daß die Zeitzonendaten einheitlich sind, und daß die Updates kontrolliert und einheitlich erfolgen, auch wenn die glibc oder die Zoneinfo-Daten einer Maschine individuell aktualisiert werden.

## mysql_config_editor

Oracle will nicht mehr, daß Logindaten im Klartext in `.my.cnf` Dateien in Deinem Userhome liegen.
Darum gibt es jetzt eine `.mylogin.cnf`.
Dies ist eine Binärdatei, die man mit `mysql_config_editor` bearbeiten kann, um dort Logindaten (und nur das) zu speichern ([Doku](https://dev.mysql.com/doc/refman/8.0/en/mysql-config-editor.html)).

Das Programm `mysql_config_editor` ist leider nicht in der Lage, ein Paßwort auf stdin oder mit einer Kommandozeilenoption zu setzen. Daher kann man `.mylogin.cnf` Dateien nicht gut (mit Ansible) provisionieren.

- [Artikel der das Format erklärt](https://blog.koehntopp.info/2020/09/23/mylogin-cnf.html)
- [Code zur Provisionierung](https://github.com/isotopp/mysql-config-coder)

## mysql

Der MySQL Kommandozeilenclient ist weitaus besser als die meisten Leute denken.
Er hat `readline`/`editline` Support und einige Kommandos, die mit Backslash beginnen (`\h für Hilfe`).
Wenn etwas nicht geht (in einer Programmiersprache oder in einem SQL-Editor), dann sollte man den Fehler unbedingt im `mysql`-Kommandozeilenclient nachstellen.
Gelingt das nicht, liegt es an der Plattform/dem Client.

Speziell "PHP Myadmin" ist ein Problem.
Das ist in [Connection Scoped State](https://blog.koehntopp.info/2020/07/28/mysql-connection-scoped-state.html) genauer erklärt.

Speziell ist es bei jedem Webclient so, daß jedes abgesendete Kommando an einen anderen Worker des Servers geht und es nicht gewährleistet ist, daß State vom ersten Kommando im zweiten Kommando noch bereitsteht. Transaktionen, Variablen und andere Dinge gibt es also nicht. Das kann sehr merkwürdige Effekte erzeugen.

Auch viele GUI-Clients machen nach jedem Kommando die Verbindung zu und für das folgende Kommando dann wieder auf. Auch da ist dann nix mit State.

## mysqldump

`mysqldump` wandelt eine existierende Datenbank in SQL Statements um, indem es sich an die Datenbank verbindet, Schema und Daten ausliest und daraus Statements generiert.

Ein Dump ist ein Script, das man dann mit dem `mysql` Kommandozeilenclient einlesen kann (zum Beispiel mit `source blabla.sql).
Das funktioniert natürlich nur bis zu einer gewissen Datenbank-Größe. 
Der Dump ist dabei weniger der Problem als das Wiedereinlesen, speziell das Erzeugen der Indices.

Wichtige Optionen:

```bash
$ mysqldump ... --no-data $datenbank $tabelle
```

Das macht einen Dump des Schemas für die angegebene Tabelle in der angegebenen Datenbank (alle Tabellen, wenn nur eine Datenbank angegeben ist).

```mysqldump ... --single-transaction --master-data=2 --events --triggers --routines ...```

Das generiert einen Dump mit Daten.
Wieder kann eine Tabelle oder eine Datenbank ohne Tabelle angegeben werden.
Die Optionen sorgen dafür, daß auch Events, Trigger und Stored Routines und Functions mit gedumpt werden.

Durch `--single-transaction` wird InnoDB dazu angehalten, den ganzen Dump in einer Read-Only Transaktion mit einem konsistenten View zu machen.
Die Datenbank läuft also während des Dump weiter, aber der Dump sieht die Transaktionen nicht, die während des Dumps passiert sind, und ist konsistent.
Die Option `--master-data=2` wird die zum Dump passende Position im Binlog mit vermerkt, aber als Kommentar.
Das gibt einem die notwendige Information, aber auch Flexibilität beim Wiedereinlesen des Dump.

Der Schalter `--opt` ist automatisch an.
Schau nach, was er genau bedeutet:

```bash
  --opt               Same as --add-drop-table, --add-locks, --create-options,
                      --quick, --extended-insert, --lock-tables, --set-charset,
                      and --disable-keys. Enabled by default, disable with
                      --skip-opt.
```

Wenn man mit `sed` oder anderen Tools dabei will, ist `--skip-extended-insert` eventuell hilfreich.
Es bewirkt, daß die von `--opt` automatisch aktivierte Option `--extended-insert` abgeschaltet wird.

## mysqlbinlog

Das Kommando `mysqlbinlog` wandelt das Binlog in lesbares SQL um.
Wenn das Binlog selbst in Row Format vorliegt, ist das auch sehr leicht parsebares, fest formatiertes SQL, weil der Row Change als SQL-Statement emuliert wird.

Mit der Option `-vvv` kann man einige sonst nur als Base64 vorliegenden Dinge als SQL interpretiert bekommen.

Mit den verschiedenen `--start` und `--stop` Varianten kann man Teile des Binlogs aussägen.
Das braucht man meistens, wenn Replikation oder Restore hoffnungslos vergurkt sind und man mit ein wenig Replication Surgery Dinge richten muß, etwa das falsche `DROP TABLE` da auslassen oder ähnliches.

## mysqlsh

Eine Shell mit Python-API, die man zur Automatisierung von MySQL Operations in Zukunft verwenden soll.
Ein Thema für sich.

## gemeinsame Optionen

Connection Parameter:

```bash
--user
--password
--host
--port
```

Wichtig für `libmysqlclient.so` basierende Clients:

```
--host=localhost vs. --host=127.0.0.1
--socket vs. --port
```

Optionen vs. ini-File vs. SET Kommando:

```
--ich_bin_eine_option -> ich-bin-eine-option= -> SET ich_bin_eine_option
- ausser, wenn sie das nicht ist
```

Größen in der Config vs. Größen in MySQL Client:

- --size=17K (M, G, T, P, E) (base-1024) -> SET size=17*1024

## persisted config variables vs. my.cnf vs. mysqlsh

Neu in MySQL 8: `SET PERSIST @bla = ...` und `SET PERSIST_ONLY @bla = ...`
Solche "Persisted config variables" werden dann in `$DATADIR/mysqld-auto.cnf` hinterlegt.
Das gibt es, weil Oracle ja jetzt MySQL-as-a-Service betreibt und man da schlecht eine `.cnf` hinterlegen kann.

## .mylogin.cnf

[Deutsch](https://blog.koehntopp.info/2012/10/03/mylogin-cnf-passworte-wiederherstellen.html),
[English](https://blog.koehntopp.info/2020/09/23/mylogin-cnf.html)

World Writeable `.*.cnf` wird ignoriert.

## Config-Staub (doof)

- ini-files, merged in order
- groups, merged in order

Und Include-Direktiven:

- !include ...
- !includedir ... # files must be *.cnf

## Meine ~/.my.cnf

```
[mysql]
user=kris
password=geheim
database=kris
show-warnings
prompt=\U [\d]>\_
```

## mysqlsh URI like dbstrings

- https://dev.mysql.com/doc/refman/8.0/en/connecting-using-uri-or-key-value-pairs.html#connecting-using-uri
- [scheme://][user[:[password]]@]host[:port][/schema][?attribute1=value1&attribute2=value2...

- mysqlsh hat eine Python API, dort shell.parseUri() and shell.unparseUri()
