# Projekt 5 — Detaillierte Notizen: Schritte, Begründungen, Probleme

## 1. App Service erstellen

**Was:** Ein App Service mit Python-3.14-Runtime auf Linux erstellt, zunächst im Free-Tier (F1).

**Warum:** Free-Tier ist ideal für Entwicklung/Test, kostet nichts, hat aber Einschränkungen (kein Custom-Domain, keine Deployment Slots, kein Autoscaling) — genau diese Einschränkungen wurden im Projektverlauf zum eigentlichen Lernpunkt.

Dieser Schritt verlief ohne Probleme.

## 2. CI/CD mit GitHub Actions

**Was:** Eine minimale Flask-App (`app.py`, `requirements.txt`) in einem separaten GitHub-Repo (`az104-project5-webapp`) angelegt und über den Deployment Center mit GitHub Actions verbunden.

**Warum:** Eine automatisierte Pipeline (statt manuellem Upload/FTP) ist Standard-Praxis — jeder Push auf den verbundenen Branch löst automatisch Build + Deploy aus.

**Problem #1 — Identity-Anforderung, gleiches Muster wie in vorherigen Projekten:**
Der Deployment-Center-Wizard verlangte eine **User-assigned Managed Identity** für die GitHub-Actions-Authentifizierung (System-assigned wurde nicht angeboten).

**Lösung:** Eine neue User-assigned Managed Identity (`id-project5-deploy`) erstellt und ausgewählt.

**Problem #2 — Identity hatte zunächst keine Schreibrechte:**
Fehlermeldung: "This identity does not have write permissions on this app."

**Lösung:** Über Access Control (IAM) der App Service-Ressource die Rolle **Website Contributor** der Managed Identity zugewiesen. Wie bei der Key-Vault-Rollenzuweisung in Projekt 1 half ein Re-Login (Token-Refresh), bis die RBAC-Änderung propagiert war.

**Problem #3 — App zeigte nur die Azure-Standard-Platzhalterseite:**
Nach erfolgreichem GitHub-Actions-Deploy zeigte die App-URL weiterhin "Hey, Python developers! ... Time to take the next step and deploy your code." statt der eigenen Ausgabe.

**Diagnose:** Log Stream zeigte: `App Command Line not configured, will attempt auto-detect` und `No framework detected; using default app from /opt/defaultsite`. Oryx (Azures Build-/Erkennungssystem) konnte die Flask-App nicht automatisch erkennen und startete stattdessen die Standard-Demo-Seite.

**Lösung:** Unter **Configuration → Stack settings → Startup Command** explizit gesetzt:
```
gunicorn --bind=0.0.0.0 --timeout 600 app:app
```
Nach einem Neustart zeigte der Log Stream danach korrekt: `Site's appCommandLine: gunicorn --bind=0.0.0.0 --timeout 600 app:app` und `Found build manifest file... Extracting...`. **Lektion:** Bei Python-Web-Apps auf App Service lieber von Anfang an ein explizites Startup Command setzen, statt auf Auto-Detection zu vertrauen.

## 3. Deployment Slots

**Was:** Ein `Staging`-Slot erstellt, um eine neue App-Version isoliert zu testen, bevor sie live geht.

**Problem #4 — Free-Tier unterstützt keine Deployment Slots:**
Die Deployment-Slots-Seite zeigte direkt: "Upgrade to a standard or premium plan to add slots."

**Lösung:** App Service Plan kurzzeitig auf **Standard (S1)** hochskaliert (bewusste, zeitlich begrenzte Kostenentscheidung, danach wieder zurückskaliert).

**Problem #5 — Beide Slots zeigten identischen Inhalt:**
Nach dem ersten Staging-Deploy zeigten sowohl Production als auch Staging denselben (neuen) Text.

**Diagnose:** Beide Slots waren im Deployment Center auf denselben Branch (`main`) desselben Repos konfiguriert — jeder Push auf `main` deployte also in beide Slots gleichzeitig.

