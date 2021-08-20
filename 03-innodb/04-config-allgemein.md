# Config Allgemein

## OS Tuning

Weil wir den File System Buffer Cache nicht nutzen, können wir vm.swappiness runterdrehen. Je nach Kernelversion auf 0 oder 1.

```bash
$ cat /etc/sysctl.conf
vm.swappiness = 1
```

Für die Daten-Platte von MySQL sollte der `noop` Scheduler verwendet werden.
Das ist inzwischen auch meist der Default.
Eingestellt wird er in `/sys/block/*/queue/scheduler`.
Leider kann der Change nicht via `sysctl` eingestellt werden (das ist für /proc), sondern muß als Kernel Bootoption übergeben werden (`elevator=noop`).

## RAID

Wenn man Replikation hat, und sicher ist, immer genug Replicas zu haben, dann braucht man kein RAID (bzw kann RAID-0 verwenden).
Wenn man auf das Überleben der Datendisk angewiesen ist, sollte man RAID-1 oder RAID-10 verwenden.

Nie RAID 5:

RAID-5, RAID-6, Erasure Coding oder andere Dinge sind mit großer Wahrscheinlichkeit wenig performant, da MySQL faktisch nur 16 KB oder 1 KB Randwrites macht.
Dazu kommt die unglaublich schlechte Performance von RAID-5 und Freunden im Degraded State.

Man will auf jeden Fall eine Hot Spare, aber bei RAID-5 und Freunden ist sie *zwingend* und die Rekonstruktion muß mit der höchsten möglichen Performance laufen, auch wenn dies den Produktions-SLO verletzt.
Das ist so, weil die Read Performance von RAID-5 und Freunden im Degraded State sowieso nutzlos schlecht ist.

## File System Options

MySQL funktioniert am Besten auf XFS, weil das planbare Performance hat.
Wir wollen das System mit `relatime` und `bigtime` (5.10 oder höher) verwenden.
`relatime` vermeidet atime-Updates im Dateisystem (`noatime` würde sie ganz abschalten, bringt aber kaum Performancegewinne).
`bigtime` verwendet 64 Bit time_t Werte und vermeidet das 2038 Problem.

Wird ein Controller oder eine NVME Flash mit Batterie/Powercap verwendet, kann man `nobarrier` konfigurieren.
Das muß man jedoch unbedingt testen (oder halt beim Default, `barrier` bleiben).
Für NVME ist das meist auch kein Problem das drin zu lassen.

### ZFS

ZFS ist ein Dateisystem, das nie Daten überschreibt und seinen Datenbaum als Merkle-Tree versiegelt und signiert.
Für MySQL bedeutet das, daß eine Reihe von Dingen bei ZFS und bei MySQL zu konfigurieren ist.

Eine sinnvolle Übersicht bei [shattered silicon](https://shatteredsilicon.net/blog/2020/06/05/mysql-mariadb-innodb-on-zfs/).

Die Empfehlung ist immer noch, XFS zu verwenden.

## NUMA

Das Kommando `numactl --hardware` zeigt die Struktur der Hardware an.
Hat man mehr als eine NUMA-Node, muß man sehen, daß der Speicher korrekt verwendet wird:
Bei NUMA sitzt der Speicher hinter der CPU, und zum Zugriff auf den Speicher einer anderen CPU muß die eigene CPU über den QPI-Link auf den Speicher einer anderen CPU zugreifen.
Programme wie Intels vTune (kostenpflichtig) können das messen und zeigen.

Kleine Prozesse (kleiner als der Speicher einer einzelnen NUMA-Node) sollten "lokal" liegen, um QPI-Traffic zu vermeiden.
Zum Beispiel starten wir `nginx` mit uwsgi-Workern in einer "Zerg" Konfiguration, wo pro Xeon Sockel ein Satz uwsgi-Worker läuft, die jeweils mit `membind` und `cpunodebind` auf ihre Node festgenagelt sind.
Ein Satz Worker läuft also mit `numactl -M 0 -N 0`, der andere mit `numactl -M 1 -N 1`.
Dadurch wird der Speicherzugriff enorm beschleunigt und der QPI-Link frei gehalten.

Dasselbe tun wir mit unseren KVMs.
Und aus diesem Grund sind alle KVMs deutlich kleiner als eine NUMA-Node.

Für MySQL und andere sehr grosse Prozesse gilt das nicht.
Hinzu kommt, daß MySQL in der Regel IO-Bound ist, und der Zugriff auf die Platte langsamer ist als ein Zugriff auf langsames RAM.
Von MySQL wollen wir also, daß es allen Speicher nutzt, auch "schlechten" Speicher auf der anderen NUMA-Node.

MySQL 8 kennt NUMA und kann damit umgehen.
Der Buffer Pool wird so alloziert, daß er sich über alle NUMA Nodes erstreckt, der Rest wird bevorzugt lokal alloziert.
Es ist also nichts einzustellen.

Bei älteren MySQL Versionen muß man `mysql.server` editieren und dafür sorgen, daß statt `mysqld` die Variante `numactl -i all mysqld` gestartet wird.

Bei MySQL in einer KVM ist es nutzbringend, wenn man dafür sorgt, daß die KVM NUMA-aware gestartet wird. In der KVM ist dann nichts zu tun.

Siehe dazu auch [MySQL Swap Insanity](https://blog.jcole.us/2010/09/28/mysql-swap-insanity-and-the-numa-architecture/).

