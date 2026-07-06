# Projekt 2 — Detaillierte Notizen: Schritte, Begründungen, Probleme

## 1. Zwei Subnets: Web vs. Database

**Was:** Das VNet (10.0.0.0/16) wurde in zwei kleine Subnets aufgeteilt: `Web` (10.0.1.0/27) für alles, was Internetkontakt braucht, und `Database` (10.0.0.0/27) als isolierte Zone ohne Default-Outbound.

**Warum:** Netzwerksegmentierung nach Funktion (Tiering) begrenzt die Angriffsfläche — nur die Web-Schicht muss überhaupt mit dem Internet sprechen, die Datenschicht braucht das nicht. Das "Private Subnet"-Flag auf `Database` entfernt sogar den standardmäßigen Outbound-Internetzugang, den Azure-Subnets sonst implizit haben.

**Problem #1 — Azure legt automatisch ein erstes Subnet an:**
Beim Erstellen des VNets hatte Azure bereits ein Subnet namens `default` (10.0.0.0/24) vorbelegt, das nicht meinen Namens- und Größenvorgaben entsprach.

**Lösung:** Das bestehende `default`-Subnet direkt bearbeitet (umbenannt zu `Database`, Größe auf /27 reduziert, Private Subnet + Service Endpoint aktiviert) statt ein zusätzliches Subnet danebenzulegen. **Lektion:** Immer prüfen, was der Portal-Wizard schon vorausgefüllt hat, bevor man einfach draufloslegt.

## 2. NSG mit Default-Deny auf dem Web-Subnet

**Was:** Die NSG `nsg-project2-web` erlaubt nur eingehenden HTTPS-Traffic (Port 443), alles andere wird explizit geblockt.

**Warum:** Gleiches Least-Privilege-Prinzip wie in Projekt 1 — die Web-Schicht soll nur über den einen Kanal erreichbar sein, den sie tatsächlich braucht.

**Problem #2 — NSG blockiert den eigenen Verbindungstest:**
Für die spätere Verifikation des Storage-Zugriffs wurde eine Test-VM im `Web`-Subnet platziert. SSH (Port 22) wurde durch die Default-Deny-Regel sofort blockiert.

**Lösung:** Eine temporäre Inbound-Regel `Temp_Allow_SSH` mit Source `My IP` (nicht `Any`!) und höherer Priorität als die Deny-Regel hinzugefügt, nur für die Dauer des Tests. **Lektion:** Auch temporärer Zugriff sollte scope-begrenzt bleiben — "kurz mal alles öffnen" ist keine Option, selbst wenn's nur für 10 Minuten ist.

## 3. Storage Account ohne öffentlichen Zugriff

**Was:** Beim Storage Account wurde "Public network access" komplett deaktiviert, der einzige Zugriffsweg läuft über einen Private Endpoint im `Database`-Subnet.

**Warum:** Ein Storage Account mit öffentlichem Endpunkt ist über das gesamte Internet erreichbar und hängt zur Absicherung komplett von Access-Keys/SAS-Tokens ab. Ein Private Endpoint entfernt die Angriffsfläche strukturell — der Dienst ist von außen schlicht nicht ansprechbar, unabhängig davon, wie gut die Credentials geschützt sind (Defense in Depth).

**Verifiziert durch:** Ein gültiges SAS-Token wurde nur von einer VM *innerhalb* des VNets erfolgreich genutzt (per AzCopy-Upload). Von außerhalb des VNets (z. B. dem eigenen Laptop) hätte derselbe SAS-Link ins Leere gelaufen, weil kein Netzwerkpfad existiert — das SAS-Token allein reicht nicht, wenn kein Routing zum privaten Endpunkt besteht.

## 4. Infrastructure as Code: ARM-Template-Export

**Was:** Sowohl das VNet als auch der Storage Account wurden nach der manuellen Erstellung über "Export template" als ARM-Templates exportiert und als Template Specs im Portal registriert.

