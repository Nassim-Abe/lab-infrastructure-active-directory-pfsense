# 4. Configuration pfSense CE

## Interfaces

| Interface | Adresse | Fonction |
|---|---|---|
| WAN | DHCP/automatique sur NAT (ou pont vers l’hôte) | Internet/réseau hôte |
| LAN | `10.10.20.1/24` | `SEG_SRV` |
| OPT1 | `10.10.10.1/24` | `SEG_USR` |

Interface web documentée : `https://10.10.20.1`. Les identifiants par défaut `admin / pfsense` apparaissent dans une source : les changer immédiatement et ne jamais les conserver dans le dépôt.

## DHCP, DNS et NAT

- Activer le serveur DHCP pfSense sur LAN et OPT1 ; désactiver le DHCP VMware.
- Ajouter au besoin les mappings statiques MAC → `10.10.20.10` pour DC01 et MAC → `10.10.10.20` pour RODC01 via **Status → DHCP Leases → Add Static Mapping**.
- Activer le NAT sortant WAN pour permettre la sortie des réseaux internes.
- Configurer DNS Resolver/Forwarder pour s’appuyer sur DC01. Pour le domaine AD, les DC restent leurs propres DNS préférés afin de préserver les SRV AD.

## Filtrage

- **WAN** : conserver le refus entrant par défaut.
- **LAN** : autoriser les flux nécessaires vers pfSense et les services AD.
- **OPT1** : autoriser les clients vers les services AD/DNS requis ; refuser les accès génériques au réseau serveur.
- **Inter-segments** : ouvrir explicitement les flux nécessaires vers DC01/RODC01, pas un accès large.

Ports à prendre en compte :

| Service | Ports | Usage |
|---|---|---|
| DNS | 53 TCP/UDP | Résolution AD/clients |
| Kerberos | 88 TCP/UDP | Authentification |
| LDAP/LDAPS | 389 TCP/UDP, 636 TCP | Annuaire |
| SMB | 445 TCP | SYSVOL/NETLOGON |
| RPC | 135 TCP + ports dynamiques | Administration/réplication |
| Global Catalog | 3268/3269 TCP | Recherche AD/GC |

Les ports RPC dynamiques doivent être bornés en production. Vérifier aussi routes, heure système et résolution de noms lors du dépannage.
