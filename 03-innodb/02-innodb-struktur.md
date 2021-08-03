# InnoDB Struktur

InnoDB ist eine transaktionale Storage Engine.
Das heißt, man kann InnoDB bitten "Changes zu sammeln" und diese dann entweder alle auf einmal auszuführen, oder falls das nicht geht, gar nicht.

## Was sind Transaktionen?

```mysql
mysql> BEGIN;
...
mysql> COMMIT;

mysql> BEGIN WORK;
...
mysql> COMMIT;
mysql> START TRANSACTION READ WRITE;
...
mysql> COMMIT;
```

Normalerweise ist MySQL im `autocommit` Modus. Jedes Statement wird so ausgeführt, als ob es von einem unsichtbaren `COMMIT` gefolgt wäre.

```mysql
mysql [localhost:8025] {root} (performance_schema) > show global variables like 'autocom%';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| autocommit    | ON    |
+---------------+-------+
1 row in set (0.02 sec)
```

Durch den ausdrücklichen Beginn einer Transaktion wird der Autocommit-Modus vorübergehend verlassen und wir gelangen in eine explizite Transaktion.
Es gibt drei Wege, eine explizite Transaktion zu beginnen: `BEGIN`, `BEGIN WORK` und `START TRANSACTION READ WRITE`.
Alle drei funktionieren und bewirken dasselbe, aber nur die dritte Methode kennzeichnet die Transaktion auch ausdrücklich als `READ WRITE`.
Das ist wichtig, wenn man einen Proxy wie [`proxysql`](https://github.com/sysown/proxysql) in der Mitte hat und man diesen verwenden möchte, um `START TRANSACTION READ ONLY` zu einer Replica zu senden.

Leider sind viele Programme zu dumm, um von sich aus Reads von Writes zu trennen.
Solche Programme sind in der Regel auch zu dumm, um Transaktionen zu verwenden.
Oder, wenn sie es tun, Transaktionen korrekt zu verwenden.
In jedem Fall wird man da dann auch mit `proxysql` nicht viel retten können.

Eine Transaktion wird durch `COMMIT` beendet und geschrieben oder durch `ROLLBACK` beendet und vergessen, als ob nichts passiert wäre.

Aufgabe:
1. Datenbankserver starten. Prüfen ob `SHOW ENGINES` InnoDB anzeigt.
2. Tabelle anlegen. Prüfen, ob `SHOW CREATE TABLE LIKE ... \G` die Engine InnoDB anzeigt.
3. Zwei Verbindungen in die Datenbank aufmachen.
    1. In Verbindung 1 eine Transaktion als Read/Write starten, Daten in die Tabelle einfügen. **NICHT** Commit.
    2. In Verbindung 2 eine Transaktion als Read Only starten, die Tabelle ansehen. **NICHT** COMMIT. Sind die Daten zu sehen?

SQL kennt etwas, das sich `TRANSACTION ISOLATION LEVEL` nennt.

- `SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;`. Dann noch einmal 3.2 testen. Sind die Daten jetzt zu sehen?
- `SET TRANSACTION ISOLATION LEVEL READ COMMITTED;`. Dann noch einmal 3.2 testen. Sind die Daten jetzt zu sehen?
- In Verbindung 1 'COMMIT', dann auf Verbindung 2 noch einmal 3.2 testen. Sind die Daten jetzt zu sehen?
- `SET TRANSACTION ISOLATION LEVEL REPEATABLE READ`. Dann noch einmal 3.2 testen. Sind die Daten jetzt zu sehen?

In [dem alten deutschen Text](https://blog.koehntopp.info/2008/01/30/die-innodb-storage-engine.html) und dem [neueren englischen Text]() geht es um Isolation Level.
Das Beispiel oben sollte die durchexerzieren.
Es ist hilfreich die zu verstehen.
`SERIALIZEABLE` ist in MySQL weniger wichtig.
Falls man den benötigt hält man die Dinge falsch.

## InnoDB schreibt

Die folgenden Dinge behandelt Innodb sehr unterschiedlich als zum Beispiel Postgres, und es ist sinnvoll, das auch mal zu vergleichen.

Sowohl [der deutsche](https://blog.koehntopp.info/2008/02/03/die-innodb-storage-engine-konfiguration.html) als auch [der englische](https://blog.koehntopp.info/2020/07/27/mysql-transactions.html) Text zeigen die Grafik mit den InnoDB-Quadranten.

Wenn wir eine Transaktion beginnen, wird im Speicher der Log Buffer gefüllt.
Transaktionen können größer als ein Log Buffer werden, sollten es aber aus Gründen der Effizienz meist nicht sein.

Beim Commit wird der Log Buffer an eines der Redo Logs angehängt.
Das Redo Log hat eine feste Größe und besteht aus mindestens 2 Dateien.
Diese Dateien werden von MySQL wie ein Ringpuffer behandelt, der einen Write-Pointer und einen Read-Pointer hat.

`COMMIT` schreibt die Daten aus dem Log Buffer an die vom Write-Pointer bezeichnete Stelle und dreht diesen weiter.
Später wird Platz im Redo Log frei gemacht, indem ein Checkpoint ausgeführt wird. Dabei kann der Read-Pointer ebenfalls vorrücken.

Zugleich verändert jedes DML ("Data Modification Language", also Insert/Update/Delete/Replace und Freunde) Statement auch Tabellen. 
Tabellen bestehen aus Pages, jede Page ist 16 KB groß.
I/O erfolgt immer in ganzen Pages.
Schreibt man Daten, muss InnoDB eine Page von der Platte in eine freie Page im InnoDB Buffer Pool im Speicher laden. 

Dort werden die veränderten Table Rows entfernt und in das Undo Log kopiert. 
Stattdessen werden die neuen, veränderten Rows in die Tabelle gesetzt.
Bricht man die Transaktion ab, muss das rückgängig gemacht werden. 
Das ist aufwendig und MySQL geht davon aus, daß weitaus mehr Transaktionen mit `COMMIT` abgeschlossen werden als mit `ROLLBACK`.  

`COMMIT` schreibt die geänderten Rows *nicht* auf die Platte zurück. 
Die Buffer Pool Page im Speicher unterscheidet sich jetzt von der Page auf Platte. 
Wir nennen eine solche Page `dirty`.

Daten können nicht verloren gehen:
Schalten wir den Rechner ab, sind der Log Buffer und der Buffer Pool weg. 
Im Log Buffer stehen nur Daten, die nicht committed worden sind.
Es ist okay, solche Daten zu verlieren.

Im Buffer Pool standen dirty pages.
Schalten wir die Datenbank wieder an, erkennt sie Daten im Redo Log, lädt die Pages von der Platte und wendet die Änderungen aus dem Redo Log neu an.
Danach (am Ende der Redo Log Recovery) sieht der Speicher wieder genau so aus wie zum Zeitpunkt des Crashes.

1. COMMIT schreibt Daten aus dem Log Buffer ins Redo Log. Das reicht aus für Persistenz.
2. COMMIT ändert die Tabellen nicht. Das ist gut, denn Tabellen-Pages, die wir gerade ein wenig geändert haben, werden wir wahrscheinlich in naher Zukunft noch einmal mehr ändern.
3. Erst der Checkpoint schreibt Daten zurück.

## Checkpoints?

Es gibt drei Fälle für Checkpoints:

- Ihr ist langweilig, d.h. sie ist eine Weile idle.
- Es gibt keine InnoDB Pages mehr, die frei oder clean sind.
- Es ist kein unbenutzter Platz im Redo Log mehr.

Fall 2 ist "Buffer Pool Pressure". 
Pages im Buffer Pool, die frei sind, sind sofort benutzbar, wenn wir welche benötigen. 
Das ist gut.
Pages im Buffer Pool, die clean sind, haben Daten, aber wir haben eine Kopie dieser Daten auf der Platte.
Wir können sie also auch jederzeit freigeben und dann anderweitig verwenden.

Wenn jedoch sehr viele dirty Pages im Buffer Pool sind, dann checkpointen wir nicht genug. 
Es gibt eine Reihe von Variablen, mit denen wir das Verhalten von MySQL beim Checkpointen beeinflussen können:
Verschiedene dirty page low und high water marks bestimmen, ob MySQL eher zurückhaltend oder aggressiv Checkpoints macht.

Fall 3 ist "Redo Log Pressure".
Jeder Commit bewegt den Write-Pointer im Redo Log vorwärts.
Ohne Checkpoints wird er irgendwann den Kreis vollenden und von hinten in den Read-Pointer laufen.
MySQL muss ein wenig vorausschauen und rechtzeitig vorher damit anfangen, Checkpoints durchzuführen.
Für Pages, die wieder clean sind, wird das Redo Log nicht mehr gebraucht und kann freigegeben werden.
Wenn man das in der richtigen Reihenfolge tut, kann man den Read-Pointer vorwärts bewegen und gewinnt Platz.

Redo Log Pressure entsteht, wenn MySQL ein zu kleines Redo Log hat, oder wenn MySQL die Geschwindigkeit der Platten falsch einschätzt und nicht früh genug mit der Arbeit beginnt. Beides kann man konfigurieren und so kontrollieren.

## Undo Log

Das Undo Log hat zwar Backing Storage auf der Platte, wird aber in der Regel nicht geschrieben, weil das nicht notwendig ist.
Es speichert alte Versionen einer Row in einer Tabelle, bis diese von keiner Transaktion mehr gesehen werden kann.
Dann werden die entsprechenden Einträge im Undo Log durch den Purge Thread freigegeben.

Das Undo Log kann durch lang laufende Transaktionen sehr groß werden, wenn parallel dazu durch weitere Verbindungen Schreibzugriffe stattfinden.
Typische Problemfälle:

- Jemand startet eine Transaktion und geht dann weg, lässt die Shell aber offen stehen.
- Jemand macht ein Backup mit `mysqldump` während parallel dazu ein CSV-Load stattfindet.

Beides sind Fälle, die ich im Umgang mit Kundensystemen gesehen habe. Das Problem dabei ist, daß MySQL langsam wird, wenn das Undo Log groß ist, egal ob die Queries, die langsam sind das Undo Log brauchen.

Die Größe des Undo Logs zu überwachen kann bei der Suche nach unerwarteten Performanceproblemen sehr helfen.

- [MySQL Undo Log](https://blog.koehntopp.info/2011/04/28/mysql-undo-log.html)
- [Der Herr House und der Herr Heisenberg haben Replication Delay](https://blog.koehntopp.info/2012/09/24/der-herr-house-und-der-herr-heisenberg-haben-replication-delay.html) und [House und Heisenberg revisited](https://blog.koehntopp.info/2012/09/25/house-und-heisenberg-revisited.html)
- [An unexpected pool size increase](https://blog.koehntopp.info/2020/10/07/an-unexpeced-pool-size-increase.html)
- [A blast from the past](https://blog.koehntopp.info/2019/11/18/a-blast-from-the-past.html)

