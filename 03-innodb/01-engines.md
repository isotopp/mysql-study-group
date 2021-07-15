# Storage Engines und HANDLER

MySQL war ursprünglich ein SQL-Parser und ein simpler Planner und Executor. 

Dabei unterstützte MySQL gar keine unterschiedlichen Query Plans, sondern immer nur einen ("[Nested Loop Join](https://en.wikipedia.org/wiki/Nested_loop_join)").
Das ist noch nicht einmal unbedingt ein Nachteil, wenn man kein Data Warehouse hat.
Denn mit einem Index und ohne Sort/Grouping kann ein Nested Loop Join ein Result streamen, also Zeile für Zeile erzeugen, ohne viel zu puffern.
Für Online Transaction Processing ist das meist der einzig sinnvolle Plan.

Der Executor greift dann auf den unterliegenden Storage Layer mit einem Satz einfacher Primitive zu.
MySQL exposed diese Primitive heute noch als SQL-Kommandos.
Dabei fühlt man sich vielleicht an DBase III erinnert, denn mit `HANDLER` Statements kann man [Data Access Programs (DAPs) schreiben als sei es wieder 1983](https://blog.koehntopp.info/2020/05/01/data-access-programs.html).

## HANDLER und DAPs

`HANDLER tbl OPEN AS t` öffnet eine Tabelle und erlaubt ihre Verwendung mit den folgenden Handler Kommandos.

`HANDLER CLOSE t` schließt diese Tabelle wieder.

Dazwischen kann man mit weiteren `HANDLER` Kommandos manuell einen Cursor (einen Datenzeiger, Record Pointer, wie ein File Pointer) durch einen Index oder die Daten schieben und sich Records greifen.

Der [DAP Artikel](https://blog.koehntopp.info/2020/05/01/data-access-programs.html) im Blog zeigt, wie das geht.
Ich muss das nicht hierein kopieren.

## Deklarativ ist besser

Niemand will DAPs schreiben.
Darum haben wir ja genau SQL erfunden, vor 50 Jahren.
Dadurch haben wir eine deklarative Sprache, in der wir sagen, was wir haben wollen, aber nicht, wie wir es kriegen.
Der Planner nimmt sich unsere Deklaration und die vorhandene Infrastruktur, generiert dann ein DAP.
Der Executor führt es aus, und liefert uns die Records, die wir wollen.

Wenn uns das zu langsam ist, schauen wir uns den Plan an, und erkennen eventuell, daß wir den Planner durch Erzeugen von mehr Infrastruktur (mehr Indices) in die Lage versetzen müssen, bessere Programme zu erzeugen.
Der Punkt dabei ist, daß das ohne Änderung des Codes oder Involvieren eines Entwicklers passiert: `ALTER TABLE tbl ADD INDEX (x)` ist eine Operations Ding, das ein Operations DBA routinemäßig tun kann ohne daß der Entwickler den Code ändern, neu compilieren oder deployen muss.

Wäre das prozedurale DAP Teil des Codes, ginge das nicht: Wir müssen einen Index anlegen und dann muss der Entwickler das DAP so umstellen, daß der Index verwendet wird.

## Von DAPs zu SQL migrieren

Vor vielen Jahren war ich in einem Auftrag für MySQL bei einem Telefonbuch Verlag nahe der holländischen Grenze, bei Rheine, und dort hatte man eine Codebase, die sehr xBase war.
Der Code war voll von kleinen DAPs.
Ich habe dem Kunden also `HANDLER` gezeigt und wie sie auf MySQL migrieren und dabei ihre existierende Codebase stumpf von xBase nach MySQL übersetzen können.
Die Abbildung ist linear, und die Übersetzung hätte man automatisieren können.
Das war ca. ein Tag von vier, die ich da war.

Die folgenden Tage haben wir dann damit zugebracht, deren DAPs in SQL-Äquivalente zu übersetzen.
Eine Schleife wird dabei zu einem SQL-Statement mit einer (eventuell komplexen) Bedingung in der `WHERE`-Clause.
Eine nested Loop wird dabei zu einem SQL-Statement mit mehreren Table References, eine für jede Schleife.
Die Bedingungen muss man dabei manuell in eine `ON`-Clause für das Join und eine `WHERE`-Clause für die Selektion aufteilen.

In Schnitt haben wir jede Data Access Function mit einem DAP drin in ein einzelnes SQL-Statement übersetzen können.
Die Funktionen sehen dann nach der Konvertierung immer so aus:

```python
def customer_by_name(conn: connection, name: str) -> List[Customer]:
    cmd = "SELECT ... FROM ... JOIN ... ON ... WHERE name = %(name)s ... ORDER BY ..."
    data = { "name": name }
    try:
        conn.execute(cmd, name)
    except OperationalError as e:
        log.error(f"Alles doof: {e}")
        sys.exit(1) 
    result = [ Customer(row["name"], row["vorname"], ... ) for row in conn.fetchone() ]
    return result 
```

Anzupassen sind dann nur noch `cmd` und das Container-Objekt (hier: `Customer`).
Eine Sammlung von diesen Funktionen zum Lesen, Schreiben und Suchen sind dann ein DAO (Data Access Object) für eine Klasse, hier `Customer`.

# Zwischen SQL und Storage Engine sitzt der Handler

Das Interface zwischen dem SQL Teil von MySQL und der Schicht, die Daten in Tabellen handhabt, ist genau das HANDLER Interface.
Es hat eine kleine Anzahl von Funktionen:

- Open und Close Table
- Read RND und Read RND Next, um den x-ten Record einer Tabelle zu lesen und von dort aus durch die Tabelle zu spazieren.
- Read First, Read Last, Read Next und Read Prev, um dasselbe auf einem Index zu tun, also einen Record "durch seinen Index, in Index Order" zu lesen.
- Handlerfunktionen zum Ändern, Schreiben und Committen eines Records.

Nicht Bestandteil der originalen Handler Spezifikation waren:

- Neu: Lockfunktionen.
- Neu: "MRR" (Multi Range Read) für so etwas wie Memcache "Get Multi", aber schlauer ("Batch Key Access" und "Engine Condition Pushdown")

Ursprünglich existiere zwar `HANDLER` als Kommando in SQL, aber keine API.
Irgendwann kam dann InnODB als 2. Storage Engine zu MyISAM dazu und man hat das Interface auch als API an die Storage Engines exponiert, damit diese von der SQL-Layer verwendet werden konnten.

Seitdem gibt es die `CREATE TABLE (...) ENGINE=...`-Clause, und die Möglichkeit, unterschiedliche Backends per Table zu verwenden.

MySQL hat das "damals" (2006-2008) sehr propagiert, und es gab eine Explosion von Storage Engines.
Das war nicht unbedingt vorteilhaft:
Nicht nur waren die meisten Engines kein Gewinn, sondern die Vielzahl der Engines machte viele Dinge notlos kompliziert.
So war zum Beispiel das Binlog von den Logs der einzelnen Engines getrennt und leider auch unsynchronisiert.
Transaktionen über Storage Engine Grenzen waren nur schwer und zu hohem Preis transaktional zu bekommen.

## Engines anzeigen

Auch in aktuellem MySQL gibt es noch mehr Engines als nur InnoDB:

```
mysql [localhost:8025] {root} (performance_schema) > show engines;
+--------------------+---------+----------------------------------------------------------------+--------------+------+------------+
| Engine             | Support | Comment                                                        | Transactions | XA   | Savepoints |
+--------------------+---------+----------------------------------------------------------------+--------------+------+------------+
| ARCHIVE            | YES     | Archive storage engine                                         | NO           | NO   | NO         |
| BLACKHOLE          | YES     | /dev/null storage engine (anything you write to it disappears) | NO           | NO   | NO         |
| MRG_MYISAM         | YES     | Collection of identical MyISAM tables                          | NO           | NO   | NO         |
| FEDERATED          | NO      | Federated MySQL storage engine                                 | NULL         | NULL | NULL       |
| MyISAM             | YES     | MyISAM storage engine                                          | NO           | NO   | NO         |
| PERFORMANCE_SCHEMA | YES     | Performance Schema                                             | NO           | NO   | NO         |
| InnoDB             | DEFAULT | Supports transactions, row-level locking, and foreign keys     | YES          | YES  | YES        |
| MEMORY             | YES     | Hash based, stored in memory, useful for temporary tables      | NO           | NO   | NO         |
| CSV                | YES     | CSV storage engine                                             | NO           | NO   | NO         |
+--------------------+---------+----------------------------------------------------------------+--------------+------+------------+
```

Keine von diesen sollte noch verwendet werden, ausgenommen:

- InnoDB, der Default.
- Performance_Schema, wird implizit da verwendet und sollte man selbst nie verwenden.
- `BLACKHOLE` kann in einigen Szenarien hilfreich sein.

Vorhanden, aber nicht sinnvoll sind:

- MyISAM
- Memory

Diese Dinge können Bestandteil einer Migration sein, aber sollten durch InnoDB ersetzt werden.

Auf keinen Fall sollte man Archive, MRG_MYISAM, FEDERATED, CSV verwenden.
Diese Dinge tun nicht das, was ihr erwartet, sind vermutlich schlecht gepflegt und nie Bestandteil einer Lösung.

## Nicht verwenden

- CSV ist ein Wochenendprojekt von Brian Aker von vor 15 Jahren.
  Weil Brian damals bei MySQL gearbeitet hat, ist es in den Source Tree geraten.
  Der Code ist klein und schlecht.
  Lies ihn.
  Danach willst Du CSV nie verwenden.
  Guter Plan.
  Verwende [Python Programme](https://blog.koehntopp.info/2020/09/28/mysql-import-csv-not-using-load-data.html) zum Laden und Dumpen von CSV.
- Archive ist eine Append-Only Engine mit Kompression und einer Reihe weiterer schlechter Eigenschaften. Verwende InnoDB Compressed Tables stattdessen.
- MRG_MYISAM ist komplett obsolet und war schon zu MySIAM Zeiten ein Quell von Bugs (einige von denen mit Data Loss). Verwende InnoDB Tabellen mit `PARTITIONED BY`-Clauses stattdessen.
- FEDERATED ist obsolet. 
  Es gibt keinen guten Ersatz.
  Das ist kein Verlust.
  Eventuell willst Du wissen, was TiDB ist.

## Migrieren

MyISAM hat 2021 in MySQL keinen Platz mehr.
Keine Transaktionen, keine Datensicherheit, funktioniert nicht gut mit Row Based Replication, und kostet Zeit beim Reparieren nach einem Crash.
Table Locks, performed also auch wie Dreck auf Maschinen mit mehreren Prozessoren.

InnoDB ersetzt MyISAM vollständig, ist sicherer und schneller.
InnoDB hat Transaktionen, vermeidet Torn Pages, und kommt mit konkurrenten Operationen durch viele CPUs gut zurecht.

Weil es in Migrationen noch vorkommt muss man MyISAM kennen.

MEMORY wurde lange Zeit implizit für interne temporäre Tabellen (etwa zum Sortieren bei ORDER BY Statements) verwendet.
Das ist mit MySQL 8 nicht mehr notwendig: Wir haben jetzt non-logged und non-persisted InnoDB stattdessen.
Viele Probleme beim Skalieren von großen Queries, die sortieren oder aggregieren fallen damit weg.

Bei älteren MySQL wird noch MEMORY verwendet, und daher muss man darüber noch reden.

## Noch mehr Engines

Spezielle Builds und alte MySQL-Versionen haben noch mehr Engines.
Die sind alle Radioaktiv und zu vermeiden.
Die ganze Storage Engine Idee hat sich im Nachhinein als Unsinn herausgestellt.

# SHOW GLOBAL STATUS

Die globale [Liste aller Statusvariablen](https://dev.mysql.com/doc/refman/8.0/en/server-status-variables.html) zählt ab [Handler_commit](https://dev.mysql.com/doc/refman/8.0/en/server-status-variables.html#statvar_Handler_commit) die verschiedenen Handler-Counter auf.

Immer wenn die SQL-Layer eine Funktion der Engine Layer aufruft, geht sie durch den entsprechenden Handler-Aufruf und der Zähler wird eins hochgedreht.
Viele Leute zeichnen `SHOW GLOBAL STATUS` Output alle x Sekunden auf (x=10 bei uns), und wenn man sich das Differenzial ansieht, kann man pauschale Aussagen über die Workload machen.
Früher war das ein wichtiges Performancetool und es war wichtig zu wissen, welcher Query-Plan wie in Handler-Counts sichtbar wurde.
Inzwischen haben wir `PERFORMANCE_SCHEMA` (`P_S`) und man muss nicht mehr Raten – dies ist obsoletes Wissen.

Anyway:

- Hohe `handler_read_rnd_next` sind ein Hinweis auf Table Scans, also auf fehlende Indices.
- Hohe `handler_read_first` sind ein Hinweis auf Index Scans, und schlechte Pläne auf dem Index.
- Lerne `P_S` und vergiss die Handler-Counts und das Slow-Log.

