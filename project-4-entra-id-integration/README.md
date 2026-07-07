# AZ-104 Projekt 4: Entra ID Integration & Identity Management

Ziel: Nutzer/Gruppen in Entra ID verwalten, eine Applikation registrieren und per Gruppe zuweisen, Zugriff über Conditional Access und MFA absichern, Sign-in-Aktivität überwachen.

## Architektur-Überblick

- **Nutzer:** `Anna Admin`, `David Dev`, `Rita Reader` (fiktive Testnutzer mit unterschiedlichen Rollen)
- **Security Group:** `app-project4-users` (enthält die drei Testnutzer, eigener Account als Owner)
- **App Registration:** `az104-project4-webapp` — Single Tenant, Redirect URI `https://localhost:5000`, API-Permission `User.Read` (Microsoft Graph, Admin Consent erteilt)
- **App-Zuweisung:** Gruppenbasiert über Enterprise Applications (nach Aktivierung von Entra ID P2 Trial)
- **Conditional Access:** konzeptionell dokumentiert (siehe Troubleshooting — im Free-Tier nicht ausführbar)
- **MFA:** Per-user MFA für alle drei Testnutzer aktiviert, für `Anna Admin` vollständig durchgetestet (Registrierung + Login)
- **Monitoring:** Sign-in-Logs verifiziert (ein Failure- und ein Success-Eintrag für Anna Admin)

## Umgesetzte Schritte

| # | Schritt | Screenshot |
|---|---|---|
| 1 | Nutzer Anna/David/Rita angelegt | `screenshots/entra-users-overview.png` |
| 2 | Security Group `app-project4-users` erstellt | `screenshots/entra-group-overview.png` |
| 3 | App Registration + API Permissions (User.Read, Admin Consent) | `screenshots/app-registration-permissions.png` |
| 4 | Gruppenbasierte App-Zuweisung in Enterprise Applications | `screenshots/enterprise-app-assignment.png` |
| 5 | Per-user MFA für alle drei Nutzer aktiviert | `screenshots/mfa-per-user-enabled.png` |
| 6 | Sign-in-Logs: Failure + Success für Anna Admin | `screenshots/sign-in-logs-anna.png` |

## Was ich gelernt habe

- **Entra ID Free vs. P1/P2** ist keine Kleinigkeit: Gruppenbasierte App-Zuweisung und Conditional Access sind reine Premium-Features, die im Free-Tier schlicht nicht verfügbar sind.
- Ein Azure-Tenant, der auf einem **privaten Microsoft-Konto** statt einem "Work or School"-Konto basiert, hat Reibungspunkte bei M365-Admin-Center-Workflows (z. B. Trial-Aktivierungen, die einen komplett neuen Tenant statt eines Upgrades erzeugen).
- **Microsoft Authenticator** verwaltet problemlos mehrere Konten (privat, Arbeit, Testnutzer) parallel auf einem Gerät.
- Sign-in-Logs erfassen auch fehlgeschlagene Login-Versuche (z. B. falsches Passwort) mit klarem Fehlercode — nützlich, um zu verifizieren, dass Monitoring tatsächlich greift.

Ausführliche Schritt-für-Schritt-Erklärung inkl. aller Troubleshooting-Schritte: siehe [`detailed-notes.md`](./detailed-notes.md).

## Cleanup

- App Registration, Security Group und Testnutzer können bestehen bleiben (kosten nichts, Entra ID Objekte sind kostenlos) oder bei Bedarf gelöscht werden
- Empfehlung für zukünftige Entra-ID-Premium-Themen (AZ-500): separater Microsoft 365 Developer Program Tenant statt Trial-Aktivierung im bestehenden Consumer-Tenant
