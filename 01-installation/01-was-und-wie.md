# Installation

Mögliche Plattformen:
- Linux
- Windows
- MacOS

- Solaris, FreeBSD, "from Source"

Tatsächliche Liste [auf den Supportseiten](https://www.mysql.com/support/supportedplatforms/database.html).

- "Red Hat" (Oracle Linux 8, RHEL 8, CentOS 8)
- "Solaris 11"
- Ubuntu "current LTS" (und leider auch ältere)
- SLES 15, OpenSuse 15.2 und leider auch 12
- Debian 10
- Windows 2019 Server, 2016 Server, 2012 Server R2
- Windows 10 (LOL)
- MacOS 11 und 10.15
- FreeBSD 12

## Anmerkungen dazu

Es werden 32-Bit Versionen unterstützt. Das ist in 2021 lächerlich und eine Datenbank mit 3 GB RAM (Windows: 2GB) braucht niemand irgendwo.

Oracle Linux ist RHEL ist CentOS. Das ist besser als Ubuntu, was wiederum besser als Debian ist. Und das MySQL Binary ist weitaus besser und neuer als das aus der Distro.

Solaris und FreeBSD sind sehr wenig verbreitet. Wenn ich sie im Feld angetroffen habe, habe ich jedes Mal plattformspezifische Fehler gefunden und heim gebracht. Nicht empfohlen.

In der Vergangenheit hatte der Debian Maintainer sehr eigene Vorstellungen davon, wie Dinge funktionieren, die MySQL Supportprozesse zerbrochen haben und nicht skalierten ("myisamcheck single-threaded im Startscript statt paralleles REPAIR TABLE bei Bedarf im Server"). Das ist inzwischen vermutlich besser (es sind 15 Jahr vergangen), aber ich habe immer noch vorbehalte. Nehmt ein RHEL-isches System.

Apple Filesystem ist untauglich für Produktion, klar.

Unsere "Standardblade" ist eine Dual Silver-4110 (16C/32T) mit 128 GB RAM und einer oder zwei 1.92TB Micron NVME sowie einer (Intel-) 10 Gbit/s Karte. Die kommt bei ca. 120-150 Euro im Monat (Capex-Abschreibung auf 5a, plus Platz, Netz und Strom im RZ auf 5a).

Vergleiche das mit AWS RDS db.m5.4xl (1/2 solche Blade), plus EBS, Netz und S3, und mit m5.4xl + EBS + Netz + S3.

### Windows

Wo?
- C:\Program Files\MySQL\MySQL Server 8.0\{bin,docs,include,lib,share}
- %PROGRAMDATA%\MySQL\MySQL Server 8.0\ (C:\ProgramData) - unbedingt umstellen

Config?
- Hier my.ini, nicht my.cnf
- Lage irgendwo in SYSTEM32 oder andere wilde Orte -> unbedingt mit Option bei der Servicedefinition angeben.

Betrieb?
- Alle möglichen Restriktionen (Ports limitiert, DROP DATABASE vs. File Locks, ^Z in Files und Pipelines, \ Zeichen und so weiter)

Service?
- Register als Service 
- mysqld.exe Servicename --install --defaults-file=...my.ini, dann sc start/stop service (vormals net start/stop)
- Löschen mit sc delete name oder mysqld --remove name

Doku?
- https://dev.mysql.com/doc/refman/8.0/en/windows-start-service.html

"Don't."

Server?
- Lange Liste von Windows Server Versionen oben
- Einsatz bei einem Kunden in Karlsruhe: Kein Server-Windows auf Kassensystemen verwendet, weil kein Speicher vorhanden (teuer, Kosten mal Anzahl der Kassen). MySQL 2x langsamer als IBM DB/2 und MS SQL-Server. Auf Windows Server sind alle gleich langsam. Wieso?
- Nach langem Tracing: Auf Nicht-Server Windows sind manche Varianten des Windows-Äquivalentes von fsync()/fdatasync() als NO-OP implementiert. Das passiert im Kerneltreiber, der den Platten Filtertreiber aufruft. MySQL verwendet eine andere, ältere Weise des Plattensyncs, die immer funktioniert ("Kasse kann jederzeit hart ausgeschaltet werden"), aber die Performanceziele nicht erfüllt.
- Die Kassen wurden massiv mit Hardware aufgerüstet, das hat das Projekt sehr verteuert.

### Also

- Zum Spielen auf Mac: Homebrew
- Zum Spielen auf Windows: Das MSI von Oracle
- Zum Spielen auf Linux: dbdeployer und dann ein Sortiment Versionen

- Produktion auf was mit RHEL Stammbaum

## EOL Liste

- Externe Quelle [End Of Life](https://endoflife.software/applications/databases/mysql)
- Schwerer lesbar: [Oracle selbst](https://www.mysql.com/support/eol-notice.html)

## Stabilität

- DRM ("Developer Milestone Release"), eine Art Alpha, Spielzeug zum Testen neuer Features, leicht radioaktiv.
- RC (Release Candidate), eine Art Beta, "als Replica in Produktion mitlaufen lassen und Probleme melden, sonst sind die hinterher im Release"
  - GA (General Availability), ein Release

*Achtung:* 8.0 nicht notwendigerweise innerhalb der Serie downgradeable (vorher: innerhalb einer Release-Series stabiles Binärformat)

## Trickery

Einige Packager starten den Dienst automatisch nach der Installation des Paketes.

Das ist ein unglaublicher Nerv, weil es die Ansibilisierung von Dingen erschwert und die klassische Trifekta kaputt machen ("Paket installieren", "Config instantiieren", "Dienst starten"). Der Dienst startet bevor Config da ist und macht wilde Dinge, ist eventuell auch ungesichert.

Ich kacke da gerne vorher eine defekte `/etc/mysql/my.cnf` ins System, die einen Start des Dienstes ohne weitere Parameter verhindert.

# Selber Testen

Auf meinem Mac: Homebrew, Docker

Auf meinem Testserver: [`dbdeployer`](https://github.com/datacharmer/dbdeployer). Damit kann ich ein MySQL-Museum ohne Container betreiben und auch schnell Replikations-Setups durchtesten.

