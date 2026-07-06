# AZ-104 Projekt 2: Azure Networking & Storage

Ziel: Ein VNet mit isolierten Subnets aufbauen, den Storage-Zugriff über Private Endpoint statt öffentlichem Internet absichern, NSG-Regeln nach Least-Privilege durchsetzen und das Deployment per ARM-Template-Export reproduzierbar machen.

## Architektur-Überblick

- **Resource Group:** `RG-AZ104-PROJECT2` (eigene RG pro Projekt → einfaches Cleanup am Ende)
- **VNet:** 10.0.0.0/16 mit zwei Subnets:
  - `Database` (10.0.0.0/27) — Private Subnet (kein Default-Outbound), Service Endpoint `Microsoft.Storage`
  - `Web` (10.0.1.0/27) — NSG-geschützt, nur HTTPS eingehend erlaubt
- **NSG (`nsg-project2-web`):** Allow HTTPS (443) → Deny All (Rest), assoziiert mit dem `Web`-Subnet
- **Storage Account:** Public Network Access **deaktiviert**, Zugriff ausschließlich über Private Endpoint im `Database`-Subnet, LRS-Replikation, Microsoft-managed Encryption
- **Infrastructure as Code:** VNet + Storage Account als ARM-Templates exportiert und als Template Specs registriert (reproduzierbares Deployment)
- **Verifikation:** Temporäre Test-VM im `Web`-Subnet, Zugriff auf den Storage Account per SAS-Token via AzCopy erfolgreich getestet — beweist, dass der Private Endpoint funktioniert, obwohl kein öffentlicher Zugriff möglich ist

## Umgesetzte Schritte

| # | Schritt | Screenshot |
|---|---|---|
| 1 | VNet mit `Database`- und `Web`-Subnet erstellt | `screenshots/vnet-overview.png` |
| 2 | NSG erstellt, mit `Web`-Subnet assoziiert, Allow HTTPS + Deny All Regeln gesetzt | `screenshots/nsg-rules.png` |
| 3 | Storage Account erstellt, Public Access deaktiviert | `screenshots/storage-account-networking.png` |
| 4 | Private Endpoint im `Database`-Subnet angelegt, Verbindung "Approved" | `screenshots/private-endpoint-overview.png` |
| 5 | VNet + Storage Account als ARM-Template exportiert und als Template Specs importiert | `screenshots/template-specs-overview.png` |
| 6 | SAS-Token generiert, Zugriff über temporäre Test-VM mit AzCopy erfolgreich verifiziert | `screenshots/azcopy-upload-success.png` |

## Was ich gelernt habe

- **Private Endpoints** machen einen Dienst faktisch unsichtbar fürs öffentliche Internet — selbst ein gültiges SAS-Token nützt nichts ohne Netzwerkpfad ins VNet.
- Azure legt beim VNet-Erstellen automatisch ein erstes Subnet namens `default` an — dieses muss man bewusst umbenennen/anpassen, statt es zu übernehmen.
- **Template Specs** erwarten die reine `template.json`, nicht das komplette Export-ZIP und nicht die `parameters.json`.
- Azure Cloud Shell ist ein komplett eigenes Dateisystem, getrennt vom lokalen Rechner — ein lokaler Download landet nicht automatisch dort.
- Temporärer Zugriff (z. B. SSH nur für Testzwecke) sollte immer scope-begrenzt (`My IP` statt `Any`) und nach Gebrauch wieder entfernt werden.

Ausführliche Schritt-für-Schritt-Erklärung inkl. aller Troubleshooting-Schritte: siehe [`detailed-notes.md`](./detailed-notes.md).

## Cleanup

- Temporäre NSG-Regel `Temp_Allow_SSH` entfernen (falls noch nicht geschehen)
- Temporäre Test-VM löschen (war nicht Teil der eigentlichen Projekt-Ressourcen)
- Nach Abschluss der Dokumentation: `RG-AZ104-PROJECT2` komplett löschen, um weitere Kosten zu vermeiden
