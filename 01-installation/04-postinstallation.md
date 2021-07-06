# Wo kommt meine Config her?

Hierarchisch zusammengeklöppelt, daher schwer zu lesen.

```bash
$ mysqld --verbose --help 2>&1 | grep cnf | head -1
/etc/my.cnf /etc/mysql/my.cnf /Users/kkoehntopp/homebrew/etc/my.cnf ~/.my.cnf

$ mysqld --help --verbose | grep 'groups are read'
The following groups are read: mysqld server mysqld-8.0
```

Optionen zur Vereinfachung:
```
--defaults-file=
--defaults-extra-file=
```

Ich mag es einfach, daher die my.cnf für den Server nach `/mysql/clustername/etc/my.cnf` mit einer `[mysqld]` Section und gut ist.
Dann `--defaults-file=...`

Neu in MySQL 8.0:
- SET PERSISTENT
- SET PERSISTENT_ONLY
- $datadir/mysqld-auto.cnf

Warum?
Oracle betreibt jetzt selbst MySQL in seiner Cloud, und interessiert sich jetzt viel mehr für Operations.
Außerdem "MySQL aaS", also kein Zugang zu den Kisten für Kunden.

# Postinstallation

"Dein RPM macht das schon"

- Root Account gesichert?
- Permissions im Datadir okay?
- Keine weiteren Accounts da (früher ""@localhost)
- Restartable? Service?
- Help Tables, TZ Tables geladen?

```
mysqld --initialize
mysqld --initialize-insecure
```

# Upgrades

- Read the fucking release notes. Yes, all of them!
    - Security Content
    - Incompatible Changes
    - Functional changes
    - Fugbixes
- Test the fucking upgrade

Reihenfolge:
- Upgrades im Replication Tree bottom up zur Wurzel
    - Intermediate Master Trick für Seamless Master Upgrade ("ein Server mehr")

Was geht?

https://dev.mysql.com/doc/refman/8.0/en/upgrade-paths.html
- "Von 5.7"
-  "Nur GA nach GA"
- Am Besten erst in 5.7 nach .lastest, dann nach 8.0
- GA beginnt mit 8.0.11

Upgrade früher mit `mysql_upgrade` external Script, jetzt (.16) Teil des Binary.
Die Systemtabellen in `mysql.*`, und die Views in `sys.*` sowie `I_S` und `P_S` werden aktualisiert.
Bei Millionen von Tabellen kann das schon mal weh tun (Out of Filehandle Bug).

- "--upgrade=NONE", vor .16: --no-dd-upgrade
- --upgrade=NONE,MINIMAL,AUTO,FORCE

## Ich kann schon 5.7

https://dev.mysql.com/doc/refman/8.0/en/upgrading-from-previous-series.html

Pitfall: Partitioning nur noch mit InnoDB und NDB, das muß vorher angepasst werden

Pitfall: Default Character Set utf8mb4

Pitfall: lower_case_table_names jetzt "fix"

Pitfall: NO_AUTO_CREATE_USER ist jetzt der Default und das Keyword existiert nicht mehr (Config anpassen, ggf Objekte modifizieren)

Pitfall: SRID auf SPATIAL indices müssen definiert sein, sonst werden sie nicht verwendet

Pitfall: I_S Table Renames für InnoDB (rename INNODB_SYS_ nach INNODB_ )

Changed Defaults (lesen!)

https://dev.mysql.com/doc/refman/8.0/en/upgrade-prerequisites.html
Update Preps und Checks

https://dev.mysql.com/doc/refman/8.0/en/rebuilding-tables.html
Kann sein, daß einige Indices gedropped und neu gebaut werden müssen

## Downgrade

Downgrade from MySQL 8.0 to MySQL 5.7, *or from a MySQL 8.0 release to a previous MySQL 8.0 release, is not supported*

