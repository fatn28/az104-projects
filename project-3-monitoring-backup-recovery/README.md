# AZ-104 Projekt 3: Monitoring, Backup & Recovery

Ziel: Eine VM überwachen (Metriken + Alerts), regelmäßige Backups einrichten, Disaster Recovery in eine andere Region konfigurieren und Log-Daten per KQL auswerten.

## Architektur-Überblick

- **Resource Group:** `RG-AZ104-PROJECT3`
- **VM (Monitoring/Backup):** `vmaz104project3` — Ubuntu 24.04 LTS
- **VM (Disaster Recovery Demo):** `vmaz104project3.1` — Ubuntu 20.04 LTS (separate VM, siehe Troubleshooting)
- **Monitoring:** Azure Monitor Agent + Data Collection Rule (Performance Counters → Log Analytics Workspace), Alert-Regel bei CPU > 80 %
- **Backup:** Recovery Services Vault, tägliche Backup-Policy, 7 Tage Retention
- **Disaster Recovery:** Azure-zu-Azure-Replikation (Poland Central → Sweden Central), Single-Instance-Verfügbarkeit
- **Log-Analyse:** KQL-Query gegen die `Perf`-Tabelle im Log Analytics Workspace

## Umgesetzte Schritte

| # | Schritt | Screenshot |
|---|---|---|
| 1 | VM Insights aktiviert, CPU-Alert (>80 %) mit E-Mail-Benachrichtigung eingerichtet | `screenshots/monitor-alert-rule.png` |
| 2 | Recovery Services Vault + Backup-Policy erstellt, VM gesichert | `screenshots/backup-configuration.png` |
| 3 | Azure-zu-Azure-Replikation für Test-VM (Ubuntu 20.04) gestartet | `screenshots/site-recovery-replication.png` |
| 4 | Data Collection Rule mit Performance Countern erstellt, Azure Monitor Agent installiert | `screenshots/dcr-overview.png` |
| 5 | KQL-Query gegen `Perf`-Tabelle erfolgreich mit echten CPU-Daten | `screenshots/log-query-results.png` |
| 6 | Vault-Verschlüsselung und Security Level bestätigt | `screenshots/vault-properties-security.png` |

## Was ich gelernt habe

- Die neue "Configure monitor"-Oberfläche für VM Insights trennt zwischen **OpenTelemetry-Metriken** (moderner Prometheus-Pfad, braucht eine User-assigned Identity) und **Classic Log-based metrics** — letztere hat in der Praxis keinen funktionierenden Agent mehr deployed, da der klassische Log-Analytics-Agent abgekündigt ist.
- **Azure Monitor Agent** wird nicht manuell aus der Extensions-Galerie installiert, sondern automatisch, sobald eine **Data Collection Rule** mit der VM verknüpft wird.
- **Azure Site Recovery unterstützt Ubuntu 24.04 LTS nur in der "Modernized experience"**, nicht im klassischen Flow über die VM-Kachel — das führte zu wiederholten Extension-Konflikten.
- Hängende/fehlgeschlagene VM-Extensions können die Installation neuer Extensions blockieren — eine bereinigte Extensions-Liste ist Voraussetzung für neue Deployments.
- Bei Azure-Site-Recovery-Zielkonfigurationen vereinfacht **"Single instance"** statt "Availability zone" die Einrichtung erheblich (keine Proximity-Placement-Group- oder Capacity-Reservation-Abhängigkeiten).

Ausführliche Schritt-für-Schritt-Erklärung inkl. aller Troubleshooting-Schritte: siehe [`detailed-notes.md`](./detailed-notes.md).

## Cleanup

- Test-VM `vmaz104project3.1` und deren Replikation nach Abschluss deaktivieren/löschen
- Nach Abschluss der Dokumentation: `RG-AZ104-PROJECT3` sowie die automatisch angelegten Site-Recovery-Support-Ressourcengruppen komplett löschen
