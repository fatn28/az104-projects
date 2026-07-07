# AZ-104 Projekt 5: App Service Deployment & Scaling

Ziel: Eine Web-App über eine CI/CD-Pipeline auf Azure App Service deployen, mit Deployment Slots risikofrei aktualisieren, per Autoscaling auf Last reagieren und Backup + Monitoring einrichten.

## Architektur-Überblick

- **Resource Group:** `RG-AZ104-PROJECT5`
- **App Service:** `az104-project5-webapp` — Python 3.14, Linux, zunächst Free (F1), für Slots/Autoscaling temporär auf Standard (S1) hochskaliert, danach zurück auf Free
- **Code:** Minimale Flask-App im GitHub-Repo `az104-project5-webapp` (`app.py`, `requirements.txt`)
- **CI/CD:** GitHub Actions über Deployment Center, Authentifizierung via User-assigned Managed Identity (`id-project5-deploy`)
- **Deployment Slots:** `Staging`-Slot (separater Branch `staging`) + `Production` (Branch `main`), erfolgreich geswappt
- **Autoscaling:** Custom Autoscale (Rules Based), CPU > 70 % → +1 Instanz, CPU < 30 % → -1 Instanz, Min 1 / Max 3
- **Backup:** Manuelles Backup erfolgreich durchgeführt
- **Monitoring:** Alert-Regel auf Server-Fehler/Response Time mit E-Mail-Benachrichtigung

## Umgesetzte Schritte

| # | Schritt | Screenshot |
|---|---|---|
| 1 | App Service erstellt (Python, Linux, Free-Tier) | `screenshots/app-service-overview.png` |
| 2 | Flask-App über GitHub Actions erfolgreich deployed | `screenshots/github-actions-deploy-success.png` |
| 3 | Production-Ausgabe vor dem Swap | `screenshots/webapp-output-production.png` |
| 4 | Staging-Ausgabe vor dem Swap | `screenshots/webapp-output-staging.png` |
| 5 | Deployment Slots erstellt und erfolgreich geswappt | `screenshots/deployment-slots-swap.png` |
| 6 | Custom-Autoscale-Regeln (CPU-basiert) eingerichtet | `screenshots/autoscale-rules.png` |
| 7 | Manuelles Backup erfolgreich | `screenshots/backup-success.png` |
| 8 | Alert-Regel für Monitoring erstellt | `screenshots/alert-rule-metrics.png` |

## Was ich gelernt habe

- **Deployment Slots** und **Custom Autoscale** brauchen mindestens den **Standard-Tarif** — der Free-Tier unterstützt beides nicht. Kurzzeitiges Hochskalieren zum Testen, danach zurück auf Free, hält die Kosten minimal.
- Azures GitHub-Actions-Integration im Deployment Center verlangt eine **User-assigned Managed Identity** mit **Website-Contributor**-Rolle auf der App — RBAC-Propagation kann wie bei Key-Vault-Rollen ein paar Minuten dauern.
- Ohne explizites **Startup Command** erkennt Oryx (Azures Build-System) eine einfache Flask-App nicht automatisch — die Standard-Platzhalterseite läuft dann statt des eigenen Codes.
- Zwei Deployment Slots, die auf denselben Branch zeigen, deployen **identischen Inhalt** — für echte Unterscheidung braucht jeder Slot einen eigenen Branch und eine eigene Deployment-Center-Konfiguration.
- "Automatic" Scaling (plattformverwaltet) braucht Premium v2/v3; **"Rules Based"** Scaling (klassisches, metrikbasiertes Autoscaling) funktioniert bereits auf Standard.
- Ein **409-Conflict-Fehler** bei GitHub-Actions-Deployments ist meist eine zeitliche Überschneidung mit einer anderen laufenden Deployment-Operation — ein einfacher Re-Run löst es meistens.

Ausführliche Schritt-für-Schritt-Erklärung inkl. aller Troubleshooting-Schritte: siehe [`detailed-notes.md`](./detailed-notes.md).

## Cleanup

- App Service Plan von Standard (S1) zurück auf Free (F1) skaliert ✅
- `RG-AZ104-PROJECT5` komplett gelöscht ✅
- GitHub-Repo `az104-project5-webapp` bleibt bestehen (kostenlos, eigentliches Code-Artefakt)
