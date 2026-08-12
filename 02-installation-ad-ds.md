# 2. Installation d’Active Directory Domain Services

## Paramètres DC01

- Nom : `DC01`
- Domaine : `entreprise.lab`
- NetBIOS : `ENTREPRISE`
- IP : `10.10.20.10/24`
- Passerelle : `10.10.20.1`
- DNS préféré : `10.10.20.10` (lui-même, exigence critique AD)
- Rôles : AD DS, DNS intégré à AD et DHCP

## Configuration et promotion

À exécuter dans PowerShell administrateur sur le futur DC :

```powershell
Rename-Computer -NewName "DC01" -Restart
New-NetIPAddress -InterfaceAlias "Ethernet0" -IPAddress 10.10.20.10 -PrefixLength 24 -DefaultGateway 10.10.20.1
Set-DnsClientServerAddress -InterfaceAlias "Ethernet0" -ServerAddresses 10.10.20.10
Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools
Install-ADDSForest `
  -DomainName "entreprise.lab" `
  -DomainNetBIOSName "ENTREPRISE" `
  -InstallDns:$true `
  -CreateDnsDelegation:$false `
  -DatabasePath "C:\Windows\NTDS" `
  -LogPath "C:\Windows\NTDS" `
  -SysvolPath "C:\Windows\SYSVOL" `
  -Force:$true
```

Une variante plus courte présente dans les sources est :

```powershell
Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools
Install-ADDSForest -DomainName "entreprise.lab" -DomainNetBIOSName "ENTREPRISE" -InstallDns:$true -Force
```

Configurer ensuite le rôle DHCP sur DC01 uniquement si le scénario le requiert ; le DHCP des segments de lab reste fourni par pfSense pour éviter les conflits.

## Organisation AD recommandée

```text
entreprise.lab
├── Domain Controllers
│   ├── DC01 (RWDC, DNS, DHCP)
│   └── RODC01 (RODC, DNS, filiale)
├── OU=Serveurs
├── OU=Utilisateurs-Filiale
├── OU=Postes-Clients
└── GPO=Corporate Wallpaper
```

## Contrôle initial

```powershell
Get-ADDomainController -Identity DC01 |
  Format-List Name,IPv4Address,IsGlobalCatalog,OperatingSystem
```

Le DNS doit héberger les zones AD et publier notamment `_ldap._tcp.dc._msdcs.entreprise.lab`.
