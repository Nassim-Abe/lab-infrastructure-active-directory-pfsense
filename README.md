# Laboratoire AD DS, RODC et pfSense

[![Statut](https://img.shields.io/badge/Statut-lab%20document%C3%A9%20%26%20valid%C3%A9-success)](docs/05-tests-validation.md)
[![Hyperviseur](https://img.shields.io/badge/Hyperviseur-VMware%20Workstation-607078?logo=vmware)](docs/01-preparation-environnement.md)
[![Windows Server](https://img.shields.io/badge/Windows%20Server-2019-0078D6?logo=windows)](docs/02-installation-ad-ds.md)
[![Pare-feu](https://img.shields.io/badge/Pare--feu-pfSense%20CE-212121)](docs/04-configuration-pfsense.md)
[![Licence](https://img.shields.io/badge/Licence-MIT-green)](LICENSE)

Documentation reproductible d’un laboratoire PME simulé : **Windows Server 2019**, Active Directory Domain Services, DNS, DHCP, **RODC** et pfSense CE sous VMware Workstation. Les fichiers sont organisés pour être suivis dans l’ordre, y compris par un débutant.

> **Note sur les sources :** un document source mentionne Windows Server 2019, tandis que la version la plus complète et la plus récente documente Windows Server 2019. Cette refonte retient 2019 ; les procédures AD DS restent transposables à 2019.

## Sommaire

- [Contexte et objectifs](#contexte-et-objectifs)
- [Architecture](#architecture)
- [Adressage](#adressage)
- [Prérequis](#prérequis)
- [Structure du dépôt](#structure-du-dépôt)
- [Reproduire le lab](#reproduire-le-lab)
- [Validation](#validation)
- [Limites et améliorations](#limites-et-améliorations)
- [Auteur et licence](#auteur-et-licence)

## Contexte et objectifs

- Centraliser l’authentification dans le domaine `entreprise.lab` (NetBIOS `ENTREPRISE`) avec `DC01`, contrôleur inscriptible hébergeant AD DS, DNS et DHCP.
- Déployer `RODC01`, contrôleur en lecture seule dans le segment utilisateurs/filiale, avec réplication unidirectionnelle.
- Isoler les serveurs (`SEG_SRV`) et les utilisateurs (`SEG_USR`) derrière pfSense.
- Joindre `CLI01` à `CLI10` au domaine et appliquer la GPO **Corporate Wallpaper** avec une cible de moins de 90 secondes.
- Vérifier la dépendance critique AD ↔ DNS, les enregistrements SRV, la réplication, la PRP et le mode dégradé.

## Architecture

```text
                         Internet / hôte
                              │ WAN — NAT
                              ▼
                    ┌────────────────────┐
                    │      pfSense CE    │
                    │ LAN 10.10.20.1/24  │──── VMnet2 / SEG_SRV
                    │ OPT1 10.10.10.1/24 │──── VMnet1 / SEG_USR
                    └─────────┬──────────┘
                              │
       ┌──────────────────────┴──────────────────────┐
       ▼                                             ▼
  DC01 10.10.20.10/24                         RODC01 10.10.10.20/24
  RWDC • AD DS • DNS • DHCP                    RODC • DNS • filiale
  (DNS préféré : lui-même)                     (DNS préféré : lui-même)
                                                    │
                                               CLI01…CLI10 (DHCP)
```

## Adressage

| Équipement | Rôle | Adresse / masque | Passerelle | DNS préféré |
|---|---|---|---|---|
| pfSense WAN | NAT | DHCP/automatique | — | — |
| pfSense LAN | Serveurs | `10.10.20.1/24` | `10.10.20.1` | `127.0.0.1` |
| pfSense OPT1 | Utilisateurs | `10.10.10.1/24` | `10.10.10.1` | `127.0.0.1` |
| DC01 | RWDC / AD DS / DNS / DHCP | `10.10.20.10/24` | `10.10.20.1` | `10.10.20.10` |
| RODC01 | Contrôleur en lecture seule | `10.10.10.20/24` | `10.10.10.1` | `10.10.10.20` |
| SRV02 (optionnel) | Serveur supplémentaire | `10.10.20.20/24` | `10.10.20.1` | DNS AD |
| CLI01…CLI10 | Clients | DHCP sur OPT1 | `10.10.10.1` | pfSense/AD selon bail |

| VMnet | Sous-réseau | Mode | DHCP VMware |
|---|---|---|---|
| VMnet1 | `10.10.10.0/24` | Host-only | Désactivé |
| VMnet2 | `10.10.20.0/24` | Host-only | Désactivé |

> Le DHCP doit être fourni par pfSense sur LAN et OPT1 afin d’éviter deux serveurs DHCP concurrents. Prévoir des réservations MAC pour DC01 et RODC01 si nécessaire.

## Prérequis

- VMware Workstation Pro (VirtualBox est possible en adaptant les noms VMnet et les réseaux host-only).
- ISO pfSense CE, Windows Server 2019 pour DC01/RODC01 et Windows 11 pour les clients ; les sources indiquent le portail de licences Microsoft pour les ISO.
- Recommandation de lab : 2 vCPU et 4–8 Go RAM par serveur Windows, 2 vCPU et 2–4 Go pour pfSense, 2–4 Go par client ; prévoir environ 24 Go RAM pour un scénario complet avec plusieurs VMs.
- Deux réseaux virtuels host-only et un réseau NAT pour le WAN. Les adresses indiquées sont de laboratoire et ne doivent pas être réutilisées sur un réseau de production.

## Structure du dépôt

```text
.
├── README.md
├── LICENSE
├── CONTRIBUTING.md
├── .gitignore
├── GUIDE_PUBLICATION_GITHUB.md
├── docs/
│   ├── 01-preparation-environnement.md
│   ├── 02-installation-ad-ds.md
│   ├── 03-configuration-rodc.md
│   ├── 04-configuration-pfsense.md
│   ├── 05-tests-validation.md
│   └── screenshots/              # captures à ajouter
└── Media/screenshots/             # emplacement conseillé pour les captures
```

## Reproduire le lab

1. Préparer VMware et les trois réseaux selon [la préparation](docs/01-preparation-environnement.md).
2. Installer/configurer pfSense, ses interfaces, son DHCP, son DNS Resolver et son NAT ([guide pfSense](docs/04-configuration-pfsense.md)).
3. Installer AD DS/DNS/DHCP sur DC01 et créer `entreprise.lab` ([installation AD DS](docs/02-installation-ad-ds.md)).
4. Joindre puis promouvoir RODC01 et configurer sa PRP ([configuration RODC](docs/03-configuration-rodc.md)).
5. Joindre les clients, déployer **Corporate Wallpaper**, puis exécuter les contrôles ([tests](docs/05-tests-validation.md)).

## Validation

Les sources rapportent : DC01 et RODC01 promus, RODC01 Global Catalog, SRV AD résolus, réplication `repadmin` sans erreur, PRP configurée, réseaux isolés, DHCP pfSense actif, pings et DNS depuis OPT1 fonctionnels, et GPO appliquée en moins de 90 secondes. Rejouer les tests dans l’environnement cible avant toute mise en production.

## Limites et améliorations

- Le lab n’est pas une architecture de production : pas de haute disponibilité pfSense, de second RWDC, de sauvegarde/restauration testée ni de PKI.
- Les ports RPC dynamiques doivent être bornés/durcis en production ; les règles inter-segments doivent être testées flux par flux.
- Compléter les OU, délégations, comptes de service gMSA, journalisation, supervision, NTP et durcissement Windows.
- Automatiser les scripts et ajouter des captures dans `Media/screenshots/` ou `docs/screenshots/`.

## Auteur et licence

Auteur/contact : **Nassim Abe** — remplacer ce libellé par le contact public souhaité.

Distribué sous licence [MIT](LICENSE).
