# Ein Blick in Production

Eine etwas ältere Blade, die verwendet wird, um verschiedene Datenbanken zu hosten, die von den DBAs verwendet werden.

## CPU

```bash
$ lscpu
Architecture:        x86_64
CPU op-mode(s):      32-bit, 64-bit
Byte Order:          Little Endian
CPU(s):              32
On-line CPU(s) list: 0-31
Thread(s) per core:  2
Core(s) per socket:  8
Socket(s):           2
NUMA node(s):        2
Vendor ID:           GenuineIntel
CPU family:          6
Model:               79
Model name:          Intel(R) Xeon(R) CPU E5-2620 v4 @ 2.10GHz
Stepping:            1
CPU MHz:             2530.577
CPU max MHz:         3000.0000
CPU min MHz:         1200.0000
BogoMIPS:            4190.32
Virtualization:      VT-x
L1d cache:           32K
L1i cache:           32K
L2 cache:            256K
L3 cache:            20480K
NUMA node0 CPU(s):   0,2,4,6,8,10,12,14,16,18,20,22,24,26,28,30
NUMA node1 CPU(s):   1,3,5,7,9,11,13,15,17,19,21,23,25,27,29,31
Flags:               fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush dts acpi mmx fxsr sse sse2 ss ht tm pbe syscall nx pdpe1gb rdtscp lm constant_tsc arch_perfmon pebs bts rep_good nopl xtopology nonstop_tsc cpuid aperfmperf pni pclmulqdq dtes64 monitor ds_cpl vmx smx est tm2 ssse3 sdbg fma cx16 xtpr pdcm pcid dca sse4_1 sse4_2 x2apic movbe popcnt tsc_deadline_timer aes xsave avx f16c rdrand lahf_lm abm 3dnowprefetch cpuid_fault epb cat_l3 cdp_l3 invpcid_single pti ssbd ibrs ibpb stibp tpr_shadow vnmi flexpriority ept vpid ept_ad fsgsbase tsc_adjust bmi1 hle avx2 smep bmi2 erms invpcid rtm cqm rdt_a rdseed adx smap intel_pt xsaveopt cqm_llc cqm_occup_llc cqm_mbm_total cqm_mbm_local dtherm ida arat pln pts md_clear flush_l1d

$ free -m
total        used        free      shared  buff/cache   available
Mem:         128819       32354       78663        1458       17801       94164
Swap:           998           0         998

$ sudo numactl -H
available: 2 nodes (0-1)
node 0 cpus: 0 2 4 6 8 10 12 14 16 18 20 22 24 26 28 30
node 0 size: 63157 MB
node 0 free: 36263 MB
node 1 cpus: 1 3 5 7 9 11 13 15 17 19 21 23 25 27 29 31
node 1 size: 63209 MB
node 1 free: 42388 MB
node distances:
node   0   1
0:  10  21
1:  21  10
```

