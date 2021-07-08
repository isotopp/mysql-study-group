# mysqld

Aus dem RPM: `/usr/sbin/mysqld`

- Start aus dem Unit-File, dort "bare" (ohne weiteren Wrapper): `/usr/lib/systemd/system/mysqld.service`
- "after network", "after syslog", "wanted=multi.user"
- Als mysql.mysql
    - Alle Dateien in `$datadir` müssen diesem User gehören (Error 13 beim Start)

Weitere Optionen aus dem Service-File:
```    
ExecStartPre=/usr/bin/mysqld_pre_systemd  ## mostly tests need for upgrade
ExecStart=/usr/sbin/mysqld $MYSQLD_OPTS
EnvironmentFile=-/etc/sysconfig/mysql
LimitNOFILE = 10000
Restart=on-failure
RestartPreventExitStatus=1
Environment=MYSQLD_PARENT_PID=1
PrivateTmp=false
```

Früher:
- `mysql.server` -> `/etc/init.d/mysqld`
- das startet `mysqld_safe`
- das startet `mysqld`

Probleme:
- Filehandles werden im `mysqld_safe` hochgesetzt -> falsche Group in my.cnf, unklar
- NUMA-Handling brauchte Patches im `mysqld_safe` (siehe unten)
- Distros machen den Systemstart komisch

Außerdem `mysqld_multi`: Ein defekter Weg, mehr als eine Instanz pro Maschine zu haben. Nie, nie, nie verwenden.

## Postinstallation Handling

Nur in älteren Versionen von MySQL (vor 8.0.16)

- `mysql_install_db`: Erzeuge die Systemdatenbanken, lade die Help-Tables.

- `mysql_secure_installation`: Erzwinge Passwort, lösche anonyme User, andere Nacharbeiten.
- `mysql_tzinfo_to_sql`: Laden von Zeitzoneninformation

und

- `mysql_upgrade`: Aktualisiere die Systemtabellen für die neue Major Version.

## Altes MySQL freihändig starten

Ghetto-Sandbox

Start muß im ausgepackten tar erfolgen. Dies geht nicht.

```bash
kris@server:/tmp$ mkdir keks; ~/opt/mysql/5.0.96/scripts/mysql_install_db --user=kris --datadir=/tmp/keks
FATAL ERROR: Could not find ./bin/my_print_defaults

If you are using a binary release, you must run this script from
within the directory the archive extracted into.  If you compiled
MySQL yourself you must run 'make install' first.
```

Stattdessen dies:

```bash
kris@server:/tmp$ cd ~/opt/mysql/5.0.96/
kris@server:~/opt/mysql/5.0.96$ mkdir -p /tmp/keks; ~/opt/mysql/5.0.96/scripts/mysql_install_db --user=kris --datadir=/tmp/keks
Installing MySQL system tables...
OK
Filling help tables...
OK

To start mysqld at boot time you have to copy
support-files/mysql.server to the right place for your system

PLEASE REMEMBER TO SET A PASSWORD FOR THE MySQL root USER !
To do so, start the server, then issue the following commands:
./bin/mysqladmin -u root password 'new-password'
./bin/mysqladmin -u root -h server password 'new-password'

Alternatively you can run:
./bin/mysql_secure_installation

which will also give you the option of removing the test
databases and anonymous user created by default.  This is
strongly recommended for production servers.

See the manual for more instructions.

You can start the MySQL daemon with:
cd . ; ./bin/mysqld_safe &

You can test the MySQL daemon with mysql-test-run.pl
cd mysql-test ; perl mysql-test-run.pl

Please report any problems with the ./bin/mysqlbug script!

The latest information about MySQL is available on the web at
http://www.mysql.com
Support MySQL by buying support/licenses at http://shop.mysql.com
```

Das kann man jetzt freihändig starten:

```bash
kris@server:~/opt/mysql/5.0.96$ cat my.cnf
[mysqld]
socket=/tmp/keks.socket
port=3307
user=kris
datadir=/tmp/keks

[client]
socket=/tmp/keks.socket
port=3307

kris@server:~/opt/mysql/5.0.96$ bin/mysqld --defaults-file=$(pwd)/my.cnf --basedir=$(pwd) --console
InnoDB: The first specified data file ./ibdata1 did not exist:
InnoDB: a new database to be created!
210704  9:55:55  InnoDB: Setting file ./ibdata1 size to 10 MB
InnoDB: Database physically writes the file full: wait...
210704  9:55:55  InnoDB: Log file ./ib_logfile0 did not exist: new to be created
InnoDB: Setting log file ./ib_logfile0 size to 5 MB
InnoDB: Database physically writes the file full: wait...
210704  9:55:55  InnoDB: Log file ./ib_logfile1 did not exist: new to be created
InnoDB: Setting log file ./ib_logfile1 size to 5 MB
InnoDB: Database physically writes the file full: wait...
InnoDB: Doublewrite buffer not found: creating new
InnoDB: Doublewrite buffer created
InnoDB: Creating foreign key constraint system tables
InnoDB: Foreign key constraint system tables created
210704  9:55:55  InnoDB: Started; log sequence number 0 0
210704  9:55:55 [Note] bin/mysqld: ready for connections.
Version: '5.0.96'  socket: '/tmp/keks.socket'  port: 3307  MySQL Community Server (GPL)
```

Kann man dann auch verwenden:

```bash
$ bin/mysql --user=root --port=3307 --host=127.0.0.1 --password="" mysql
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 4
Server version: 5.0.96 MySQL Community Server (GPL)

Copyright (c) 2000, 2011, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
```
