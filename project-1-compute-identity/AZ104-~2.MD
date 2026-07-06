# Projekt 1 — Detaillierte Notizen: Schritte, Begründungen, Probleme

## 1. Resource Group pro Projekt

**Was:** Eigene Resource Group `RG-AZ104-PROJECT1` statt einer gemeinsamen RG für alle Projekte.

**Warum:** Eine RG pro Projekt macht Aufräumen trivial — ein einziger "Delete Resource Group"-Klick entfernt garantiert alle zugehörigen Ressourcen, ohne dass man Gefahr läuft, etwas übersehen aus einem anderen Projekt mit zu löschen (oder umgekehrt: etwas stehen zu lassen und weiter Kosten zu verursachen).

## 2. VM mit SSH-Key statt Passwort

**Was:** Bei der VM-Erstellung SSH-Public-Key-Authentifizierung gewählt, Azure hat automatisch ein Schlüsselpaar generiert.

**Warum:** Passwort-Authentifizierung für SSH ist anfällig für Brute-Force-Angriffe und wird in produktiven/sicherheitsbewussten Umgebungen praktisch nie verwendet. Key-basierte Auth ist Industriestandard.

**Problem #1 — Quota Exceeded:**
Beim ersten Deployment-Versuch in der Region `Germany West Central` kam der Fehler:
```
Operation could not be completed as it results in exceeding approved
standardBasv2Family Cores quota. Current Limit: 0
```
**Ursache:** Frische Pay-as-you-go-Subscriptions bekommen aus Betrugsschutz-Gründen für manche VM-Familien in manchen Regionen initial ein Kontingent von 0 Cores.

**Lösung:** Statt eine Quota-Erhöhung zu beantragen (dauert, auch wenn oft automatisch genehmigt), einfach eine andere Region gewählt (Poland Central) — dort war bereits ausreichend Kontingent frei. **Lektion:** Quota ist regionsspezifisch, ein Regionswechsel ist oft der schnellere Fix als ein Support-Ticket.

## 3. Netzwerksicherheit: NSG mit minimaler Angriffsfläche

**Was:** Die Standard-NSG lässt nur eingehenden SSH-Traffic (Port 22) zu, alles andere läuft in die implizite `DenyAllInBound`-Regel.

**Warum:** Least-Privilege-Prinzip — die VM soll administrierbar sein (SSH), aber sonst keine unnötig offenen Ports haben, die eine Angriffsfläche bieten (z. B. HTTP/HTTPS, obwohl kein Webserver läuft).

**Verifiziert durch:** Test im Browser mit der öffentlichen IP der VM — Zugriff über HTTP/HTTPS schlägt erwartungsgemäß fehl (Timeout), da nur Port 22 erlaubt ist.

## 4. Schlüsselverwaltung über Key Vault statt lokaler Datei

**Was:** Der private SSH-Key wurde nicht dauerhaft lokal aufbewahrt, sondern in einen Azure Key Vault importiert.

**Warum:** Zentrale, auditierbare Verwaltung sensibler Schlüssel/Secrets ist Kernbestandteil von Cloud-Security-Praxis. Ein Key Vault bietet Zugriffskontrolle (RBAC), Audit-Logs und Verschlüsselung at rest — eine lose `.pem`-Datei auf der Festplatte bietet nichts davon.

**Problem #2 — Key nicht in Cloud Shell auffindbar:**
Nach dem (vermeintlichen) Upload des privaten Keys in die Azure Cloud Shell schlug `chmod 600 vm-project1-webkey` mit `No such file or directory` fehl.

**Diagnose:** `ls -la` im Home-Verzeichnis zeigte, dass die Datei gar nicht vorhanden war — der Upload war nie durchgelaufen.

**Lösung:** Upload über das Cloud-Shell-eigene Upload-Icon erneut ausgeführt, und der tatsächliche lokale Dateiname geprüft — Azure hatte die Datei beim Download mit Unterstrich und `.pem`-Endung benannt (`vm-project1-web_key.pem`), nicht wie ursprünglich angenommen (`vm-project1-webkey`). **Lektion:** Immer den tatsächlichen Dateinamen im Downloads-Ordner prüfen, statt ihn zu erraten.

## 5. RBAC über Entra-Gruppe statt Einzelnutzer

**Was:** Statt Rollen direkt an den eigenen Benutzer zu vergeben, wurde eine Security-Gruppe (`az104-project-admins`) erstellt und die Rollen der Gruppe zugewiesen.