**Lösung:** Zwei getrennte Branches angelegt (`main` mit Originaltext, `staging` mit abweichendem Text), und die Deployment-Center-Konfiguration des Staging-Slots gezielt auf den `staging`-Branch umgestellt (über "Disconnect" + Neuverbindung, da der Branch nicht direkt im bestehenden Setup änderbar war).

**Problem #6 — 409-Conflict-Fehler beim Deploy:**
Ein GitHub-Actions-Lauf schlug mit `Failed to deploy web package using OneDeploy to App Service. Conflict (CODE: 409)` fehl.

**Diagnose:** Eine andere Deployment-Operation lief zeitgleich (durch das Neuverbinden des Deployment Centers ausgelöst).

**Lösung:** Workflow über "Re-run failed jobs" erneut gestartet — lief danach fehlerfrei durch, da keine Überschneidung mehr vorlag.

**Ergebnis:** Nach erfolgreicher Trennung der Slots per Branch wurde der **Swap** (Staging → Production) erfolgreich durchgeführt — die Production-URL zeigte danach den vorher nur in Staging sichtbaren Inhalt, ohne Downtime.

## 4. Autoscaling

**Was:** Custom-Autoscale-Regeln eingerichtet: CPU > 70 % → Instanz +1, CPU < 30 % → Instanz -1, mit Min/Max-Grenzen.

**Problem #7 — "Automatic" Scaling verlangte Premium-Tarif:**
Die neue, vereinheitlichte Scale-out-Oberfläche bot direkt "Automatic" (plattformverwaltetes Scaling) an, das aber explizit einen Premium-v2/v3-Tarif voraussetzt.

**Lösung:** Stattdessen **"Rules Based"** gewählt — das klassische, metrikbasierte Autoscaling, das bereits auf Standard (S1) verfügbar ist und funktional identisch zu dem ist, was das Projekt eigentlich verlangt.

## 5. Backup & Monitoring

**Was:** Ein manuelles Backup über die Backups-Funktion des App Service ausgeführt, sowie eine Alert-Regel (Server-Fehler/Response Time) mit E-Mail-Benachrichtigung eingerichtet.

Diese Schritte verliefen ohne nennenswerte Probleme.

## 6. Kosten im Blick behalten

**Was:** Nach dem Testen von Slots und Autoscaling wurde der App Service Plan sofort wieder auf **Free (F1)** zurückskaliert.

**Warum:** Standard (S1) kostet laufend (~0,095 $/Stunde), während Free nichts kostet. Bewusstes, zeitlich begrenztes Hochskalieren nur für den Test-/Demo-Zeitraum ist die kosteneffizienteste Art, Premium-Features trotzdem einmal live zu sehen.

## Zusammenfassung der Kernprobleme

| Problem | Ursache | Fix |
|---|---|---|
| Identity-Auswahl im Deployment Center | Nur User-assigned Identity akzeptiert | Neue Managed Identity erstellt |
| Identity ohne Schreibrechte | Fehlende RBAC-Rolle | Website Contributor zugewiesen, Re-Login für Propagation |
| Nur Standard-Platzhalterseite sichtbar | Kein Startup Command, Oryx erkennt Flask nicht automatisch | Startup Command (`gunicorn ... app:app`) explizit gesetzt |
| Deployment Slots nicht verfügbar | Free-Tier unterstützt keine Slots | Kurzzeitig auf Standard (S1) hochskaliert |
| Beide Slots zeigten gleichen Inhalt | Beide auf denselben Branch konfiguriert | Getrennte Branches (main/staging) + Deployment Center neu verbunden |
| 409-Conflict beim Deploy | Überschneidende Deployment-Operationen | Workflow erneut ausgeführt |
| Automatic Scaling nicht wählbar | Erfordert Premium v2/v3 | "Rules Based" Scaling stattdessen gewählt |
