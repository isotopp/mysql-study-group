# Notizen zum Meeting am 29.07.21

## Vergleich InnoDB mit MyISAM

- InnoDB ist default Engine
- MyISAM sollte man heutzutage nicht mehr verwenden
- InnoDB kann Row-Level Locking, MyISAM lockt immer die ganze Tabelle
- InnoDB ist in der einzelperformance zwar langsamer als MyISAM aber durch die parallelisierung (row/table-Locking) erhöht sich die Gesamtperformance
- InnoDB hat Checksums auf den Datenblöcken und kann daher Fehlerhafte Daten von der Platte erkennen

## Primary Keys bei InnoDB essentiell wichtig
- InnoDB kann mit definierten PRIMARY-Key gut um
- Überhaupt keinen PRIMARY-Key zu haben ist aber für Performance eine sehr schlechte idee
-- InnoDB versucht sich dann die beste Spalte für einen Primary-Key zu identifizieren und nimmt im zweifelsfall die DB_ROW_ID.
-- Daher immer selbst einen anständigen PK definieren.
-- Mit Indices nicht geizen
-- Es gibt ein Konfig-Flag, welches PKs auf Tabellen erzwingt. Ohne lassen sich die Tabellen dann nicht anlegen. ([sql+require_primary_key](https://dev.mysql.com/doc/refman/8.0/en/server-system-variables.html#sysvar_sql_require_primary_key))

## Transaktionen
- SHOW ENGINE INNODB STATUS;
-- Zeigt auch die offenen Transaktionen an
-- Kann zum Identifizieren von Transaktionssündern verwendet werden
-- START TRANSACTION READY ONLY sorgt dafür, dass die DB für diese Session "eingefrohren" wird und diese Ansicht konsistent ist.
--- Updates laufen im Hintergrund weiter, Undo-Log wird also größer solange diese Transaktion offen ist.
- https://blog.koehntopp.info/2019/11/18/a-blast-from-the-past.html
- https://github.com/innotop/innotop
- https://blog.koehntopp.info/2020/07/30/mysql-transactions-writing-data.html
- https://blog.koehntopp.info/2020/07/27/mysql-commit-size-and-speed.html
- Lang laufende Transaktionen vermeiden
-- Rollbacks in MySQL sind recht teuer
-- weil Daten aus dem Undolog zurück in die Daten geschrieben werden müssen
-- daher Transaktionen nicht zu groß machen
-- zu kleine Transaktionen sind aber auch nicht hilfreich
-- Daten einer Transaktion wird erst mit dem COMMIT ins Binlog geschrieben
-- Replikationen werden also erst mit dem COMMIT durchgeführt
-- Wenn das undolog groß wird, wird alles langsam

## Caches und Performance
- query_cache gibt es in MySQL 8 nicht mehr
- Frage: IO-Verteilen
-- man kann beim CREATE TABLE DATA Directoriey angeben
-- Ab 8.0.21 ist die Wahl des Verzeichnisses eingeschränkt
-- Ist aber nicht empfehlenswert
-- IO-Verteilung lieber über Storage-Subsystem/LVM verwenden
-- Ausweichen auf SSD sinnvoller (Beispiel: 1 SSD ca 20.000 IOps)
-- TLC-SSD / NVME empfehlenswert
-- Für produktive Datenbanken zum Geld verdienen heute kein rotierendes Rost mehr einsetzen
- vm.swapiness auf 1 führt dazu, dass bei swap-entscheidungen als erstes den FS-Cache wegschreibt (wert 0-100, 100=apps swapen)
- TODO: numactl --interleave recherchieren
- XFS hat im Gegensatz zu extX eine planbare Performance weil es sich konstant verhält. ext macht alle X-Sekunden eine Pause zum flushen

## ACID Model
- Hinweis auf falsche Doku. Consistency bezieht sich auf Foreign Key Contraints
- Durch die MySQL Default-Einstellungen kann man den mysqld mit -9 killen, ohne dass Daten verrlohren gehen, die bereits commited wurden.
- https://blog.koehntopp.info/2020/11/27/backups-and-replication.html

## Architektur
- Grafik: InnoDB Architektur
- Clean Pages im Buffer-Pool: Pages, die mit denen in den Tablespace Dateien übereinstimmen
- Dirty Pages im Buffer-Pool: Pages mit änderungen aber noch nicht im Tablespace gespeichert (Commit steht schon im Redo-Log)
- Checkpoints schreiben Dirty-Pages aus dem Buffer-Pool auf das Storage
- Checkpoints werden regelmässig erstellt
- Checkpoints werden bei vollem Buffer-Pool erstellt
- Checkpoints bei vollem Redo-Log
-- immer 2 Dateien
-- werden als erstes beim initiieren der DB angelegt

