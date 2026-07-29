# lab-infrastructure-active-directory-pfsense
 Déploiement d'une infrastructure d'entreprise virtuelle (Active Directory WS2019, pfSense, VMware).
# 🛠️ Labo Intégré Active Directory & Pare-feu pfSense (VMware Workstation)

> **Description du projet :** Conception, déploiement et validation d'une infrastructure réseau et d'un contrôleur de domaine Windows Server 2022 segmenté via pare-feu pfSense en environnement virtuel.

---

## 🌐 1. Architecture & Topologie Réseau

L'environnement est isolé du réseau physique à l'aide de réseaux virtuels **Host-Only** sur VMware Workstation. La segmentation permet d'isoler la zone réseau des serveurs de celle des utilisateurs.

```mermaid
flowchart TD
    subgraph Internet / WAN
        WAN[Internet / Réseau Hôte]
    end

    subgraph pfSense [Pare-feu pfSense]
        pWAN[Interface WAN - NAT]
        pLAN[Interface LAN - 10.10.20.1/24]
        pOPT1[Interface OPT1 - 10.10.10.1/24]
    end

    subgraph VMnet2 [VMnet2 : Segment Serveurs 10.10.20.0/24]
        DC01["🖥️ DC01 (Windows Server 2022)<br/>IP: 10.10.20.10<br/>Rôles: AD DS / DNS / DHCP"]
    end

    subgraph VMnet1 [VMnet1 : Segment Utilisateurs 10.10.10.0/24]
        CLI01["💻 Postes de travail Clients<br/>DHCP via pfSense / DC01"]
    end

    WAN <--> pWAN
    pLAN <-->|Switch Virtuel VMnet2| DC01
    pOPT1 <-->|Switch Virtuel VMnet1| CLI01

📊 2. Plan d'Adressage IP & Interconnexions
Copier le tableau


Appareil / VM
Rôle / Fonction
Interface
Adresse IP / Masque
Passerelle
DNS Préféré
Commutateur Virtuel



pfSense
Pare-feu / Routeur
WAN
DHCP (NAT)
Auto
127.0.0.1
NAT (Temporaire)


pfSense
Passerelle LAN
LAN (Serveurs)
10.10.20.1/24
10.10.20.1
127.0.0.1
VMnet2 (Host-Only)


pfSense
Passerelle OPT1
OPT1 (Utilisateurs)
10.10.10.1/24
10.10.10.1
127.0.0.1
VMnet1 (Host-Only)


DC01
Contrôleur de Domaine
LAN
10.10.20.10/24
10.10.20.1
10.10.20.10
VMnet2 (Host-Only)



✅ 3. Points Critiques & Validations Techniques

 Isolation Réseau : Configuration des VMnet1 et VMnet2 en Host-Only sans serveur DHCP VMware actif.
 Services pfSense : Serveur DHCP actif sur les segments LAN/OPT1 et règles d'inter-routage configurées.
 Auto-référencement DNS AD : DC01 configuré pour pointer vers son propre service DNS (10.10.20.10) pour assurer le bon fonctionnement du domaine.
 Connectivité & Résolution : Validation bidirectionnelle du réseau (ping, nslookup).
 Résolution des enregistrements SRV : Vérification de la disponibilité des enregistrements _ldap._tcp.dc._msdcs nécessaires à l'authentification Active Directory.


🧪 4. Démarche Expérimentale & Test de Résilience
Afin de valider la dépendance critique entre Active Directory et le DNS :

Validation de l'authentification : Intégration de postes clients au domaine et validation de l'application des GPO.
Simulation de panne : Arrêt volontaire du service DNS sur DC01.
Résultat observé : Impossibilité d'ouvrir une session de domaine et échec de la localisation des contrôleurs de domaine par les clients, confirmant l'importance vitale du service DNS auto-référencé dans l'architecture AD DS.


📸 5. Captures d'Écran de Validation (Livrables)
Copier le tableau


Description de la preuve
Image



Éditeur de réseau virtuel VMware (VMnet1 & VMnet2)
![VMware Network](img/vmware-net.png)


Configuration ipconfig /all sur DC01 (IP statique & DNS)
![ipconfig DC01](img/dc01-ipconfig.png)


Interface de gestion Web pfSense (https://10.10.20.1)
![pfSense Dashboard](img/pfsense-dashboard.png)
