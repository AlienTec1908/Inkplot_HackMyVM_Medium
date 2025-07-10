# Inkplot - HackMyVM

**Schwierigkeitsgrad:** Medium 🟡

---

## ℹ️ Maschineninformationen

*   **Plattform:** HackMyVM
*   **VM Link:** [https://hackmyvm.eu/machines/machine.php?vm=Inkplot](https://hackmyvm.eu/machines/machine.php?vm=Inkplot)
*   **Autor:** DarkSpirit

![Inkplot Machine Icon](Inkplot.png)

---

## 🏁 Übersicht

Dieser Bericht dokumentiert den Penetrationstest der virtuellen Maschine "Inkplot" von HackMyVM. Das Ziel war die Erlangung von Systemzugriff und die Ausweitung der Berechtigungen bis auf Root-Ebene. Der Test umfasste die Aufklärung offener Dienste, die Ausnutzung eines Informationslecks über einen WebSocket-Dienst, Hash-Cracking, den Umgang mit einem unsicheren Verschlüsselungsskript und die Ausnutzung von falsch konfigurierten Sudo-Berechtigungen und Dateiberechtigungen für eine administrative Gruppe.

---

## 📖 Zusammenfassung des Walkthroughs

Der Pentest gliederte sich in folgende Hauptphasen:

### 🔎 Reconnaissance

*   Netzwerkscan (`arp-scan`) zur Identifizierung der Ziel-IP (192.168.2.43).
*   Hinzufügen des Hostnamens `inplot.hmv` zur lokalen `/etc/hosts`.
*   Umfassender Portscan (`nmap`), der Port 22 (SSH - OpenSSH 9.2p1) und Port 3000 (WebSocket - Ogar agar.io server) als offen identifizierte.

### 🌐 Web Enumeration

*   Test des Dienstes auf Port 3000 mit `nikto` (wenige Funde) und `curl` (antwortet mit `426 Upgrade Required`), was auf ein WebSocket-Protokoll hindeutete.
*   Verbindung zum WebSocket-Dienst mit `wscat` und Abfangen einer kritischen Konversation, die die Benutzernamen `Leila`, `Bob`, `Alice` und einen partiellen MD5-Hash (`d51540...`) sowie dessen Relevanz für den Zugriff auf "Leila" preisgab.
*   Erstellung und Ausführung eines benutzerdefinierten Python-Skripts (`hash_crack.py`) zum Knacken des MD5-Hashes aus einer Wortliste (`rockyou.txt`), unter Berücksichtigung eines angehängten Zeilenumbruchs.
*   Das Skript lieferte zwei Passwortkandidaten, darunter `palmira`.

### 💻 Initialer Zugriff

*   Erfolgreiche SSH-Anmeldung am Zielsystem als Benutzer `leila` unter Verwendung des geknackten Passworts `palmira`. Dies verschaffte eine initiale Shell als `leila`.

### 📈 Privilege Escalation

*   Prüfung der `sudo`-Berechtigungen für `leila` (`sudo -l`). Gefunden: `(pauline : pauline) NOPASSWD: /usr/bin/python3 /home/pauline/cipher.py*`.
*   Identifizierung des Benutzers `pauline` (`ls ..`) und Untersuchung des Home-Verzeichnisses von `pauline` (`ls -la /home/pauline`), das u.a. `cipher.py`, `keys.json` und `user.txt` enthielt (für `leila` lesbar).
*   Analyse des Python-Skripts `cipher.py`: Liest Schlüssel aus `keys.json`, verwendet ARC4-Verschlüsselung und Base64-Kodierung, hat eine interaktive Entschlüsselungsfunktion.
*   Testläufe mit `cipher.py` (als `pauline` via `sudo`) und Analyse der Ausgaben zeigten die Funktionsweise und dass die resultierenden Dateien Base64-kodiert sind.
*   Entschlüsselung der User-Flag: Der Inhalt von `/home/pauline/user.txt.enc` wurde als Base64-kodierter, verschlüsselter Inhalt von `user.txt` identifiziert. `cat /home/pauline/user.txt.enc | base64 -d` ergab die User-Flag.
*   Extraktion des verschlüsselten privaten SSH-Schlüssels von `pauline` (`/home/pauline/.ssh/id_rsa`) durch Verschlüsselung mit `cipher.py` und Base64-Dekodierung des Outputs.
*   Finden des Entschlüsselungsschlüssels (`aLLtBh0BVCFSvfZ203sM`) in `/home/pauline/keys.json`.
*   Offline-Entschlüsselung des privaten SSH-Schlüssels von `pauline` mit dem gefundenen Schlüssel und dem `cipher.py` Skript.
*   Erfolgreiche SSH-Anmeldung am Zielsystem als Benutzer `pauline` unter Verwendung des entschlüsselten privaten SSH-Schlüssels.
*   Prüfung der `sudo`-Berechtigungen für `pauline` (`sudo -l`). Keine direkte NOPASSWD-Regel für root gefunden, aber Mitglied der Gruppe `admin` (`id`).
*   Suche nach Dateien/Verzeichnissen der Gruppe `admin` (`find / -group admin`) identifizierte `/usr/lib/systemd/system-sleep`.
*   Prüfung der Berechtigungen von `/usr/lib/systemd/system-sleep` (`ls -la`) ergab Schreibberechtigungen für die Gruppe `admin` (`drwxrwx---`).
*   Erstellung eines Bash Reverse-Shell-Skripts (`root.sh`) in `/tmp` und Kopieren nach `/usr/lib/systemd/system-sleep`.
*   Einrichtung eines Netcat-Listeners auf der Angreifer-Maschine.
*   Warten auf ein System-Suspend-Ereignis, das das Skript als Root auslöste.
*   Der Netcat-Listener empfing eine Root-Shell vom Zielsystem.

### 🚩 Flags

*   **User Flag:** Gefunden in `/home/pauline/user.txt`
    ` `a2c145eb8279c2f920de6871bef794fa` `
*   **Root Flag:** Gefunden in `/root/root.txt`
    ` `4d9089c262be4a03e3ebfdaff0a8f7c6` `

---

## 🧠 Wichtige Erkenntnisse

*   **Informationslecks:** Sensitive Daten (wie Benutzerkonversationen, partielle Hashes) können über ungesicherte Dienste wie WebSockets zugänglich sein.
*   **Hash Cracking:** Das Verständnis spezifischer Hashing-Methoden (z.B. mit angehängtem Zeilenumbruch) ist entscheidend. MD5 ist anfällig für Wörterbuchangriffe.
*   **Unsichere Skripte & Schlüsselverwaltung:** Skripte, die Passwörter oder Schlüssel im Klartext (z.B. in JSON-Dateien) speichern, stellen ein hohes Risiko dar, insbesondere wenn diese Skripte mit erhöhten Berechtigungen ausführbar sind oder von anderen Benutzern gelesen werden können.
*   **Fehlkonfigurierte Sudo-Berechtigungen:** `NOPASSWD`-Einträge für Binärdateien oder Skripte, die Dateisystemoperationen durchführen können (auch indirekt wie `cipher.py`), sind klassische PE-Vektoren. Wildcards (`*`) in `sudoers` erhöhen das Risiko.
*   **Dateiberechtigungen & Gruppenmitgliedschaften:** Schreibrechte für administrative Gruppen in kritischen Systemverzeichnissen (wie `systemd`-Verzeichnisse) ermöglichen das Platzieren von Root-Skripten und führen zur vollständigen Systemkompromittierung.
*   **Systemd Execution:** Skripte in bestimmten `systemd`-Verzeichnissen werden mit Root-Rechten bei spezifischen Systemereignissen (Suspend/Resume) ausgeführt.

---

## 📄 Vollständiger Bericht

Eine detaillierte Schritt-für-Schritt-Anleitung, inklusive Befehlsausgaben, Analyse, Bewertung und Empfehlungen für jeden Schritt, finden Sie im vollständigen HTML-Bericht:

[**➡️ Vollständigen Pentest-Bericht hier ansehen**](https://alientec1908.github.io/Inkplot_HackMyVM_Medium/)

---

*Berichtsdatum: 14. Juni 2025*
*Pentest durchgeführt von Ben Chehade*