**Warum:** Der Export bereits bestehender Ressourcen als Template ist ein guter Einstieg in Infrastructure as Code, bevor man mit Terraform (nächster Roadmap-Punkt) komplett von Grund auf deklarativ arbeitet. Template Specs machen ein Template zentral wiederverwendbar und versionierbar, statt es nur als lokale Datei liegen zu haben.

**Problem #3 — ZIP statt JSON hochgeladen:**
Der Template-Export liefert ein ZIP-Archiv mit `template.json` und `parameters.json`. Der Import bei Template Specs verlangt aber eine einzelne JSON-Datei, keine ZIP.

**Lösung:** ZIP lokal entpackt und ausschließlich die `template.json` hochgeladen — die `parameters.json` wird beim Import nicht benötigt (Parameterwerte werden erst beim eigentlichen Deploy des Template Specs abgefragt). **Lektion:** Export-Format ≠ Import-Format — nicht einfach das Rohprodukt eines Tools an das nächste weiterreichen, ohne zu prüfen, was erwartet wird.

## 5. Verbindungsaufbau zur Test-VM

**Was:** Für den Verbindungstest zum privat abgesicherten Storage Account wurde eine kleine, temporäre VM im `Web`-Subnet angelegt und per SSH administriert.

**Problem #4 — Key lag lokal, nicht in der Cloud Shell:**
Nach dem Download des privaten SSH-Keys schlug die Verbindung über Azure Cloud Shell fehl, weil die Datei nur lokal auf dem eigenen Rechner lag.

**Lösung:** Zwei Optionen abgewogen — Datei explizit in die Cloud Shell hochladen, oder direkt das lokale Terminal verwenden (kein Upload nötig, da der Key eh schon lokal liegt). Lokales Terminal gewählt, da einfacher. **Lektion:** Cloud Shell ist eine komplett separate Umgebung mit eigenem Dateisystem — "heruntergeladen" heißt nicht "überall verfügbar".

**Problem #5 — AzCopy-Installation schlug wegen Tippfehler fehl:**
```
curl -sL https://aka.ms/downloadazcopy-v10-liux -o azcopy.tar.gz
```
Ergebnis: `gzip: stdin: not in gzip format` — die heruntergeladene Datei war kein gültiges Archiv.

**Diagnose:** Die URL enthielt einen Tippfehler (`liux` statt `linux`), wodurch curl eine Fehlerseite statt der echten Binary heruntergeladen hat.

**Lösung:** URL korrigiert (`downloadazcopy-v10-linux`), danach lief `tar -xf` und `sudo cp azcopy /usr/bin/` fehlerfrei durch. **Lektion:** Bei "Datei ist kaputt"-Fehlern zuerst die Downloadquelle/URL prüfen, bevor man tiefer im Tool selbst sucht.

## Zusammenfassung der Kernprobleme

| Problem | Ursache | Fix |
|---|---|---|
| Vorbelegtes `default`-Subnet passte nicht | Azure-Wizard legt automatisch ein erstes Subnet an | Bestehendes Subnet bearbeitet statt neues angelegt |
| NSG blockiert eigenen SSH-Verbindungstest | Default-Deny-Regel auf dem Web-Subnet | Temporäre, scope-begrenzte Allow-Regel (`My IP`) hinzugefügt, später entfernt |
| SAS-Token allein reichte nicht für Zugriffstest | Public Network Access am Storage Account deaktiviert | Zugriff über VM *innerhalb* des VNets getestet, bewusst als Verifikation der Isolation genutzt |
| ZIP-Datei ließ sich nicht als Template importieren | Template Specs erwarten reine `template.json`, kein ZIP | ZIP entpackt, nur `template.json` hochgeladen |
| Key nicht in Cloud Shell auffindbar | Datei lag nur lokal, nicht in der Cloud-Shell-Umgebung | Lokales Terminal statt Cloud Shell verwendet |
| AzCopy-Download schlug fehl | Tippfehler in der Download-URL | URL korrigiert, Installation erfolgreich wiederholt |
