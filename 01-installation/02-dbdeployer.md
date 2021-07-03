# dbdeployer

Aktuell: [1.62.0](https://github.com/datacharmer/dbdeployer/releases/download/v1.62.0/dbdeployer-1.62.0.linux.tar.gz)

Homepage: [dbdeployer.com](https://www.dbdeployer.com/)

Multi-Command Go Binary.

Ich droppe es nach `/home/kris/bin/dbdeployer` und gut ist.

## Verzeichnisse

```bash
$ pwd
/home/kris
$ cp dbdeployer-1.62.0.linux bin/dbdeployer

$ mkdir dbdeployer    # hier liegen meine tgz Files
$ cd dbdeployer

$ dbdeployer init
SANDBOX_BINARY /home/kris/opt/mysql
SANDBOX_HOME   /home/kris/sandboxes

--------------------------------------------------------------------------------
Directory /home/kris/opt/mysql ($SANDBOX_BINARY) was created
This directory is the destination for expanded tarballs

--------------------------------------------------------------------------------
Directory /home/kris/sandboxes ($SANDBOX_HOME) was created
This directory is the destination for deployed sandboxes

--------------------------------------------------------------------------------
# dbdeployer downloads get mysql-8.0.25-linux-glibc2.17-x86_64-minimal.tar.xz
....  51 MB
--------------------------------------------------------------------------------
# dbdeployer unpack mysql-8.0.25-linux-glibc2.17-x86_64-minimal.tar.xz
Unpacking tarball mysql-8.0.25-linux-glibc2.17-x86_64-minimal.tar.xz to $HOME/opt/mysql/8.0.25
Renaming directory /home/kris/opt/mysql/mysql-8.0.25-linux-glibc2.17-x86_64-minimal to /home/kris/opt/mysql/8.0.25
--------------------------------------------------------------------------------
dbdeployer versions
Basedir: /home/kris/opt/mysql
8.0.25
--------------------------------------------------------------------------------
# dbdeployer defaults enable-bash-completion --run-it --remote
  94 kB
Download of file dbdeployer_completion.sh was successful
# completion file: dbdeployer_completion.sh
# Running: sudo cp dbdeployer_completion.sh /etc/bash_completion.d
[sudo] password for kris:
# File copied to /etc/bash_completion.d/dbdeployer_completion.sh
# Run the command 'source /etc/bash_completion'
```

## dbdeployer Update

> Starting with version 1.36.0, dbdeployer is able to update itself by getting the newest release from GitHub.
> 
> The quickest way of doing it is by running
> `dbdeployer update`

```bash
$ dbdeployer info releases latest
--------------------------------------------------------------------------------
Remote version: v1.62.0
Release:        dbdeployer 1.62.0
Date:           2021-06-06T15:24:28Z
## NEW FEATURES

* Add command `downloads tree` (shows tarballs by version)
* Add option `--sort-by` to `downloads list`

## DEPRECATION

* Deprecated support for TiDB database. There is no interest in this flavor, and the TiDB community has developed its
  own tool to achieve the same goal.
--------------------------------------------------------------------------------
        dbdeployer-1.62.0-docs.linux.sha256 (95 B)
        dbdeployer-1.62.0-docs.linux.tar.gz (6.3 MB)
        dbdeployer-1.62.0-docs.linux.tar.gz.sha256 (102 B)
        dbdeployer-1.62.0-docs.osx.sha256 (93 B)
        dbdeployer-1.62.0-docs.osx.tar.gz (6.2 MB)
        dbdeployer-1.62.0-docs.osx.tar.gz.sha256 (100 B)
        dbdeployer-1.62.0.linux.sha256 (90 B)
        dbdeployer-1.62.0.linux.tar.gz (6.1 MB)
        dbdeployer-1.62.0.linux.tar.gz.sha256 (97 B)
        dbdeployer-1.62.0.osx.sha256 (88 B)
        dbdeployer-1.62.0.osx.tar.gz (6.0 MB)
        dbdeployer-1.62.0.osx.tar.gz.sha256 (95 B)
```

## Welche Versionen sind noch vorhanden?

```bash
kris@server:~$ dbdeployer downloads list --flavor mysql
Available tarballs  ()
                        name                           OS     version   flavor     size   minimal
---------------------------------------------------- ------- --------- -------- -------- ---------
 mysql-4.1.22.tar.xz                                  Linux    4.1.22   mysql    4.6 MB   Y
 mysql-5.0.96-linux-x86_64-glibc23.tar.gz             Linux    5.0.96   mysql    127 MB
 ...
 mysql-8.0.23-linux-glibc2.17-x86_64-minimal.tar.xz   Linux    8.0.23   mysql     52 MB   Y
 mysql-8.0.24-linux-glibc2.17-x86_64-minimal.tar.xz   Linux    8.0.24   mysql     51 MB   Y
 mysql-8.0.25-linux-glibc2.12-x86_64.tar.xz           Linux    8.0.25   mysql    896 MB
 mysql-8.0.25-linux-glibc2.17-x86_64-minimal.tar.xz   Linux    8.0.25   mysql     51 MB   Y
 ```

und in hübsch:

```bash
 $ dbdeployer downloads tree --flavor=mysql --max-items 1000
 Vers                          name                          version     size   minimal
------ ---------------------------------------------------- --------- -------- ---------
 4.1    mysql-4.1.22.tar.xz                                   4.1.22   4.6 MB   Y

 5.0    mysql-5.0.96.tar.xz                                   5.0.96   5.5 MB   Y
        mysql-5.0.96-linux-x86_64-glibc23.tar.gz              5.0.96   127 MB

 5.1    mysql-5.1.72.tar.xz                                   5.1.72    10 MB   Y
        mysql-5.1.73-linux-x86_64-glibc23.tar.gz              5.1.73   134 MB

 5.5    mysql-5.5.61-linux-glibc2.12-x86_64.tar.gz            5.5.61   199 MB
        mysql-5.5.61.tar.xz                                   5.5.61   6.6 MB   Y
        mysql-5.5.62-linux-glibc2.12-x86_64.tar.gz            5.5.62   199 MB
        mysql-5.5.62.tar.xz                                   5.5.62   6.6 MB   Y

 5.6    mysql-5.6.43-linux-glibc2.12-x86_64.tar.gz            5.6.43   329 MB
        mysql-5.6.43.tar.xz                                   5.6.43   9.0 MB   Y
        mysql-5.6.44-linux-glibc2.12-x86_64.tar.gz            5.6.44   329 MB
        mysql-5.6.44.tar.xz                                   5.6.44   9.1 MB   Y

 5.7    mysql-5.7.25-linux-glibc2.12-x86_64.tar.gz            5.7.25   645 MB
        mysql-5.7.25.tar.xz                                   5.7.25    23 MB   Y
        mysql-5.7.26-linux-glibc2.12-x86_64.tar.gz            5.7.26   645 MB
        mysql-5.7.26.tar.xz                                   5.7.26    23 MB   Y
        mysql-5.7.27-linux-glibc2.12-x86_64.tar.gz            5.7.27   645 MB
        mysql-5.7.28-linux-glibc2.12-x86_64.tar.gz            5.7.28   725 MB
        mysql-5.7.29-linux-glibc2.12-x86_64.tar.gz            5.7.29   665 MB
        mysql-5.7.30-linux-glibc2.12-x86_64.tar.gz            5.7.30   660 MB
        mysql-5.7.31-linux-glibc2.12-x86_64.tar.gz            5.7.31   376 MB
        mysql-5.7.34-linux-glibc2.12-x86_64.tar.gz            5.7.34   665 MB

 8.0    mysql-8.0.13-linux-glibc2.12-x86_64.tar.xz            8.0.13   394 MB
        mysql-8.0.15-linux-glibc2.12-x86_64.tar.xz            8.0.15   376 MB
        mysql-8.0.16-linux-glibc2.12-x86_64.tar.xz            8.0.16   461 MB
        mysql-8.0.16-linux-x86_64-minimal.tar.xz              8.0.16    44 MB   Y
        mysql-8.0.17-linux-glibc2.12-x86_64.tar.xz            8.0.17   480 MB
        mysql-8.0.17-linux-x86_64-minimal.tar.xz              8.0.17    45 MB   Y
        mysql-8.0.18-linux-glibc2.12-x86_64.tar.xz            8.0.18   504 MB
        mysql-8.0.18-linux-x86_64-minimal.tar.xz              8.0.18    48 MB   Y
        mysql-8.0.19-linux-x86_64-minimal.tar.xz              8.0.19    45 MB
        mysql-8.0.20-linux-x86_64-minimal.tar.xz              8.0.20    44 MB   Y
        mysql-8.0.21-linux-glibc2.17-x86_64-minimal.tar.xz    8.0.21    48 MB   Y
        mysql-8.0.22-linux-glibc2.17-x86_64-minimal.tar.xz    8.0.22    51 MB   Y
        mysql-8.0.23-linux-glibc2.17-x86_64-minimal.tar.xz    8.0.23    52 MB   Y
        mysql-8.0.24-linux-glibc2.17-x86_64-minimal.tar.xz    8.0.24    51 MB   Y
        mysql-8.0.25-linux-glibc2.12-x86_64.tar.xz            8.0.25   896 MB
        mysql-8.0.25-linux-glibc2.17-x86_64-minimal.tar.xz    8.0.25    51 MB   Y
```

## Version runterladen

Heruntergeladen wird in das aktuelle Verzeichnis:

```bash
$ cd ~/dbdeployer
$ dbdeployer downloads get mysql-8.0.24-linux-glibc2.17-x86_64-minimal.tar.xz
Downloading mysql-8.0.24-linux-glibc2.17-x86_64-minimal.tar.xz
....  51 MB
File /home/kris/mysql-8.0.24-linux-glibc2.17-x86_64-minimal.tar.xz downloaded
Checksum matches
```

## Version auspacken

Ausgepackt wird die Datei im aktuellen Verzeichnis.

```bash
$ cd ~/dbdeployer
$ dbdeployer unpack mysql-8.0.25-linux-glibc2.17-x86_64-minimal.tar.xz
Unpacking tarball mysql-8.0.25-linux-glibc2.17-x86_64-minimal.tar.xz to $HOME/opt/mysql/8.0.25
.........100.........200...230
Renaming directory /home/kris/opt/mysql/mysql-8.0.25-linux-glibc2.17-x86_64-minimal to /home/kris/opt/mysql/8.0.25

$ ls -l ~/opt/mysql
total 88
drwxr-xr-x 8 kris kris  4096 Sep  4  2020 4.1.22
drwxr-xr-x 7 kris kris   136 Sep  4  2020 5.0.96
drwxr-xr-x 7 kris kris   136 Sep  4  2020 5.1.72
drwxr-xr-x 7 kris kris    85 Sep  4  2020 5.5.62
drwxr-xr-x 7 kris kris    98 Aug 18  2020 5.6.44
drwxr-xr-x 9 kris kris   133 Aug 19  2020 5.7.31
drwxr-xr-x 9 kris kris   133 Jan 29 10:40 8.0.19
drwxr-xr-x 9 kris kris   133 Aug 18  2020 8.0.20
drwxr-xr-x 9 kris kris   133 Jan 29 10:41 8.0.21
drwxr-xr-x 9 kris kris   133 Jan 29 10:41 8.0.22
drwxr-xr-x 9 kris kris   133 Jan 29 10:13 8.0.23
drwxr-xr-x 9 kris kris   133 Jul  3 15:39 8.0.25

$ dbdeployer versions
Basedir: /home/kris/opt/mysql
4.1.22  5.0.96  5.1.72  5.5.62  5.6.44  5.7.31  8.0.19  8.0.20  8.0.21  8.0.22  8.0.23
8.0.24  8.0.25
```

## Version ausprobieren

Für jede mit `unpack` ausgepackte Version können wir jetzt spielen:

```bash
$ dbdeployer deploy single 8.0.25
$ dbdeployer deploy single 8.0.25
Database installed in $HOME/sandboxes/msb_8_0_25
run 'dbdeployer usage single' for basic instructions'
. sandbox server started
```

und

```bash
$ dbdeployer usage single

        USING A SANDBOX

Change directory to the newly created one (default: $SANDBOX_HOME/msb_VERSION
for single sandboxes)
[ $SANDBOX_HOME = $HOME/sandboxes unless modified with flag --sandbox-home ]

The sandbox directory of the instance you just created contains some handy
scripts to manage your server easily and in isolation.

"./start", "./status", "./restart", and "./stop" do what their name suggests.
start and restart accept parameters that are eventually passed to the server.
e.g.:

  ./start --server-id=1001

  ./restart --event-scheduler=disabled

"./use" calls the command line client with the appropriate parameters,
Example:

    ./use -BN -e "select @@server_id"
    ./use -u root

"./clear" stops the server and removes everything from the data directory,
letting you ready to start from scratch. (Warning! It's irreversible!)

"./send_kill [destroy]" does almost the same as "./stop", as it sends a SIGTERM (-15) kill
to shut down the server. Additionally, when the regular kill fails, it will
send an unfriendly SIGKILL (-9) to the unresponsive server.
The argument "destroy" will immediately kill the server with SIGKILL (-9).

"./add_option" will add one or more options to my.sandbox.cnf, and restarts the
server to apply the changes.

"init_db" and "load_grants" are used during the server initialization, and should not be used
in normal operations. They are nonetheless useful to see which operations were performed
to set up the server.

"./show_binlog" and "./show_relaylog" will show the latest binary log or relay-log.

"./my" is a prefix script to invoke any command named "my*" from the
MySQL /bin directory. It is important to use it rather than the
corresponding globally installed tool, because this guarantees
that you will be using the tool for the version you have deployed.
Examples:

    ./my sqldump db_name
    ./my sqlbinlog somefile

"./mysqlsh" invokes the mysql shell. Unlike other commands, this one only works
if mysqlsh was installed, with preference to the binaries found in "basedir".
This script is created only if the X plugin was enabled (5.7.12+ with --enable-mysqlx
or 8.0.11+ without --disable-mysqlx)

"./use_admin" is created when the sandbox is deployed with --enable-admin-address (8.0.14+)
and allows using the database as administrator, with a dedicated port.
```

## Arbeiten

```bash
$ cd ~/sandboxes/msb_8_0_25/
kris@server:~/sandboxes/msb_8_0_25$ ./status
msb_8_0_25 on
```

und

```bash
$ ./use --user=root
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 10
Server version: 8.0.25 MySQL Community Server - GPL

Copyright (c) 2000, 2021, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql [localhost:8025] {root} ((none)) > select current_user();
+----------------+
| current_user() |
+----------------+
| root@localhost |
+----------------+
1 row in set (0.00 sec)
```

### Größe

```bash
$ find . -type d
.
./data
./data/#innodb_temp
./data/mysql
./data/performance_schema
./data/sys
./data/test
./tmp
kris@server:~/sandboxes/msb_8_0_25$ du -sh data
187M    data
```

Also:

```bash
$ for i in 4.1.22 5.0.96 5.1.72 5.5.62 5.6.44 5.7.34; do dbdeployer deploy single $i; done
...
r$ lsof -i -n -P | grep mysql
mysqld  2829904 kris   21u  IPv6 645644477      0t0  TCP *:18025 (LISTEN)
mysqld  2829904 kris   24u  IPv4 645570634      0t0  TCP 127.0.0.1:8025 (LISTEN)
mysqld  2831796 kris    3u  IPv4 645785180      0t0  TCP 127.0.0.1:4122 (LISTEN)
mysqld  2831944 kris   10u  IPv4 645830827      0t0  TCP 127.0.0.1:5096 (LISTEN)
mysqld  2832152 kris   10u  IPv4 645829347      0t0  TCP 127.0.0.1:5172 (LISTEN)
mysqld  2832474 kris   10u  IPv4 645832761      0t0  TCP 127.0.0.1:5562 (LISTEN)
mysqld  2832844 kris   10u  IPv4 645830363      0t0  TCP 127.0.0.1:5644 (LISTEN)
mysqld  2833194 kris   20u  IPv4 645900477      0t0  TCP 127.0.0.1:5734 (LISTEN)
$ du -sh */data
21M     msb_4_1_22/data
21M     msb_5_0_96/data
21M     msb_5_1_72/data
39M     msb_5_5_62/data
111M    msb_5_6_44/data
134M    msb_5_7_34/data
187M    msb_8_0_25/data
```

Was genau ist so viel größer geworden?

```bash
$ ls -lh */data
msb_4_1_22/data:
total 21M
-rw-rw---- 1 kris kris  10M Jul  3 16:01 ibdata1
-rw-rw---- 1 kris kris 5.0M Jul  3 16:01 ib_logfile0
-rw-rw---- 1 kris kris 5.0M Jul  3 16:01 ib_logfile1
-rw-rw---- 1 kris kris 1.1K Jul  3 16:01 msandbox.err
drwx------ 2 kris kris 4.0K Jul  3 16:01 mysql
-rw-rw---- 1 kris kris    8 Jul  3 16:01 mysql_sandbox4122.pid
drwx------ 2 kris kris    6 Jul  3 16:01 test
...

msb_8_0_25/data:
total 185M
-rw-r----- 1 kris kris   56 Jul  3 15:57  auto.cnf
-rw-r----- 1 kris kris 8.2K Jul  3 15:57  binlog.000001
-rw-r----- 1 kris kris   16 Jul  3 15:57  binlog.index
-rw------- 1 kris kris 1.7K Jul  3 15:57  ca-key.pem
-rw-r--r-- 1 kris kris 1.1K Jul  3 15:57  ca.pem
-rw-r--r-- 1 kris kris 1.1K Jul  3 15:57  client-cert.pem
-rw------- 1 kris kris 1.7K Jul  3 15:57  client-key.pem
-rw-r----- 1 kris kris 192K Jul  3 15:59 '#ib_16384_0.dblwr'
-rw-r----- 1 kris kris 8.2M Jul  3 15:57 '#ib_16384_1.dblwr'
-rw-r----- 1 kris kris 5.5K Jul  3 15:57  ib_buffer_pool
-rw-r----- 1 kris kris  12M Jul  3 15:57  ibdata1
-rw-r----- 1 kris kris  48M Jul  3 15:59  ib_logfile0
-rw-r----- 1 kris kris  48M Jul  3 15:57  ib_logfile1
-rw-r----- 1 kris kris  12M Jul  3 15:57  ibtmp1
drwxr-x--- 2 kris kris 4.0K Jul  3 15:57 '#innodb_temp'
-rw-r----- 1 kris kris 1.2K Jul  3 15:57  msandbox.err
drwxr-x--- 2 kris kris  137 Jul  3 15:57  mysql
-rw-r----- 1 kris kris  24M Jul  3 15:57  mysql.ibd
-rw-r----- 1 kris kris    8 Jul  3 15:57  mysql_sandbox8025.pid
drwxr-x--- 2 kris kris 8.0K Jul  3 15:57  performance_schema
-rw------- 1 kris kris 1.7K Jul  3 15:57  private_key.pem
-rw-r--r-- 1 kris kris  452 Jul  3 15:57  public_key.pem
-rw-r--r-- 1 kris kris 1.1K Jul  3 15:57  server-cert.pem
-rw------- 1 kris kris 1.7K Jul  3 15:57  server-key.pem
drwxr-x--- 2 kris kris   27 Jul  3 15:57  sys
drwxr-x--- 2 kris kris    6 Jul  3 15:57  test
-rw-r----- 1 kris kris  16M Jul  3 15:59  undo_001
-rw-r----- 1 kris kris  16M Jul  3 15:59  undo_002
```

## Und Ende

```bash
$ cd ~/sandboxes
$ for i in msb*; do (cd $i; ./stop); done
stop /home/kris/sandboxes/msb_4_1_22
stop /home/kris/sandboxes/msb_5_0_96
stop /home/kris/sandboxes/msb_5_1_72
stop /home/kris/sandboxes/msb_5_5_62
stop /home/kris/sandboxes/msb_5_6_44
stop /home/kris/sandboxes/msb_5_7_34
stop /home/kris/sandboxes/msb_8_0_25
```
