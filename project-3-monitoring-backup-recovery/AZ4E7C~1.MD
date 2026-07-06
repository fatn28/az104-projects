# Projekt 3 — Detaillierte Notizen: Schritte, Begründungen, Probleme

Dieses Projekt hatte deutlich mehr echte Stolpersteine als die vorherigen — vor allem, weil sich Azures Monitoring- und Site-Recovery-Onboarding-Flows in den letzten Jahren stark verändert haben und viele Tutorials/Projektbeschreibungen noch den alten ("Classic") Weg beschreiben, der in der aktuellen Portal-Version teilweise nicht mehr funktioniert.

## 1. VM Insights & Alerts

**Was:** Azure Monitor für die VM aktiviert, CPU-Alert bei > 80 % mit E-Mail-Benachrichtigung.

**Warum:** Ohne Monitoring merkt man Performance-Probleme oder Ausfälle oft erst, wenn Nutzer sich beschweren. Ein einfacher CPU-Schwellwert-Alert ist der Basis-Baustein für proaktives Infrastruktur-Monitoring.

**Problem #1 — "Select identity" verlangt zwingend eine User-assigned Identity:**
Beim Aktivieren von "OpenTelemetry metrics" (dem neuen, empfohlenen Pfad) verlangte der Wizard eine Identity-Auswahl. Trotz aktivierter System-assigned Identity an der VM selbst zeigte das Panel ausschließlich eine Auswahl für **User-assigned** Identities — leer, da keine existierte.

**Lösung (Workaround für dieses Projekt):** Auf **"[Classic] Log-based metrics"** umgestellt, um die Identity-Anforderung zu umgehen. **Lektion:** Nicht jede Checkbox-Kombination in modernisierten Azure-Wizards ist gleich gut unterstützt — manchmal ist der vermeintlich "alte" Pfad der pragmatischere.

**Problem #2 — Classic Log-based metrics deployt keinen funktionierenden Agent:**
Nach Aktivierung blieb die `Perf`-Tabelle im Log Analytics Workspace dauerhaft leer, auch nach langem Warten.

**Diagnose:** Ein Blick auf die VM-Extensions zeigte: **kein** Monitoring-Agent war überhaupt installiert. Der klassische Log-Analytics-Agent (OMS-Agent) ist mittlerweile von Microsoft abgekündigt — die "Classic"-Checkbox im Wizard hat schlicht keine funktionierende Wirkung mehr.

**Lösung:** Statt über den VM-Insights-Wizard direkt eine **Data Collection Rule (DCR)** mit Performance-Counter-Datenquelle erstellt und die VM als Ressource hinzugefügt. Beim Verknüpfen der DCR installiert Azure automatisch den **Azure Monitor Agent** im Hintergrund — kein manueller Extension-Such-Schritt nötig (der offizielle Agent taucht in der Extensions-Galerie unter "+ Add" auch gar nicht auf). **Lektion:** Bei Monitoring-Problemen immer zuerst die VM-Extensions prüfen, ob überhaupt ein Agent läuft, bevor man an der Query selbst zweifelt.

**Problem #3 — Neu installierte Agent-Extension blieb auf "Unavailable" hängen:**
Nach dem Erstellen der DCR zeigte die `AzureMonitorLinuxAgent`-Extension den Status "Unavailable".

**Diagnose:** Auf derselben VM hingen noch zwei Site-Recovery-Extensions aus einem fehlgeschlagenen Site-Recovery-Versuch (siehe Punkt 3) — eine davon im Zustand "Provisioning failed". Eine hängende Extension kann die Verarbeitung neuer Extension-Installationen auf derselben VM blockieren.

**Lösung:** Beide Site-Recovery-Extensions von dieser VM entfernt (sie gehörten ohnehin nicht dorthin) — danach ging die Azure-Monitor-Agent-Installation sofort auf "Provisioning succeeded", und die `Perf`-Tabelle füllte sich innerhalb weniger Minuten mit echten CPU-Daten. **Lektion:** Extensions auf einer VM sind nicht komplett unabhängig voneinander — ein hängender Zustand kann andere Installationen blockieren.

## 2. Azure Backup

**Was:** Recovery Services Vault mit täglicher Backup-Policy (7 Tage Retention) für die VM eingerichtet.

**Warum:** Ein Backup schützt gegen Datenverlust durch versehentliches Löschen, Korruption oder Ransomware — unabhängig von Hochverfügbarkeits-Mechanismen wie Site Recovery, die primär gegen Regionsausfälle schützen, nicht gegen logische Fehler.

Dieser Schritt verlief ohne nennenswerte Probleme.

## 3. Azure Site Recovery

**Was:** Azure-zu-Azure-Replikation von einer VM in eine zweite Region eingerichtet, um Failover-Fähigkeit zu demonstrieren.

