# Laboratoire AD, RODC et pfSense

Documentation technique d’un laboratoire PME simulé sous **VMware Workstation**, avec **Windows Server 2019**, **Active Directory DS**, **DNS**, **RODC** et **pfSense CE**.

> Les résultats « validés » ci-dessous reprennent les vérifications décrites dans les documents sources fournis. Ils doivent être rejoués dans l’environnement cible avant toute mise en production.

## Objectifs

- Déployer `DC01`, contrôleur de domaine inscriptible, avec AD DS et DNS intégrés.
- Déployer `RODC01`, contrôleur de domaine en lecture seule pour le segment filiale.
- Isoler les serveurs et les utilisateurs derrière pfSense.
- Joindre les clients au domaine et tester l’application des GPO.
- Vérifier DNS, réplication AD, PRP et fonctionnement en mode dégradé.

## Topologie réseau

```text
Internet / hôte
       │ WAN — NAT
       ▼
┌──────────────────┐
│   pfSense CE     │
│ LAN  10.10.20.1  │──── VMnet2 — SEG_SRV — DC01 10.10.20.10
│ OPT1 10.10.10.1  │──── VMnet1 — SEG_USR — RODC01 10.10.10.20
└──────────────────┘                         └── CLI01…CLI10 (DHCP)
```

## Adressage IP

| Équipement | Rôle | Adresse / masque | Passerelle | DNS préféré |
|---|---|---:|---:|---:|
| pfSense | WAN NAT | DHCP / automatique | — | — |
| pfSense | LAN serveurs | `10.10.20.1/24` | `10.10.20.1` | `127.0.0.1` |
| pfSense | OPT1 utilisateurs | `10.10.10.1/24` | `10.10.10.1` | `127.0.0.1` |
| DC01 | RWDC / AD DS / DNS / DHCP | `10.10.20.10/24` | `10.10.20.1` | `10.10.20.10` |
| RODC01 | Contrôleur en lecture seule | `10.10.10.20/24` | `10.10.10.1` | `10.10.10.20` |

| VMnet | Réseau | Mode | DHCP VMware |
|---|---|---|---|
| VMnet1 | `10.10.10.0/24` | Host-only | Désactivé |
| VMnet2 | `10.10.20.0/24` | Host-only | Désactivé |

Le DHCP est fourni par pfSense sur LAN et OPT1. Les réservations MAC de `DC01` et `RODC01` sont recommandées.

## Règles et segmentation pfSense

- **WAN** : politique entrante restrictive ; NAT sortant vers le réseau hôte.
- **LAN** : segment serveurs, avec accès contrôlé aux services AD/DNS.
- **OPT1** : segment clients et filiale ; autoriser uniquement les flux AD nécessaires vers `DC01` et `RODC01`.
- **Inter-segments** : refuser les accès génériques au réseau serveurs ; ouvrir explicitement DNS, Kerberos, LDAP/LDAPS, SMB, RPC et Global Catalog selon le besoin.
- **DNS** : le domaine `corp.lan` dépend des enregistrements SRV AD ; chaque DC se référence lui-même comme DNS préféré.

Ports courants à considérer : `53 TCP/UDP` DNS, `88 TCP/UDP` Kerberos, `389 TCP/UDP` LDAP, `445 TCP` SMB, `135 TCP` RPC, `3268/3269 TCP` Global Catalog et ports RPC dynamiques pour la réplication.

## Déploiement AD et RODC

Domaine : `corp.lan` ; nom NetBIOS : `CORP`.

```powershell
# DC01
Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools
Install-ADDSForest -DomainName "corp.lan" -DomainNetBIOSName "CORP" -InstallDns:$true -Force

# RODC01
Add-Computer -DomainName "corp.lan" -Credential (Get-Credential CORP\Administrator) -Restart
Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools
Install-ADDSDomainController -DomainName "corp.lan" -InstallDns:$true -ReadOnlyReplica:$true -Force
```

## Vérification DNS et réplication

```powershell
Get-ADDomainController -Filter * |
  Select-Object Name,IPv4Address,IsGlobalCatalog,OperatingSystem

nslookup -type=srv _ldap._tcp.dc._msdcs.corp.lan 10.10.20.10

repadmin /replsummary
repadmin /showrepl DC01
repadmin /showrepl RODC01
repadmin /syncall /AeP
Get-ADReplicationPartnerMetadata -Target RODC01
```

Résultats attendus : contrôleurs visibles, enregistrements SRV résolus et absence d’erreurs de réplication.

## GPO et tests clients

```powershell
Add-Computer -DomainName "corp.lan" -Credential (Get-Credential CORP\Administrator) -Restart
gpupdate /force
gpresult /r
```

La GPO `Corporate Wallpaper` peut utiliser le partage :

```text
\\DC01\NETLOGON\wallpaper\corp.jpg
```

La cible documentée est une application en moins de 90 secondes.

## PRP du RODC

Le cache des mots de passe est vide par défaut. Autoriser uniquement les utilisateurs de la filiale via un groupe dédié ; refuser les comptes privilégiés et les comptes de service sensibles.

```powershell
Get-ADDomainController -Identity RODC01 | Set-ADObject -Add @{
  "msDS-RevealOnDemandGroup" = @(
    "CN=Allowed RODC Password Replication Group,CN=Groups,DC=corp,DC=lan"
  )
}
```

## Validation et mode dégradé

| Vérification | Commande / observation |
|---|---|
| Réseau | `ipconfig /all`, `ping`, routes pfSense |
| DNS AD | `nslookup -type=srv _ldap._tcp.dc._msdcs.corp.lan` |
| Réplication | `repadmin /replsummary`, `repadmin /showrepl` |
| GPO | `gpupdate /force`, `gpresult /r` |
| DNS arrêté | L’ouverture de session de domaine échoue ; après redémarrage du DNS, le fonctionnement revient selon le scénario source |

## Arborescence du dépôt

```text
.
├── README.md
├── Documentation_Lab_AD_RODC_pfSense.docx
├── docs/
│   └── screenshots/
├── scripts/
│   ├── powershell/
│   └── pfsense/
└── diagrams/
```

## Conclusion

La conception sépare les segments, centralise l’identité sur `DC01` et apporte un point de service local en filiale avec `RODC01`. La disponibilité de DNS, la réplication AD, la PRP et les règles inter-segments sont les contrôles prioritaires.
