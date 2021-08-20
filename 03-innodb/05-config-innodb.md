# InnoDB Data Structures and Config

- [In memory structures](https://dev.mysql.com/doc/refman/8.0/en/innodb-in-memory-structures.html)
- [On disk structures](https://dev.mysql.com/doc/refman/8.0/en/innodb-on-disk-structures.html)

## Buffer Pool

Der meiste Speicher in InnoDB geht in den Buffer Pool.
Der Buffer Pool enthält InnoDB Pages (16 KB) aus Tablespaces von der Platte.
Sind Plattenabbild und Speicherabbild identisch, heißt die Page `clean`, sonst ist sie `dirty`.
Clean Pages können jederzeit im Speicher freigegeben werden, denn dadurch gehen keine Daten auf der Platte verloren.

## Buffer Pool und Writes (Transaktionen)

Dirty Pages müssen zurück geschrieben werden, damit das Abbild auf der Platte wieder mit dem Bild im Speicher übereinstimmt.
Die Änderungen einer Dirty Page im Speicher gegenüber ihrer älteren Version auf Disk wird im Redo Log auf der Platte geloggt, in einer Binärdiff, dem Redo Log Entry.

Crashed der Rechner, bevor eine Dirty Page zurück geschrieben wird, kann ihr Zustand durch eine Redo Log Recovery wiederhergestellt werden.
Dazu wird die Version der Page von der Platte geladen, die Transaktionsnummer gelesen und danach werden alle Transaktionen aus demn Redo Log angewendet, die seitdem geschehen sind und die diese Page betreffen.

Das bedeutet auch, daß die Redo Log Entries für diese Page nicht freigegeben werden können bis die Dirty Page zurück geschrieben worden ist.
Da das Redo Log endliche, feste Größe hat, kann es also dazu kommen, daß MySQL zwangsweise dirty Pages zurück schreiben muß, um Platz im Redo Log zu schaffen.

Das Zurückschreiben von Dirty Pages in die Tablespaces heißt Checkpoint, und in der Regel ist eine Page im Speicher durch mehrere Transaktionen modifiziert worden, bevor sie gecheckpointed wird.
Eine große Anzahl von Dirty Pages (man kann in Grenzen konfigurieren, wie viele das sein können) bewirkt also, daß Transaktionen auf einer Page zusammengefaßt werdne können und Disk Writes gespart werden.

## Buffer Pool und Reads

Der Buffer Pool dient auch als Read Cache unter Kontrolle der Anwendung.
Weil MySQL mehr über die Daten weiß als das Betriebssystem ist es sinnvoller Daten in der Anwendung zu cachen als im File System Buffer Cache.
Der File System Buffer Cache wird von InnoDB also nicht genutzt.

## Buffer Pool Management

Der InnoDB Buffer Pool ist ein LRU mit Midpoint-Insertion.
Pages gehen am Midpoint in die LRU-Liste und können dann basierend auf 2. Verwendung und Mindestabstand (Zeit) an die Spitze befördert werden.

MySQL kann die Page-Nummern der Pages im Buffer Pool regelmäßig abspeichern und eim Start des Servers neu laden.
Dadurch startet der Server mit warmem oder zumindest nicht ganz kaltem Pool.

Ein sehr großer InnoDB Buffer Pool kann in mehrere Sub-Pools aufgespalten werden.
Das verhindert Lock Contention.
Es ist sinnvoll bei Pools > 16 GB.

## Status

`SHOW ENGINE INNODB STATUS\G`

Braucht das \G wegen der Länge.
Braucht `pager less -SE`, wegen der Länge.

Besser als `SHOW ENGINE INNODB STATUS` ist es `INFORMATION_SCHEMA zu verwenden.
Die Daten da sind besser parsebar, und inzwischen auch detaillierter.
Die meisten `SHOW ENGINE INNODB STATUS`-Variablen findet man in [I_S.INNODB_BUFFER_POOL_STATS](https://dev.mysql.com/doc/refman/8.0/en/innodb-information-schema-buffer-pool-tables.html#innodb-information-schema-buffer-pool-stats-example).

# Was konfigurieren

## Größe

So groß als möglich ohne daß die Kiste swappt.
Es ist wie 17+4 spielen.

Variablen:

- `innodb_buffer_pool_size` - Gesamtgröße vom Buffer Pool. Kann inzwischen zur Laufzeit geändert werden, aber da das checkpointen kann, kann das dauern.
- `innodb_buffer_pool_chunk_size` - wenn man die Gesamtgröße zur Laufzeit anpasst, passiert das automatisch in Chunk Size Schritten, damit das System sinnvoll checkpointen kann.
- `innodb_buffer_pool_instances` - die Anzahl der Sub-Pools, die generiert werden. Gesamtgröße muß ein ganzzahliges Vielfaches von `innodb_buffer_pool_chunk_size` mal `innodb_buffer_pool_instances` sein. Es gibt keinen Mechanismus, der das erzwingt, aber wenn es nicht stimmt, passieren unintendierte Dinge, d.h. es wird wild auf- oder abgerundet.

Wir haben 128 GB RAM in einer Standardblade, und setzen zum Beispiel 100 GB Buffer Pool als 25 Instances zu 4 GB Größe, und verwenden die Standard Chunk Size.

Chunk Size kann nicht zur Laufzeit geändert werden. Gesamtgröße und Anzahl der Pools schon, aber halt immer nur auf Chunk-Grenzen.

Resizing interagiert mit laufenden Transaktionen und mit Checkpointing, dauert also.
Man kann in der Status Variable `Innodb_buffer_pool_resize_status` nachsehen, was gerade passiert, oder das Log tailen:

```bash
# Oder was mit "watch"
while :
do
  mysql -e 'show status where Variable_name = "Innodb_buffer_pool_resize_status"'
  sleep 1 
done
```

oder `tail -F /var/log/mysql.err`

Obwohl der Server weiterläuft, ist die Resize-Operation in weiten Teilen blocking.
Der Zugriff auf den Server stalled also, aber immerhin bleibt der Cache warm und die Verbindungen halten.

`innodb_buffer_pool-size` ist die eine Sache, die man korrekt konfigurieren muß, damit überhaupt was geht.

## Scan Resistance

Was ist ein LRU und wie funktioniert er?
Was ist Scan Resistance, und wieso braucht man das?

- `innodb_old_blocks_pct` - Anzahl der Buffer Pool Pages, die der unteren Hälfte (unterhalb des Midpoints) zugeordnet werden. 
- `innodb_old_blocks_time` - Eine Page muß mindestens so viele ms im Old Pool sein, bevor ein 2. Zugriff sie in den New Pool ziehen kann.

Wir konfigurieren das in der Regel nicht.

## Readahead

Tablespaces sind aus Extents zusammengebaut, das sind in der Regel 64 pages (1 MB bei 16 KB Pages).
Wenn genug Pages aus einem Extent gelesen werden, oder wenn genug Pages in Reihe gelesen werden, triggert InnoDB einen Readahead.

- `innodb_read_ahead_threshold` - wenn mehr als so viele Pages in Reihe gelesen werden, wird der ganze Extent rein gesaugt (Default 56).
- `innodb_random_read_ahead` - wenn der Schalter auf 1 gesetzt wird, wird der ganze Extent rein gesagt, wenn mehr als 13 aufeinanderfolgende Pages des Extent bereits gelesen worden sind.

Wir konfigurieren das in der Regel nicht.

## Checkpointing (Flushing)

Checkpointing wird von den InnoDB Page Cleaner Threads durchgeführt.

- `innodb_page_cleaners` - Anzahl der Page Cleaner Threads (4), maximal Buffer Pool Instances viele (einer pro Pool).
- `innodb_max_dirty_pages_pct_lwm` - Cleaner startet, wenn die Anzahl der Dirty Pages den Low Water Mark Level erreicht ("early flushing", 10%).
- `innodb_max_dirty_pages_pct` - Cleaner checkpointed richtig aggressiv, wenn die Anzahl der Dirty Pages diesen Level erreicht (90%).
- `innodb_flush_neighbors` - 0 (off) für SSD, NVME, 1 (on) flusht zusammenhängende Pages im Extent, 2 (hard on) flusht alle Pages im Extent.
- `innodb_lru_scan_depth` - wie weit in der LRU der Cleaner voraus gucken soll (quasi die Batch Size pro Sekunde). Es werden scan depth * pool instances viele Pages pro Sekunde erwogen.

Wir tunen diese Variablen in der Regel nicht (Ausnahmen sind sehr, sehr write intensive Workloads).
Das ist alles ganz furchtbares Gefummel.

Wir verlassen uns stattdessen auf das Adaptive Checkpointing:

Man will so viele Dirty Pages haben, wie man sich leisten kann, weil Dirty Pages Writes zusammenlegen.
Das findet seine Grenzen aber in einer Redo Log Recovery (dauert länger je mehr Dirty Pages), und in der Größe des Redo Logs (darf nie ganz voll sein, sonst aua).

Man will auch keine plötzlichen scharfen Checkpoints mit Panik-Flushing, weil entweder zu viele Dirty Pages da sind oder das Redo Log voll ist.
In beiden Fällen würden Writes stallen, und das gibt Mecker von den Anwendern.

Adaptive Flushing dreht also die Checkpointing-Rate dynamisch hoch, je mehr Dirty Pages es findet und je voller das Redo Log wird.

- `innodb_adaptive_flushing` - Schalter, an oder aus. Will "an", dringend, und das ist auch der Default.
- `innodb_adaptive_flushing_lwm` - Low Water Mark für Redo Log Füllstand. Adaptive Flushing geht IMMER an, wenn diese Marke passiert wird. Die HWM ist immer 75%.
- `innodb_flushing_avg_loops` - "Dämpfung", niedriger Wert = schnelle Anpassung von Adaptive Flushing, hoher Wert = langsame Anpassung.
- `innodb_io_capacity` - Anzahl der IOPS, die das Disk Subsystem angeblich hat und auf der Adaptive Flushing seine Sensitivität berechnet.
- `innodb_idle_flush_pct` - Manche Leute wollen nicht, daß die Datenbank aus Langeweile checkpointed. idle_flush_pct ist die IO Capacity, die angenommen wird, wenn Idle Flushing aktiv ist (Default: 100).

Wir konfigurieren diese Werte in der Regel nicht, mit Ausnahme von `innodb_io_capacity`.

## Buffer Pool Dumps

Wenn man die Datenbank startet, ist die `VIRT` Anzeige im `ps` sehr groß, aber `RES` ist noch sehr klein:
Viel Speicher ist beim Betriebssystem bestellt, aber noch nicht benutzt worden, also auch noch nicht comitted.
Die Datenbank ist kalt und muß beim ersten Zugriff erst mal Daten von der Disk in den Buffer Pool laden.

Das muß nicht sein.
Man kann vor dem Stop oder in regelmäßigen Intervallen darum Bitten, eine Liste der Page# in eine Datei in Datadir zu schreiben.

Es ist nicht sinnvoll, den gesamten Buffer Pool zu sichern.
Meist reichen so 25-50%, damit die Datenbank nicht ganz kalt startet.
Der Dump erfolgt in LRU Reihenfolge, es werden also die ganz heißen Pages gedumpt.

Laden der Seiten beim Start erfolgt so in großen Batches, der Warmup ist also schneller als ein natürlicher Warmup.

- `innodb_buffer_pool_dump_pct` - Prozent des Pools, der gedumpt werden soll (Default: 25).
- `innodb_buffer_pool_filename` - Datei, in der das gespeichert wird (Default: ib_buffer_pool)
- `innodb_buffer_pool_dump_at_shutdown` - Schalter (Default: ON)
- `innodb_buffer_pool_load_at_startup` - Schalter (Default: ON)
- `innodb_buffer_pool_dump_now` - Trigger (Auf 1 setzen startet einen Dump)
- `innodb_buffer_pool_load_now` - Trigger (sinnlos, nie verwenden)
- `innodb_buffer_pool_load_abort` - Trigger (bricht einen Load ab)

Statusvariablen `Innodb_buffer_pool_dump_status`, und `Innodb_buffer_pool_load_status`.
Es gibt auch `P_S` Instruments dafür, das wäre vorzuziehen.

Es ist sinnvoll, den Buffer Pool einmal pro Tag zu dumpen.
Es ist nicht sinnvoll, den Buffer Pool öfter als einmal pro Stunden zu dumpen.

## I_S Tabellen

```mysql
mysql> SHOW TABLES FROM INFORMATION_SCHEMA LIKE 'INNODB_BUFFER%';
+-----------------------------------------------+
| Tables_in_INFORMATION_SCHEMA (INNODB_BUFFER%) |
+-----------------------------------------------+
| INNODB_BUFFER_PAGE_LRU                        |
| INNODB_BUFFER_PAGE                            |
| INNODB_BUFFER_POOL_STATS                      |
+-----------------------------------------------+
```

Sowohl `INNODB_BUFFER_PAGE` als auch `INNODB_BUFFER_PAGE_LRU` sind potentiell giftig: Zugriff auf die Tabellen lockt den Buffer Pool, weil der Zugriff exakte und konsistente Daten haben will.
Auf einem großen und beschäftigten Server bringt das Monitoring so den Server um.
NIE, NIE, NIE in Produktion machen.

(Generell gilt: P_S lockt nicht, I_S lockt möglicherweise. P_S Daten sind ungenau, weil sie sich beim Angucken ändern können)

Zugriff auf I_S.IBPS ist sicher und ersetzt `SHOW ENGINE INNODB STATUS` weitgehend.

## Change Buffer

Batched Änderungen an Secondary Indexes.

- `innodb_change_buffering` - all (Default), none, inserts, deletes, changes, purges
- `innodb_change_buffer_max_size` - als Prozent vom Buffer Pool (Default: 25, Max: 50)

Nix zu löten an der Holzkiste.

## AHI

InnoDB kann mehr Speicher verbrauchen als es auf Disk braucht.
Dazu wandelt es bestimmte Teile des B+ Trees in Hashes im Speicher um (AHI).

AHI hatte mal ein Global Lock.
Das hat unter Last zu Stalls geführt (Flash Piles).
Also haben wir den AHI vielfach aus.

In MySQL 8 ist AHI Partitioned.
Ich habe keine Ahnung, ob das was verbessert.
Sollte es eigentlich.

- `innodb_adaptive_hash_index_parts` - Default: 8, Max: 512.

## Log Buffer

Transaktionen werden von jeder Connection im RAM im Log Buffer vorbereitet und dann beim COMMIT rausgeschrieben.
Dazu alloziert jede Connection einen Puffer, der maximal `innodb_log_buffer_size` groß werden kann (Default: 16 MB).

Transaktionen können größer als ein Log Buffer werden.
Dann müssen sie aber ohne COMMIT vorab ins Redo Log geflushed werden.
In einer Redo Log Recovery wird alles Recovered, was im Redo Log steht.
Am Ende fehlt das Commit, es wird also ein ROLLBACK gemacht.
Das ist eine Undo Log Recovery und nicht sehr schnell.

Log Buffer sollte so gross sein, daß ein p90 von allen Transaktionen da rein passt.
Man passt besser die Anwendung als den Log Buffer an.

`innodb_flush_log_at_commit` bestimmt, was beim COMMIT passiert.
Der Default, 1, schreibt auf Disk.
Das ist ACID-Durable, der Default und empfohlen.

Man kann den Wert auf 2 setzen.
Dann schreibt der COMMIT in den Linux File System Buffer Cache und einmal pro Sekunde auf die Platte (fdatasync)
Das ist bei langsamen Platten sehr viel schneller ("Yolo-Mode").
Bei einem Crash des mysqld sind keine Daten verloren.
Bei einem Crash des Rechners ist unter Umständen eine Sekunde verloren.
Das ist als Catchup-Wert bei laggenden Replicas unter Umständen sinnvoll, zurückstellen nicht vergessen.

Man kann den Wert auf 0 setzen.
Dann macht COMMIT gar nix (nur logisch).
Das bringt gegenüber 2 keine Zeitvorteile.
Nicht verwenden.

`innodb-flush-log-at-timeout` - Wenn ich oben "eine Sekunde" gesagt habe, dann meine ich eigentlich dies hier.
Man kann den Timeout noch höher drehen, für mehr Yolo.
Das bringt gegenüber dem Default keine Zeitvorteile.
Nicht verwenden.

`[innodb_flush_method](https://dev.mysql.com/doc/refman/8.0/en/innodb-parameters.html#sysvar_innodb_flush_method)` - Wenn ich oben fdatasync schreibe, dann meine ich eigentlich das.
Katalog von betriebssytemspezfischen Werten zum Synchen von Daten auf Disk.
Für Linux und XFS auf `O_DIRECT` setzen.
Seit 8.0.14 kann man auch `O_DIRECT_NO_FSYNC` nehmen.

Wir nehmen `O_DIRECT`.

## Tables

- File Per Table (immer an).
- Zentraler Tablespace, File schrumpft nie.
  - früher Undo Log, früher DoubleWrite Buffer, immer Metadata.
  - früher optional Tabellen, doll nicht empfohlen
- InnoDB Tabellen nicht kopierbar auf OS-Ebene
  - EXPORT, IMPORT
  - früher frm-Dateien und Metadata Store, divergent. Heute nur noch Metadata.
- Row Formats
- Primary Key
  - Clustered Index
- DATA DIRECTORY Clause
  - Don't.
- TABLESPACE Clause
  - Neu. Large untested.
- ALTER TABLE ... IMPORT TABLESPACE ...
  - same Version, same row format, foreign keys not checked und überhaupt. 
  - Kein FULLTEXT (Workaround existiert)
  - gut für partial restore, eventuell. Kein Transportmechanismus.
  - FLUSH TABLES ... FOR EXPORT
  - Metadaten exportiert aus system tablespace nach .cfg, dazu die .ibd.
- `innodb_autoinc_lock_mode (Default: 2). Kann Löcher erzeugen, unsafe mit SBR.
  - Wir setzen das auf 1, wo noch SBR im Einsatz ist (fast nirgends).

## Indexes

- Noch mal Clustered index
- PK ist echt Mandat. Tabellen ohne PK sind technisch gesehen standardkonform, aber sehr aua.
- SK verwendet den PK als Row Pointer. INDEX(a) ist also immer INDEX(a,id)
  - BIGINT UNSIGNED NOT NULL, nicht mehr als ca. 32 Bytes
  - `SERIAL`
- [sorted index build](https://dev.mysql.com/doc/refman/8.0/en/sorted-index-builds.html)
  - Checkpoint forced.
  - Optimizer Statistics Update needed (ANALYZE TABLE, ANALYZE TABLE ... UPDATE HISTGRAM ON, `information_schema_stats_expiry` (seconds, Default: 86400)
- Neu: `innodb_fill_factor`

## Tablespaces

- system (change buffer)
  - früher auch DD, DoubleWrite Buffer, Undo Log
  - `innodb_data_file_path=ibdata:10M:autoextend`, `innodb_autoextend_increment=8M` (Semicolon, nur ein Autoextend)
  - "raw disk partitions" (don't!)

- `innodb_file_per_table = 1`
  - Increment ist immer 4MB (außer für kleine Tabellen)

- CREATE TABLESPACE ts ("General Tablespaces")
  - CREATE TABLE t () TABLESPACE = ts
  - support limited, viele Einschränkungen

- Undo Log jetzt ein oder mehr Tablespaces
  - früher Teil vom System Tablespace
  - besseres Platzmanagement

- Temporary Tablespaces
  - Neu, früher HEAP/MEMORY TABLES, dann MyISAM Spillover, viele Nachteile (MEMORY = keine VAR* Typen)
  - Ab 8.0.16 InnoDB Tmptables (vorher: `internal_tmp_disk_storage_engine`)
  - `innodb_temp_tablespaces_dir` (datadir/#innodb_temp)
    - 10 oder mehr temp_##.ibt
    - I_S.INNODB_SESSION_TEMP_TABLESPACES und I_S.INNODB_TEMP_TABLE_INFO (implicit, explicit); I_S.FILES
  - Neu ab 8.0.22: `innodb_extend_and_initialize = 0` -> dann posix_fallocate(). Nur Linux.

## Doublewrite Buffer

- `innodb_doublewrite` - Schalter (Default: On). Off nur bei Dateisystemen, die nicht in-place Updates machen (ZFS, btrfs, bestimmte Fusion-IO)
- `innodb_doublewrite_dir` - Verlegen des Doublewrite, etwa auf ein NVME oder Fusion-IO.
- `innodb_doublewrite_files` - (Default: 2) Anzahl der #ib_..._x.dblwr Files (Max: 2x Anzahl der Buffer Pool Instances).
- `innodb_doublewrite_pages` - (Default: innodb_write_io_threads)
- `innodb_doublewrite_batch_size` - Anzahl der Pages pro Batch

Wir tunen diese Werte nicht.

## Redo Log

- `innodb_log_files_in_group` - (Default: 2)
- `innodb_log_file_size`

Redo Log Size ist das Produkt aus beiden Werten. Wir passen immer nur die `innodb_log_file_size` an.
Zielgröße ist ca. 1/8-1/6 des gesamten Buffer Pools für unsere Workload.

- Group Commit erklären.
- Redo Log Archiving weniger sinnvoll als Binlogs.

- 8.0.21: ALTER INSTANCE DISABLE INNODB REDO_LOG
  - Initial Data Load
  - Nebeneffekt von CLONE

## Undo Log

Jetzt ausgegliedert in den Undo Log Tablespace, vormals Teil vom General Tablespace.