**Warum:** Site Recovery schützt gegen den Ausfall einer ganzen Azure-Region — ein Szenario, das reines Backup nicht abdeckt (Backup schützt Daten, nicht Verfügbarkeit).

**Problem #4 — Wiederholte "Conflict"-Fehler (Code 539) beim Deployment:**
Die Replikationskonfiguration schlug mehrfach mit einem generischen ARM-Deployment-Konflikt fehl, auch nach "Redeploy" und dem Entfernen der fehlgeschlagenen Extension.

**Diagnose:** Recherche in der offiziellen Microsoft-Dokumentation ergab: **Ubuntu 24.04 LTS wird von Azure Site Recovery nur in der "Modernized experience" unterstützt, nicht im klassischen Flow** über die VM-Kachel "Disaster Recovery" — genau der Weg, den wir genutzt hatten.

**Lösung:** Eine zweite, kleine Test-VM mit **Ubuntu 20.04 LTS** angelegt (uneingeschränkt kompatibel) und die Disaster-Recovery-Konfiguration dort erfolgreich durchgeführt, statt die produktiv genutzte 24.04-VM zu riskieren. **Lektion:** Nicht jede Kombination aus Betriebssystem-Version und Azure-Feature ist vollständig unterstützt — bei wiederholten, nicht eindeutigen Fehlern lohnt sich ein Blick in die offizielle Support-Matrix, bevor man weiter an der Konfiguration herumprobiert.

**Problem #5 — Fehler beim Start der Replikation wegen Availability-Zone-Abhängigkeiten:**
Mit "Availability zone" als Ziel-Verfügbarkeitstyp schlug die Ressourcenerstellung fehl ("Failed to create one or more Azure resources"), verursacht durch fehlende Zuordnung von Proximity Placement Group und Capacity Reservation zur gewählten Zone.

**Lösung:** Ziel-Verfügbarkeitstyp auf **"Single instance"** umgestellt — das entfernt die Anforderungen für Proximity Placement Group und Capacity Reservation komplett, da diese nur für Availability-Zone- bzw. -Set-Konfigurationen gelten. Für eine reine Lern-/Testumgebung völlig ausreichend.

## 4. Log-Analyse per KQL

**Was:** Die `Perf`-Tabelle im Log Analytics Workspace mit einer KQL-Query nach durchschnittlicher CPU-Auslastung in 5-Minuten-Intervallen abgefragt.

**Problem #6 — Query-Editor verschluckte beim Einfügen die erste Zeile:**
Beim Copy-Paste der mehrzeiligen KQL-Query fehlten wiederholt die Zeile `Perf` (die Datenquelle) sowie die Pipe-Zeichen `|` vor den Folgezeilen — die Query war dadurch syntaktisch ungültig.

**Lösung:** Query manuell Zeile für Zeile eingetippt statt komplett eingefügt. Zusätzlich musste zuerst vom **Simple mode** in den **KQL mode** gewechselt werden (Simple mode ist ein visueller Query-Builder ohne Freitext-Editor). **Lektion:** Bei Copy-Paste-Problemen in Web-basierten Code-Editoren lieber kritische erste Zeilen manuell prüfen/eintippen.

## Zusammenfassung der Kernprobleme

| Problem | Ursache | Fix |
|---|---|---|
| Identity-Auswahl bei OpenTelemetry-Metriken erzwang User-assigned Identity | System-assigned wird in diesem Wizard-Schritt nicht akzeptiert | Auf Classic Log-based metrics umgestellt |
| Classic Log-based metrics lieferte keine Daten | Klassischer Log-Analytics-Agent ist abgekündigt, wird nicht mehr deployed | Data Collection Rule mit Performance Countern manuell erstellt (installiert AMA automatisch) |
| Neue Agent-Extension blieb auf "Unavailable" | Hängende fehlgeschlagene Site-Recovery-Extensions blockierten die VM | Fehlerhafte Extensions entfernt |
| Site Recovery schlug wiederholt mit Conflict-Fehler fehl | Ubuntu 24.04 LTS wird von ASR nur in der Modernized Experience unterstützt | Separate Test-VM mit Ubuntu 20.04 LTS verwendet |
| Ressourcenerstellung bei Replikationsstart fehlgeschlagen | Availability-Zone-Ziel erforderte nicht auflösbare PPG/Capacity-Reservation-Zuordnung | Auf "Single instance" als Ziel-Verfügbarkeitstyp umgestellt |
| KQL-Query war syntaktisch ungültig nach Copy-Paste | Editor verschluckte erste Zeile und Pipe-Zeichen beim Einfügen | Query manuell zeilenweise eingetippt, vorher auf KQL mode umgeschaltet |
