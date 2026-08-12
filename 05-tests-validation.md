# 5. Tests et validation

> Les sources déclarent les contrôles ci-dessous validés. Les rejouer dans le lab cible ; ce dépôt ne constitue pas une preuve indépendante d’exécution.

## Tests réseau et DNS

```powershell
ipconfig /all
ping 10.10.20.10
ping 10.10.10.20
nslookup -type=srv _ldap._tcp.dc._msdcs.entreprise.lab 10.10.20.10
```

Attendus : IP, passerelle et DNS conformes ; connectivité selon les règles pfSense ; enregistrements SRV AD retournant les DC.

## Contrôles AD et réplication

```powershell
Get-ADDomainController -Filter * |
  Select-Object Name,IPv4Address,IsGlobalCatalog,OperatingSystem
repadmin /replsummary
repadmin /showrepl DC01
repadmin /showrepl RODC01
repadmin /syncall /AeP
Get-ADReplicationPartnerMetadata -Target RODC01
```

Attendus : DC01 et RODC01 visibles, `IsGlobalCatalog` vrai pour RODC01, partenaires sans échec et réplication depuis DC01.

## Client, GPO et objectif de délai

```powershell
Add-Computer -DomainName "entreprise.lab" -Credential (Get-Credential ENTREPRISE\Administrator) -Restart
gpupdate /force
gpresult /r
```

La GPO **Corporate Wallpaper** peut utiliser `\\DC01\NETLOGON\wallpaper\corp.jpg`. Contrôler le poste dans l’OU attendue, le lien GPO, SYSVOL/NETLOGON et mesurer une application en moins de 90 secondes. Répéter depuis OPT1.

Exemple de commandes de création de la GPO rapporté par les sources :

```powershell
New-GPO -Name "Corporate Wallpaper" | `
  Set-GPPrefRegistryValue -Context Computer `
  -Key "HKLM\Software\Microsoft\Windows\CurrentVersion\Policies\System" `
  -ValueName "Wallpaper" -Type String `
  -Value "\\DC01\NETLOGON\wallpaper\corp.jpg" -Action Update
Set-GPRegistryValue -Name "Corporate Wallpaper" `
  -Key "HKCU\Software\Microsoft\Windows\CurrentVersion\Policies\System" `
  -ValueName "Wallpaper" -Type String `
  -Value "\\DC01\NETLOGON\wallpaper\corp.jpg"
```

## Script de validation de bout en bout

```powershell
$dc = "DC01"; $rodc = "RODC01"; $dns = "10.10.20.10"
Write-Host "=== Domain Controllers ===" -ForegroundColor Cyan
Get-ADDomainController -Filter * | Select Name,IPv4Address,IsGlobalCatalog,OperatingSystem | Format-Table -AutoSize
Write-Host "=== DNS SRV records ===" -ForegroundColor Cyan
nslookup -type=srv _ldap._tcp.dc._msdcs.entreprise.lab $dns
Write-Host "=== Replication summary ===" -ForegroundColor Cyan
repadmin /replsummary; repadmin /showrepl $dc; repadmin /showrepl $rodc
Write-Host "=== GPO application on a client ===" -ForegroundColor Cyan
gpresult /r
```

## Mode dégradé : DNS arrêté

Arrêter contrôlé le service DNS sur DC01 : les sources rapportent que l’ouverture de session de domaine échoue immédiatement, puis revient après redémarrage du DNS. Cette observation confirme la dépendance AD ↔ DNS et l’importance des SRV `_ldap._tcp.dc._msdcs`.

## Dépannage prioritaire

1. Authentification : DNS préféré de chaque DC, SRV AD, heure système.
2. Réplication : routes/règles pfSense, noms, ports RPC et heure.
3. GPO : jointure, lien, SYSVOL/NETLOGON et `gpresult /r`.
4. Authentification via RODC : PRP, groupe autorisé et cache pré-rempli.

## Matrice synthétique

| Test | Résultat rapporté / attendu |
|---|---|
| VMware | VMnet1/VMnet2 host-only et isolés |
| pfSense | DHCP LAN/OPT1 actif, WAN NAT |
| AD/DNS | DC promus, SRV dynamiques résolus |
| Réplication | `repadmin` sans erreur |
| PRP | `msDS-RevealOnDemandGroup` configuré |
| Connectivité | pings et `nslookup` depuis OPT1 |
| GPO | application avec cible `< 90 s` |
| DNS arrêté | ouverture de session échoue, puis revient après redémarrage |
