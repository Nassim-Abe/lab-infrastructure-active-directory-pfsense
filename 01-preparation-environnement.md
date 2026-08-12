# 1. Préparation de l’environnement

## Composants

| Composant | Choix documenté |
|---|---|
| Hyperviseur | VMware Workstation Pro |
| Pare-feu/routeur | pfSense CE |
| DC principal | Windows Server 2019 (la source alternative indique 2019) |
| RODC | Windows Server 2019 |
| Clients | Windows 11, `CLI01` à `CLI10` |
| Documentation | Markdown, Mermaid et Pandoc |

## Réseaux VMware

Créer deux réseaux **Host-only** dans l’éditeur réseau et désactiver le DHCP VMware :

| Réseau | Sous-réseau | Usage |
|---|---|---|
| VMnet1 | `10.10.10.0/24` | `SEG_USR`, OPT1, RODC01 et clients |
| VMnet2 | `10.10.20.0/24` | `SEG_SRV`, LAN et DC01 |

Raccorder le WAN pfSense au réseau NAT de l’hôte, son LAN à VMnet2 et son OPT1 à VMnet1. Créer les VMs Windows et prévoir des réservations MAC pfSense pour les IP fixes.

## Ressources conseillées

Pour un lab confortable : 2 vCPU/4–8 Go RAM par serveur Windows, 2 vCPU/2–4 Go pour pfSense et 2–4 Go par client. Prévoir environ 24 Go RAM pour pfSense, DC01, RODC01 et plusieurs clients en parallèle. Les VMs doivent rester isolées du réseau réel.

## Ordre de démarrage

1. pfSense ; 2. DC01 ; 3. RODC01 ; 4. clients. Ne pas activer simultanément le DHCP VMware et celui de pfSense.
