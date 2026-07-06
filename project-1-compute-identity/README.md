# AZ-104 Projekt 1: Azure Compute & Identity Management

Ziel: Eine VM sicher deployen, Zugriff über RBAC + Entra ID + Key Vault regeln, Governance per Azure Policy durchsetzen und Kosten überwachen.

## Architektur-Überblick

- **Resource Group:** `RG-AZ104-PROJECT1` (eigene RG pro Projekt → einfaches Cleanup am Ende)
- **VM:** `vm-project1-web` — Ubuntu 24.04 LTS, Standard D2s v3, Poland Central
- **Zugriff:** SSH-Key-Authentifizierung (kein Passwort), privater Key im Key Vault gesichert
- **RBAC:** Entra-Security-Gruppe `az104-project-admins` mit Key Vault Administrator-Rolle
- **Netzwerk:** NSG erlaubt nur Port 22 (SSH) eingehend, alles andere per Default-Deny blockiert
- **Governance:** Azure Policy Initiative erzwingt Tag-Vererbung (`Environment`) und beschränkt erlaubte Ressourcentypen auf VMs
- **Kosten:** Monatliches Budget (30 $) mit Alert-Schwellen bei 50/80/100 %

## Umgesetzte Schritte

| # | Schritt | Screenshot |
|---|---|---|
| 1 | VM erstellt (Ubuntu, SSH-Key-Auth, eigene RG) | `screenshots/vm-overview.png` |
| 2 | NSG: nur SSH (22) erlaubt, Rest Default-Deny | `screenshots/vm-networking-nsg.png` |
| 3 | Key Vault erstellt, privater SSH-Key importiert | — |
| 4 | Entra-Gruppe `az104-project-admins` angelegt, Key Vault Administrator zugewiesen | `screenshots/keyvault-iam.png` |
| 5 | Gruppenmitgliedschaft verifiziert | `screenshots/entra-group-overview.png` |
| 6 | SSH-Login über Azure Cloud Shell erfolgreich getestet | `screenshots/ssh-login-success.png` |
| 7 | Azure Policy Initiative `az104-project1-governance` erstellt und zugewiesen | `screenshots/policy-assignment.png` |
| 8 | Compliance-Status: 100 % konform | `screenshots/vm-policy-compliance.png` |
| 9 | Budget mit Alert-Schwellen eingerichtet | `screenshots/cost-budget.png` |

*(Screenshots liegen im Ordner `screenshots/` — bitte vor dem Commit dort ablegen.)*

## Was ich gelernt habe

- RBAC über eine **Gruppe statt einzelner Nutzer** zu vergeben skaliert besser — neue Teammitglieder bekommen Zugriff einfach durch Gruppenmitgliedschaft.
- Azure Policy trennt **Definition** und **Zuweisung (Assignment)** als zwei separate Schritte — eine Definition allein bewirkt noch nichts.
- Kostendaten und Advisor-Empfehlungen haben spürbare Latenz (Stunden bis Tage) — kein Grund zur Sorge bei "0 €" direkt nach dem Deployment.

Ausführliche Schritt-für-Schritt-Erklärung inkl. aller Troubleshooting-Schritte: siehe [`detailed-notes.md`](./detailed-notes.md).

## Cleanup

Nach Abschluss der Dokumentation: `RG-AZ104-PROJECT1` komplett löschen, um weitere Kosten zu vermeiden.