**Warum:** Skalierbarkeit — kommen später weitere Personen dazu (Teammitglieder, andere Admins), reicht es, sie der Gruppe hinzuzufügen, statt für jede Person einzeln Rollenzuweisungen zu pflegen. Das ist Standard-Praxis in echten Unternehmensumgebungen.

## 6. Azure Policy: Governance durchsetzen

**Was:** Eine Initiative mit drei Policies wurde erstellt:
- Tag-Vererbung von der Resource Group
- Tag-Vererbung von der Subscription
- Beschränkung auf erlaubte Ressourcentypen (`Microsoft.Compute/virtualMachines`)

**Warum:** In Unternehmensumgebungen ist konsistente Tagging-Praxis (z. B. `Environment`, `CostCenter`) essenziell für Kostenzuordnung und Ressourcenorganisation. Die Beschränkung auf erlaubte Ressourcentypen verhindert, dass in dieser Resource Group versehentlich unpassende/teure Ressourcentypen entstehen.

**Problem #3 — Falsche Policy-Auswahl:**
Beim Hinzufügen der Tag-Policies wurden zunächst zwei Varianten derselben Quelle ausgewählt ("Inherit a tag from the resource group" und "...if missing") statt einer RG- und einer Subscription-Variante.

**Lösung:** Gezielt nach "subscription" gesucht, um die zweite benötigte Policy zu finden.

**Problem #4 — Falscher Ressourcentyp:**
Bei "Allowed resource types" wurde zunächst `privateClouds/clusters/virtualMachines` (unter `Microsoft.AVS` — Azure VMware Solution) ausgewählt statt des schlichten `virtualMachines` unter `Microsoft.Compute`.

**Lösung:** Genau auf den Provider-Namespace in der Liste geachtet (`Microsoft.Compute` statt `Microsoft.AVS`), korrekten Eintrag gewählt.

**Problem #5 — Initiative wurde nie zugewiesen:**
Nach dem Erstellen der Initiative-Definition erschien sie weder unter "Assignments" noch auf der VM-Policies-Seite.

**Diagnose:** Definition und Zuweisung (Assignment) sind in Azure Policy zwei getrennte Schritte — eine erstellte Definition wird nicht automatisch angewendet.

**Lösung:** Über Policy → Definitions → die Initiative geöffnet → explizit "Assign" geklickt, Scope und Parameter (`tagName = Environment`) gesetzt. Danach zusätzlich einmal ab-/angemeldet (Token-Refresh) und die Compliance-Auswertung manuell angestoßen:
```
az policy state trigger-scan --resource-group RG-AZ104-PROJECT1
```
Ergebnis: Die Initiative erschien danach korrekt mit **100 % Compliant** auf der VM (der "Inherit a tag"-Policy-Effekt "Modify" hat den fehlenden Tag automatisch nachgetragen).

## 7. Kostenüberwachung

**Was:** Budget mit 30 $ monatlichem Limit und Alert-Schwellen bei 50/80/100 % — noch vor dem ersten Ressourcen-Deployment eingerichtet.

**Warum:** Bei Pay-as-you-go-Konten mit echter Kreditkarte ist ein Sicherheitsnetz gegen unerwartete Kosten Pflicht, nicht optional — besonders beim Experimentieren mit unbekannten Services.

**Beobachtung:** Cost Analysis zeigte direkt nach dem Deployment 0 $, und Azure Advisor zeigte keine Kostenempfehlungen. Beides ist normal: Abrechnungsdaten haben eine Latenz von mehreren Stunden bis zu einem Tag, und Advisor benötigt eine Nutzungshistorie von mehreren Tagen, bevor er Empfehlungen ausspricht — kein Fehler, einfach zu früh im Prozess.

## Zusammenfassung der Kernprobleme

| Problem | Ursache | Fix |
|---|---|---|
| Quota Exceeded bei VM-Deployment | Neue Subscription, 0 Cores Default-Quota in der Region | Andere Region gewählt |
| SSH-Key nicht in Cloud Shell auffindbar | Upload nie durchgelaufen, falscher angenommener Dateiname | Datei erneut hochgeladen, echten Namen geprüft |
| Falsche Tag-Policies ausgewählt | Zwei Varianten derselben Quelle statt RG+Subscription | Gezielt nach "subscription" gesucht |
| Falscher Ressourcentyp bei Policy | Microsoft.AVS statt Microsoft.Compute im Dropdown verwechselt | Korrekten Provider-Namespace identifiziert |
| Policy Initiative nicht sichtbar/wirksam | Definition erstellt, aber nie zugewiesen | Explizit über "Assign" zugewiesen + manueller Compliance-Scan |
