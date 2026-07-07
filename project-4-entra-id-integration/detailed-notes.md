# Projekt 4 — Detaillierte Notizen: Schritte, Begründungen, Probleme

## 1. Nutzer und Gruppen anlegen

**Was:** Drei Testnutzer (Anna Admin, David Dev, Rita Reader) in Entra ID angelegt, dazu eine Security Group `app-project4-users`, die alle drei enthält.

**Warum:** Rollenbasierte, gruppenzentrierte Zugriffsverwaltung ist Standard in Unternehmensumgebungen — Berechtigungen werden an Gruppen statt an einzelne Nutzer vergeben, damit Zugriffsänderungen skalierbar bleiben (neue Mitarbeiter → einfach zur Gruppe hinzufügen).

Dieser Schritt verlief ohne Probleme.

## 2. App Registration

**Was:** Eine App `az104-project4-webapp` registriert (Single Tenant, Redirect URI `https://localhost:5000` als Platzhalter), API-Permission `User.Read` für Microsoft Graph mit Admin Consent, ein Client Secret erzeugt.

**Warum:** App Registrations sind die Grundlage, damit eine Anwendung sich gegenüber Entra ID authentifizieren und im Namen von Nutzern auf Microsoft-Graph-Daten zugreifen kann. Admin Consent ist nötig, damit nicht jeder einzelne Nutzer beim ersten Login manuell zustimmen muss.

Dieser Schritt verlief ohne Probleme.

## 3. Gruppenbasierte App-Zuweisung — der Lizenz-Stolperstein

**Was:** Die Security Group sollte der App über Enterprise Applications zugewiesen werden, um Zugriff zu steuern.

**Problem #1 — "Groups are not available for assignment due to your Active Directory plan level":**
Beim Versuch, die Gruppe zuzuweisen, erschien diese Meldung — nur einzelne Nutzer waren auswählbar, keine Gruppen.

**Diagnose:** Gruppenbasierte App-Zuweisung ist ein **Entra ID P1/P2-Feature**, im Free-Tier (Standard bei neuen Subscriptions ohne separate Lizenzierung) nicht verfügbar.

**Workaround (Zwischenlösung):** Zunächst die drei Nutzer einzeln statt der Gruppe zugewiesen, um weiterzumachen.

**Problem #2 — Trial-Aktivierung erzeugte einen komplett neuen Tenant:**
Um die Lizenzgrenze aufzuheben, wurde versucht, einen kostenlosen Entra ID P2 Trial zu aktivieren. Nach der Aktivierung (inkl. Ausfüllen von Firmenname/Kontodaten) zeigte sich beim Tenant-Wechsel: der Trial war in einem **komplett neuen, leeren Tenant** gelandet — nicht im bestehenden Tenant mit den angelegten Nutzern, der App-Registrierung und der Gruppe.

**Diagnose:** Der ursprüngliche Azure-Tenant basiert auf einem **privaten/Consumer-Microsoft-Konto**, nicht auf einem "Work or School"-Konto. Der M365-Admin-Center-Trial-Flow für Entra ID P1/P2 ist auf Organisationskonten ausgelegt und erstellt bei einem Consumer-Konto offenbar einen neuen Business-Tenant, statt das bestehende Directory zu aktualisieren. Bestätigt wurde das zusätzlich dadurch, dass der Login ins M365 Admin Center nur mit dem neu erstellten "Business"-Account funktionierte, nicht mit dem ursprünglichen Consumer-Account.

**Lösung:** Kein weiterer Versuch, den Trial in den ursprünglichen Tenant zu bekommen — stattdessen pragmatisch mit Einzelnutzer-Zuweisung weitergearbeitet und Conditional Access (siehe unten) konzeptionell statt live dokumentiert. **Lektion für zukünftige Projekte:** Für Entra-ID-Premium-lastige Themen (v. a. Richtung AZ-500) einen dedizierten **Microsoft 365 Developer Program**-Tenant nutzen (kostenloser E5-Sandbox-Tenant mit allen Premium-Features, kein Trial-Aktivierungs-Chaos, keine Kreditkarte nötig) statt zu versuchen, den bestehenden Consumer-gebundenen Tenant nachträglich upzugraden.

