# 3. Configuration du RODC

## Paramètres

`RODC01` est un Windows Server 2019 placé dans `SEG_USR` (`10.10.10.0/24`). IP `10.10.10.20/24`, passerelle `10.10.10.1`, DNS préféré `10.10.10.20` (self-reference). Il réplique la base AD depuis DC01 en lecture seule, sert les utilisateurs de filiale et évite une copie inscriptible dans un site moins sécurisé.

## Jointure et promotion

Exécuter sur RODC01, après avoir rendu DC01 joignable et résolvable :

```powershell
Rename-Computer -NewName "RODC01" -Restart
Add-Computer -DomainName "entreprise.lab" -Credential (Get-Credential ENTREPRISE\Administrator) -Restart
Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools
Install-ADDSDomainController `
  -DomainName "entreprise.lab" `
  -Credential (Get-Credential ENTREPRISE\Administrator) `
  -SiteName "Default-First-Site-Name" `
  -InstallDns:$true `
  -ReadOnlyReplica:$true `
  -Force:$true
```

Une commande source équivalente omet `-Credential` et `-SiteName` ; les paramètres explicites ci-dessus sont conservés car ils rendent la procédure reproductible.

## Politique de réplication des mots de passe (PRP)

Par défaut, aucun mot de passe n’est mis en cache sur le RODC. Autoriser uniquement un groupe d’utilisateurs de la filiale ; refuser les administrateurs du domaine, comptes sensibles et comptes de service critiques. Le groupe indiqué par les sources est `Allowed RODC Password Replication Group`.

```powershell
Get-ADDomainController -Identity RODC01 | Set-ADObject -Add @{
  "msDS-RevealOnDemandGroup" = @(
    "CN=Allowed RODC Password Replication Group,CN=Groups,DC=entreprise,DC=lab"
  )
}
```

Pré-remplir le cache uniquement si nécessaire et administrer le RODC avec un compte local/dédié `RODC Administrator` afin de séparer les rôles.

## Contrôles

```powershell
Get-ADDomainController -Identity RODC01 |
  Format-List Name,IPv4Address,IsGlobalCatalog,OperatingSystem
repadmin /replsummary
repadmin /showrepl RODC01
Get-ADReplicationPartnerMetadata -Target RODC01
```

Le résultat attendu est un RODC identifié comme Global Catalog, une réplication sans erreur et aucune capacité d’écriture locale de l’annuaire.
