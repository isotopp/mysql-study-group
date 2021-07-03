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

### Also

- Zum Spielen auf Mac: Homebrew
- Zum Spielen auf Windows: Das MSI von Oracle
- Zum Spielen auf Linux: dbdeployer und dann ein Sortiment Versionen

- Produktion auf was mit RHEL Stammbaum

## EOL Liste

- Externe Quelle [End Of Life](https://endoflife.software/applications/databases/mysql)
- Schwerer lesbar: [Oracle selbst](https://www.mysql.com/support/eol-notice.html)


- Versionen: DMR (Developer Milestone Release), RC (Release Candidate), GA (offiziell)

- 8.0 nicht downgradeable (vorher: innerhalb einer Release-Series stabiles Binärformat)





Für Produktion hoffentlich das RPM, oder halt das Deb (schlechte Erfahrungen in der Vergangenheit)

Oft ein problem: Autostart des Dienstes nach installation eines Updates.
- Trick: Installation einer existierenden /etc/mysql/my.cnf mit einem Syntaxfehler, der einen Start verhindert.

Zum Testen auf dem eigenen Rechner:
- Homebrew?
- Docker?
- dbdeployer: Consultants und Leute, die Szenarien mit bestimmten Versionen nachstellen müssen.