Dies ist eine [Dual Socket E5-2620v4](https://ark.intel.com/content/www/us/en/ark/products/92986/intel-xeon-processor-e5-2620-v4-20m-cache-2-10-ghz.html).

Die E5-2620v4 ist eine $420 CPU von Intel, 8C/16T, [2.10-3.00 GHz](https://en.wikichip.org/wiki/intel/xeon_e5/e5-2620_v4). 
Das resultiert in einem Server mit 16C/32T.

Wir haben 2x 64 GB drin, d.h. weil dies eine Dual-Socket Kiste ist, hat jeder Socket 64 GB Speicher "hinter" sich.
Die andere Hälfte Speicher ist "langsamer" als die lokale Hälfte Speicher. 
`numactl -H` zeigt uns das an, und auch, ob der Speicher gleichmäßig genutzt wird.

## Platte

```bash
$ sudo vgs
  VG    #PV #LV #SN Attr   VSize VFree
  sysvm   1   8   0 wz--n- 1.74t 121.21g

$ sudo lvs
  LV       VG    Attr       LSize    Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  audit    sysvm -wi-ao----    1.95g
  home     sysvm -wi-ao----    9.77g
  log      sysvm -wi-ao----   10.00g
  mysqlVol sysvm -wi-ao----    1.57t
  root     sysvm -wi-ao----   15.00g
  swap     sysvm -wi-ao---- 1000.00m
  tmp      sysvm -wi-ao----    9.77g
  var      sysvm -wi-ao----   10.00g

$ df -Th
Filesystem                 Type      Size  Used Avail Use% Mounted on
devtmpfs                   devtmpfs   63G     0   63G   0% /dev
tmpfs                      tmpfs      63G  4.0K   63G   1% /dev/shm
tmpfs                      tmpfs      63G  1.5G   62G   3% /run
tmpfs                      tmpfs      63G     0   63G   0% /sys/fs/cgroup
/dev/mapper/sysvm-root     ext4       15G  5.1G  9.0G  36% /
/dev/mapper/sysvm-home     ext4      9.6G   39M  9.1G   1% /home
/dev/mapper/sysvm-tmp      ext4      9.6G   37M  9.1G   1% /tmp
/dev/mapper/sysvm-var      ext4      9.8G  695M  8.7G   8% /var
/dev/sda2                  ext4      969M  177M  727M  20% /boot
/dev/mapper/sysvm-log      ext4      9.8G  2.8G  6.6G  30% /var/log
/dev/mapper/sysvm-audit    ext4      1.9G  6.5M  1.8G   1% /var/log/audit
/dev/sda1                  vfat      200M  6.9M  193M   4% /boot/efi
/dev/loop0                 xfs       281M   41M  241M  15% /var/lib/sss/db
/dev/mapper/sysvm-mysqlVol xfs       1.6T  120G  1.5T   8% /mysql/badmin
tmpfs                      tmpfs      13G     0   13G   0% /run/user/5105

$ lsblk
  SIZE RO TYPE  MOUNTPOINT
loop0                7:0    0 286.1M  0 loop  /var/lib/sss/db
sda                  8:0    0   1.8T  0 disk
├─sda1               8:1    0   200M  0 part  /boot/efi
├─sda2               8:2    0  1000M  0 part  /boot
└─sda3               8:3    0   1.8T  0 part
  ├─sysvm-root     253:0    0    15G  0 lvm   /
  ├─sysvm-swap     253:1    0  1000M  0 lvm
  │ └─swap         253:8    0   999M  0 crypt [SWAP]
  ├─sysvm-audit    253:2    0     2G  0 lvm   /var/log/audit
  ├─sysvm-log      253:3    0    10G  0 lvm   /var/log
  ├─sysvm-var      253:4    0    10G  0 lvm   /var
  ├─sysvm-tmp      253:5    0   9.8G  0 lvm   /tmp
  ├─sysvm-home     253:6    0   9.8G  0 lvm   /home
  └─sysvm-mysqlVol 253:7    0   1.6T  0 lvm   /mysql/badmin
```

Lokaler Speicher, die Gerätenamen legen eine NVME SSD nahe, und nicht eine SSD an einem HP SmartArray oder Dell PERC (ich muß noch einmal nachsehen gehen).

Wir verwenden LVM2, dort dann alle MySQL-Dinge in ein extra Volume `mysqlVol`.

Der Mount geht nach `/mysql/<name der replication chain>`.

Dort drin dann

```bash
$ mkdir -p /mysql/<chain>/{data,log,tmp}
```

MySQL wird gestartet mit `datadir=/mysql/<chain>/data`, BinLogs gehen nach `/mysql/<chain>/logs` und der Server läuft mit `tmpdir=/mysql/<chain>/tmp`.

Warum der Chain-Name da drin?
Der Gedanke war früher, daß wir eventuell mal den Fall haben können wo eine LVM Gruppe eines anderen Servers wie auch immer herein importiert werden kann, und dann will ich `/mysql/{chain1,chain2}/data` haben können.

Was würde ich heute anders machen?
Ich würde die my.cnf auch da hin tun, damit sie auf dem LVM liegt und auch mit gesichert wird.
In einer Weise löst MySQL 8 das Problem durch die persistente Config, die ja in [`$datadir/mysqld-auto.cnf`](https://dev.mysql.com/doc/refman/8.0/en/persisted-system-variables.html#persisted-system-variables-file-handling) liegt. 

# Generelle Überlegungen zum Setup

Bei Datenbanken ist der Schlüssel zum Tempo RAM - die Maschine sollte großzügig mit Speicher ausgerüstet werden.

[Memory Saturated MySQL](https://blog.koehntopp.info/2021/03/12/memory-saturated-mysql.html)

"Der Working Set der Maschine sollte in den Speicher passen."

"Der größte Index der größten Tabelle sollte in den Speicher passen." (macht den Data Load schneller)

CPU ist minder wichtig. 
Was sind Möglichkeiten für eine Datenbank, viel CPU zu verwenden?
- Index/Full Data Scan - Index fehlt oder ist nicht effektiv.
- Sorting - eventuell im Client machen?
- Stored Routines oder Stored Functions - wegschmeißen, neu machen. Das Setup ist rettungslos kaputt.

Storage:
- "5x mehr als die Datenbank"
  - Binlogs, Dump Space usw.
  - Bei großen Installationen oder bei Replikation: 3x mehr.
  - Warum so viel?
    Wachstum, Marge: MySQL ist extrem vergnatzt, wenn es jemals eine volle Platte sieht. Kann auch schwer zu fixen sein.
    
- "0.4ms Commit Latency" -> 2500 Commit/s sequentiell

```php
# 2500 Loops pro Sekunde
foreach ($alldata as $id => $data) {
  $db->insert($id, $data);
  $db->commit();
}    
```  
 - früher:
    - drehender Rost mit 5ms Latencies
    - RAID mit wide-stripes oder JBOD mit striping
    - Raid-Controller mit CPU und Write-Cache und Battery-Buffer
    
  - heute:
    - mit NVME hat man massiven IOPs Überfluß und minimale Latencies
    - partitionieren/virtualisieren für maximale parallelität
    - Raid-Controller machen Dinge langsamer! 


## Ohne Replikation ist es defekt

MySQL ohne Replikation gibt es nicht.
Replikation erlaubt es Dir, unterbrechungsfrei Operations durchzuführen.

- Replica für Backup
- Replica zum Versionen testen
- Replica für unterbrechungsfreies Upgrade
  - "Habe mindestens eine Maschine mehr als Du brauchst" (VMs sind billig, mach Dir mehr davon!)
- Mit Replikation ist RAID sehr optional (wir fahren alle Datenbanken mit JBOD + LVM2)

## XFS, nicht ext4

Wir verwenden in `/mysql/<chain>` immer XFS.

Ein Projekt bei MySQL AB mit einem sehr großen Kunden hatte enorme Probleme, stabile Performance zu erreichen. 
Es stellt sich heraus, daß `ext4` doppelt so schnell ist ("eine halb so große Commit-Latenz hat") wie `xfs`.
Jedoch kommt es alle paar Sekunden zu einem Stall, in dem `ext4` irgendwelche Buffer im Hintergrund raus schreibt und eine Folge von Commits stauen sich. 
`xfs` zeigt dieses Verhalten nie: Die Commit-Latenz hat fast keinen Jitter.

Wir designen Systeme nicht für den Durchschnitt, sondern für den (schlechtesten) Extremfall.
Es ist also wichtig, gleichmäßige Performance zu erreichen, und dann kann man versuchen, diesen gleichmäßigen Performancefall zu verbessern.

Indem wir `xfs` verwenden können wir einigermaßen sicher sein, daß bester und schlechtester Performancefall beim Schreiben auf Disk nahe beieinander liegen.

## LVM2 unten drunter

LVM2 als Lage unter allen Datensystemen gibt uns eine Menge Flexibilität: 
Wir können Partitionen aus zugesteckten Drives zusammenkleben, erweitert, nachträglich mirrorn und sogar (mit dem extrem schlechten `lvsnapshot`) snapshotten, wenn nicht zu viel Last ist.

Wir machen alle Setups mit so viel LVM2 als möglich.

## ZFS, btrfs?

ZFS und btrfs sind möglich und eventuell sogar vorteilhaft, wenn die Hardware es her gibt (ohne Flash braucht man nicht anzutreten) und wenn man den Konfigurationstanz machen kann und will.

## Blockgrößen und Alignment

Durch das ganze Layering beim Storage kann es vorkommen, daß

- Partitionsanfänge
- Blöcke und
- Readaheads

nicht aligned oder gleich sind.
Das muß man ggf prüfen und mal validieren, sonst hat man Read Amplification und das wäre schlimm.

