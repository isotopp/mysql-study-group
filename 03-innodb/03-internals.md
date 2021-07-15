# InnoDB Ruby

Es gibt ein wunderbares Projekt von Jeremy Cole, [InnoDB Ruby](https://github.com/jeremycole/innodb_ruby).
Das tut genau das, was man denken würde:
Es greift auf InnoDB Strukturen auf der Platte zu, um sie zu analysieren, ist aber nicht in C/C++ geschrieben, sondern in Ruby.

Das macht es sehr viel einfacher, bestimmte interne Dinge zu verstehen.

Zum Tool gibt es ein [Wiki](https://github.com/jeremycole/innodb_ruby/wiki) und eine (alte) Blog-Serie von Jeremy.

Die Kenntnis dieser internals geht weit darüber hinaus, was man normal von einem MySQL DBA erwarten würde, und wir werden das hier auch nicht durchgehen.

## Einige wichtige Details in kurz


### Page

Eine InnoDB Page ist 16 KB im Speicher groß.
Mit Kompression kann sie auf der Platte kleiner sein, aber auch das wollen wir hier erst mal nicht diskutieren.

Die Page besteht aus einem Page Header, den Rows und einem Page Footer.

Im Page Header ist ein Row-Verzeichnis, mit Offsets relativ zum Anfang der Page.
Das heißt, InnoDB weiß, Row 1 steht 60 Bytes hinter dem Start der Page, Rows 5 dann 2400 Bytes hinter dem Start der Page und so weiter.
Tatsächlich speichert InnoDB nicht den Offset jeder Rwo, sondern um Platz zu sparen nur den Offset jeder 4. Row, aber das ist ein Implementierungsdetail.

Von außerhalb wird eine Row immer durch eine Page Nummer und die Row Nummer angesprochen.
Wenn InnoDB also eine Row innerhalb einer Page verschiebt, dann ist das von außen nicht zu bemerken.

Im Page Footer steht unter anderem eine Page Checksum.
InnoDB kann daher Speicherfehler auch dann erkennen, wenn man Speicher ohne ECC verwendet.
Viele Billig-Hoster hosten auf Desktop-Boards mit Desktop RAM ohne ECC, dort ist InnoDB früher oft abgestürzt – RAM ist weit weniger zuverlässig als die meisten Menschen glauben.

### Buffer Pool

Pages im Buffer Pool können wie erwähnt clean oder dirty sein.
Leider hat MySQL keine Zähler für Page Accesses, die einem erlauben würden, die Größe des Working Set zu schätzen.
Wir würden ja sehr gerne wissen, welche und wie viele Pages in den letzten 1, 10, 60, 600 und 3600 Sekunden gelesen oder geschrieben wurden.
Damit können wir dann abschätzen, ob das im Vergleich zur Gesamtgröße des Buffer Pools und damit zur Datenbank-Instanz zu viel oder zu wenig ist.
Am Ende könnten dann also gezielt die Instanz verkleinern.
Ohne diese Daten ist das ein Ratespiel.

Früher hatte der gesamte Buffer Pool ein einziges globales Lock (5.0.36 und vorher).
Mit großen Buffer Pools (32 GB damals) und mehreren CPUs (4 damals) wurde das schnell ein Problem.
Wir bekamen dann per-page Access Locks.
Jetzt gab es immer noch Contention auf den Free Lists und anderen Strukturen für Metadaten.
Seit einiger Zeit (5.7?) gibt es daher die Option, den Buffer Pool in Instanzen gleicher Größe zu unterteilen.
Die Gesamtgröße muss dann ein Vielfaches der Instanzgröße sein.
Setzt man sich also 64 GB Pool und 2 GB Instanzgröße, dann bekommt man 32 Buffer Pool Instanzen von 2 GB Größe.

### Table

Auf der Platte liegt die Tabellendefinition in der `.frm`-Datei, aber auch noch einmal in den InnoDB Metadaten in der `ibdata1`-Datei.
Mit MySQL 8 gibt es keine `.frm`-Dateien mehr, so daß sich diese Strukturen nicht mehr desynchronisieren können.

Daten liegen in den `.ibd`-Dateien.
Technisch gesehen kann man InnoDB auch dazu zwingen, Daten in `ibdata` abzulegen, aber das ist stark nicht empfohlen, aus einer Reihe von Gründen (`innodb_file_per_table`).

InnoDB kann `.ibd`-Dateien nicht verkleinern.
Platz in einer `.ibd`-Datei wird intern freigegeben und wieder verwendet (`INFORMATION_SCHEMA.TABLES, DATA_FREE`).
Mit `OPTIMIZE TABLE` kopiert InnoDB die Daten aus der alten in eine neue Tabelle um und schiebt dann die neue Tabelle an Stelle der alten.
Dadurch wird vorübergehend 2x Platz verbraucht, aber am Ende die `.ibd`-Datei durch eine kleinere ersetzt.

Eine `.ibd`-Datei wächst zunächst in Schritten von 16 KB, und ab 1 MB Größe in Schritten von 1 MB (64 Pages, ein Segment).
Die Datei enthält nur Indices, und alle Indices sind B-Bäume.
Daten stehen in den Blättern des Index `PRIMARY` (des Primärschlüssels), das ist dann ein B+-Baum.

Damit sind die Daten im Primärschlüssel in PK-Reihenfolge sortiert (PK ist immer ein `CLUSTERED INDEX` in InnoDB).
Das ist beim Datendesign wichtig, weil es bestimmt, welche Daten zusammen (auf derselben Page, im selben Segment) gespeichert werden.
Bei Veränderung des Wertes des PK muss der Record physisch bewegt werden.
Das ist Performance-Selbstmord.

Die Sekundärschlüssel verwenden den Primary Key als Zeiger auf die Daten.
Das bedeutet, `CREATE INDEX a_idx ON t (a)` wird erzeugt, indem alle `a`-Daten aus dem PK extrahiert werden, in `a`-Reihenfolge sortiert werden und dann als `(a, id)`-Paare im Sekundärschlüssel `a_idx` ablegt werden.
Damit ist `SELECT id FROM t WHERE a = 10` alleine aus dem Index zu beantworten (`a_idx` ist ein Covering Index für diese Query). 
Und damit ist `id` Bestandteil jedes anderen Schlüssels, sollte also besser wenige Bytes belegen ("Wie klein?" "Nicht mehr als 32 Bytes, besser 8").

Es bedeutet auch, daß Pages im Primary Key geteilt und Records verschoben werden können, ohne daß ein Sekundärschlüssel aktualisiert werden muss. 
Das begrenzt den Umfang von Updates.
Man stelle sich vor, der Sekundärschlüssel enthielte physische Zeiger (`page#:record#`, `page#:offset`) und eine Page im PK muss einen Split machen, 50% der Records gehen auf eine andere Page – in einer Tabelle mit 5 Sekundärschlüsseln.
Das erzeugt jedes Mal, wenn diese Situation eintritt, Änderungen in mindestens 5 weiteren Pages, die zurückgeschrieben werden müssen.

Durch Verwenden von `(page#:row#)` isoliert InnoDB Änderungen der Aufteilung in einer Page auf diese Page, und sie sind von außen nicht zu sehen.
Durch Verwenden von Primary Keys als Record Pointers in Sekundärschlüsseln isoliert InnoDB Page Splits und Page Merges im Primary Key und muss Sekundärschlüssel nicht aktualisieren, weil sich Records verschoben haben.

Weil der Primärschlüssel die physische Anordnung von Records in der Tabelle bestimmt und weil er in jedem Sekundärschlüssel verwendet wird, muss der PK mit Bedacht gewählt werden.

- Er muß vorhanden sein. InnoDB macht schlimme Dinge, wenn er nicht definiert wird (was technisch gesehen legal ist).
- Er sollte kurz sein (<32 Bytes, ideal 8 Bytes)
- Sinnvolle Reihenfolge

In den meisten Fällen ist das mit einem `id bigint unsigned not null primary key auto_increment` erledigt.
Das ist so häufig, daß man `id SERIAL` schreiben kann.
`SHOW CREATE TABLE ...\G` zeigt dann, wie das ersetzt worden ist.
