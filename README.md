# Stars - HackMyVM (Easy)
 
![Stars.png](Stars.png)

## Übersicht

*   **VM:** Stars
*   **Plattform:** HackMyVM (https://hackmyvm.eu/machines/machine.php?vm=Stars)
*   **Schwierigkeit:** Easy
*   **Autor der VM:** DarkSpirit
*   **Datum des Writeups:** 31. Oktober 2022
*   **Original-Writeup:** https://alientec1908.github.io/Stars_HackMyVM_Easy/
*   **Autor:** Ben C.

## Kurzbeschreibung

Das Ziel der "Stars"-Challenge war die Erlangung von User- und Root-Rechten auf einer als "Easy" eingestuften Maschine. Der Weg begann mit der Entdeckung eines Base64-kodierten Dateinamens (`poisonedgift.txt`) in einem Cookie auf dem Webserver (Port 80). Diese Datei enthielt einen beschädigten privaten SSH-Schlüssel. Eine weitere Notiz (`sshnote.txt`) gab den Hinweis, dass drei Großbuchstaben im Schlüssel durch Sterne ersetzt wurden. Mittels `crunch` wurden alle Kombinationen von drei Großbuchstaben generiert und ein Skript (oder manueller Prozess) verwendet, um den Schlüssel zu reparieren. Die korrekte Kombination (vermutlich `IBM`) erlaubte den SSH-Login als Benutzer `sophie`. Die Privilegieneskalation zu Root erfolgte durch Ausnutzung einer unsicheren `sudo`-Regel, die `sophie` erlaubte, die Gruppenzugehörigkeit der `/etc/shadow`-Datei zu ändern (`sudo /usr/bin/chgrp sophie /etc/shadow`). Dies ermöglichte das Auslesen der Passwort-Hashes. Der Root-Passwort-Hash wurde mit `john` geknackt (`barbarita`), was den direkten Login als `root` via `su` ermöglichte.

## Disclaimer / Wichtiger Hinweis

Die in diesem Writeup beschriebenen Techniken und Werkzeuge dienen ausschließlich zu Bildungszwecken im Rahmen von legalen Capture-The-Flag (CTF)-Wettbewerben und Penetrationstests auf Systemen, für die eine ausdrückliche Genehmigung vorliegt. Die Anwendung dieser Methoden auf Systeme ohne Erlaubnis ist illegal. Der Autor übernimmt keine Verantwortung für missbräuchliche Verwendung der hier geteilten Informationen. Handeln Sie stets ethisch und verantwortungsbewusst.

## Verwendete Tools

*   `arp-scan`
*   `nmap`
*   `gobuster`
*   `base64`
*   `curl`
*   `crunch`
*   `ssh`
*   `john` (John the Ripper)
*   `sudo`
*   `chgrp`
*   `cat`
*   `su`
*   `vi` (impliziert)
*   `mv` (impliziert)
*   `sed` (impliziert im Skript `stars.sh`)
*   `bash` (Skripting)
*   Standard Linux-Befehle (`echo`, `ls`)

## Lösungsweg (Zusammenfassung)

Der Angriff auf die Maschine "Stars" gliederte sich in folgende Phasen:

1.  **Reconnaissance & Enumeration:**
    *   IP-Findung mit `arp-scan` (`192.168.2.119`).
    *   `nmap`-Scan identifizierte offene Ports: 22 (SSH - OpenSSH 8.4p1) und 80 (HTTP - Apache 2.4.51 mit Titel "Cours PHP & MySQL").

2.  **Web Enumeration & Hinweis-Findung:**
    *   `gobuster` auf Port 80 fand nur `/index.php`.
    *   Manuelle Untersuchung der `index.php` (impliziert) offenbarte einen Cookie mit dem Base64-kodierten Wert `cG9pc29uZWRnaWZ0LnR4dA%3D%3D`.
    *   Dekodierung des Cookie-Wertes ergab den Dateinamen `poisonedgift.txt`.
    *   `curl http://192.168.2.119/poisonedgift.txt` lud einen beschädigten privaten SSH-Schlüssel herunter (vermutlich für Benutzer `sophie`).
    *   `curl http://192.168.2.119/sshnote.txt` lud eine Notiz herunter, die besagte, dass im SSH-Schlüssel drei Großbuchstaben durch Sterne (`***`) ersetzt wurden.

3.  **Initial Access (SSH-Schlüssel Reparatur & Login):**
    *   `crunch 3 3 ABCDEFGHIJKLMNOPQRSTUVWXYZ > capital.txt` generierte eine Liste aller möglichen Kombinationen von drei Großbuchstaben.
    *   Ein Skript (`stars.sh` oder ein manueller Prozess) wurde verwendet, um die Sterne im beschädigten Schlüssel (`poisonedgift.txt`) durch jede Kombination aus `capital.txt` zu ersetzen.
    *   Der mit der korrekten Kombination (vermutlich `IBM`) reparierte Schlüssel ermöglichte den SSH-Login als Benutzer `sophie` (`ssh -i [reparierter_schlüssel] sophie@192.168.2.119`).

4.  **Privilege Escalation (von `sophie` zu `root`):**
    *   `sudo -l` für `sophie` (impliziert, da die Regel bekannt wird) offenbarte, dass `sophie` den Befehl `/usr/bin/chgrp sophie /etc/shadow` ohne Passwort ausführen darf.
    *   Ausführung von `sudo /usr/bin/chgrp sophie /etc/shadow` änderte die Gruppenzugehörigkeit der `/etc/shadow`-Datei.
    *   `cat /etc/shadow` las die Datei aus und offenbarte die Passwort-Hashes, einschließlich des Root-Hashes (`$1$root$dZ6JC474uVpAeG8g0oh/7.`).
    *   Der Root-Passwort-Hash wurde mit `john --wordlist=/usr/share/wordlists/rockyou.txt [hashdatei]` geknackt. Das Passwort war `barbarita`.
    *   Login als `root` mittels `su root` und dem Passwort `barbarita`.

## Wichtige Schwachstellen und Konzepte

*   **Informationsleck durch Cookie:** Ein Base64-kodierter Dateiname in einem Cookie führte zum Fund einer sensiblen Datei.
*   **Speicherung privater SSH-Schlüssel auf Webserver:** Ein privater SSH-Schlüssel war über HTTP zugänglich.
*   **Beschädigter SSH-Schlüssel mit Hinweisen zur Reparatur:** Die Kombination aus beschädigtem Schlüssel und einer Notiz ermöglichte die Rekonstruktion des Schlüssels.
*   **Unsichere `sudo`-Konfiguration:** Die Erlaubnis, `chgrp` auf `/etc/shadow` auszuführen, ermöglichte das Auslesen der Passwort-Hashes.
*   **Schwaches Root-Passwort:** Das Root-Passwort (`barbarita`) konnte leicht mit einer Standard-Wortliste geknackt werden.
*   **Veraltetes Hashing-Format für Root:** Der Root-Passwort-Hash verwendete MD5-Crypt (`$1$`), was anfälliger für Cracking ist.

## Flags

*   **User Flag (`/home/sophie/user.txt`):** `a99ac9055a3e60a8166cdfd746511852`
*   **Root Flag (`/root/root.txt`):** `bf3b0ba0d7ebf3a1bf6f2c452510aea2`

## Tags

`HackMyVM`, `Stars`, `Easy`, `SSH`, `Cookie Manipulation`, `Base64`, `SSH Key Repair`, `sudo Exploitation`, `chgrp`, `/etc/shadow`, `Password Cracking`, `John the Ripper`, `Linux`, `Web`