## 4. Conditional Access — konzeptionell dokumentiert

**Was:** Da Conditional Access ebenfalls ein P1/P2-Feature ist und im aktuellen Tenant nicht verfügbar war, wird der Aufbau hier konzeptionell beschrieben statt live umgesetzt:

- **Users:** Zielgruppe `app-project4-users`
- **Target resources:** die registrierte App `az104-project4-webapp`
- **Grant:** "Require multifactor authentication"
- **Modus:** Empfehlung, neue Policies zunächst im **Report-only**-Modus zu testen, bevor sie erzwungen werden (verhindert versehentliches Aussperren von Nutzern)

**Warum dieser Ansatz:** Anstatt Zeit in weitere Workarounds zu investieren, wurde die reale Lizenzgrenze als das behandelt, was sie ist — ein legitimer Lernpunkt über Entra-ID-Editionsunterschiede, der in einer echten Umgebung genauso auftreten würde, wenn ein Unternehmen noch keine P1/P2-Lizenzen eingekauft hat.

## 5. MFA — Umsetzung und Verifikation

**Was:** Per-user MFA für alle drei Testnutzer aktiviert. Bei Anna Admin wurde der komplette Flow durchgespielt: Login, Passwort-Reset, MFA-Registrierung über Microsoft Authenticator (mit dem eigenen Smartphone, das problemlos mehrere Konten parallel verwalten kann).

**Problem #3 — Erster Login-Versuch schlug fehl (Fehlercode 50126):**
Der erste Anmeldeversuch mit Anna Admin zeigte im Sign-in-Log den Status "Failure" mit Fehlercode 50126 ("Error validating credentials due to invalid username or password").

**Diagnose:** Schlicht ein falsches/abgelaufenes Passwort beim ersten Versuch — kein technisches Problem, sondern normales Nutzerverhalten (genau die Art von Eintrag, die man in echten Sign-in-Logs auch erwartet).

**Lösung:** Passwort für Anna Admin zurückgesetzt, danach erfolgreicher Login mit anschließender MFA-Registrierung. Der zweite Sign-in-Log-Eintrag zeigte danach korrekt **Status: Success**.

## 6. Sign-in-Monitoring

**Was:** Sign-in-Logs nach Anna Admin gefiltert, um sowohl den fehlgeschlagenen als auch den erfolgreichen Login-Versuch nachzuvollziehen.

**Beobachtung:** Beide Ereignisse waren sauber protokolliert, inklusive IP-Adresse, verwendeter Anwendung (Azure Portal), Ressource (Azure Resource Manager) und Authentication Requirement. Das bestätigt, dass Entra ID Sign-in-Logs auch ohne Premium-Lizenz vollständig funktionieren — nur die *proaktive Steuerung* darüber (Conditional Access) ist lizenzpflichtig, die *Sichtbarkeit* selbst nicht.

## Zusammenfassung der Kernprobleme

| Problem | Ursache | Fix |
|---|---|---|
| Gruppen nicht für App-Zuweisung verfügbar | Entra ID Free-Tier, Feature erfordert P1/P2 | Zunächst Einzelnutzer-Zuweisung als Workaround |
| P2-Trial-Aktivierung erzeugte neuen, leeren Tenant | Tenant basiert auf privatem statt Work/School-Konto | Kein weiterer Versuch; Empfehlung für Microsoft 365 Developer Program für zukünftige Projekte |
| Conditional Access nicht verfügbar | Gleiches Lizenz-Problem wie oben | Policy konzeptionell dokumentiert statt live konfiguriert |
| Erster Login-Versuch fehlgeschlagen (Code 50126) | Falsches Passwort beim ersten Versuch | Passwort zurückgesetzt, danach erfolgreicher Login + MFA-Registrierung |
