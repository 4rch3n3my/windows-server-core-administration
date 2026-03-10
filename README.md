# Windows Server Core — Déploiement & Administration PowerShell

> Déploiement d'un Windows Server en mode Core sur Proxmox VE, administré exclusivement via PowerShell et outils CLI. Projet réalisé dans le cadre d'une montée en compétences pour un poste d'Administrateur Systèmes et Réseaux.

---

## Contexte

L'objectif de ce projet est de simuler un environnement de production réaliste :
- Pas d'interface graphique → administration 100% CLI/PowerShell
- Déploiement sur hyperviseur Proxmox VE
- Configuration Active Directory, GPO, sauvegardes et supervision via scripts

---

## Environnement technique

| Composant | Détail |
|---|---|
| Hyperviseur | Proxmox VE 8.x |
| OS | Windows Server 2022 Standard Core |
| RAM allouée | 4 Go |
| Stockage système | 40 Go (VirtIO SCSI) |
| Stockage sauvegarde | 10 Go (VirtIO Block) |
| Réseau | VirtIO, IP statique |
| Drivers | VirtIO Win ISO |

---

## Étapes du projet

### 1. Création de la VM sur Proxmox

- Création VM avec firmware UEFI + TPM 2.0
- Attachement de l'ISO Windows Server et de l'ISO VirtIO drivers en IDE
- Disque système en VirtIO SCSI, disque sauvegarde en VirtIO Block
- Réseau en VirtIO

> ⚠️ Les lecteurs CD/DVD doivent être en IDE. Les disques durs en VirtIO SCSI ou VirtIO Block.

```bash
# Vérification de la VM depuis le shell Proxmox
qm status <vmid>
```

### 2. Installation en mode Core

Lors de l'installation, sélectionner :
```
Windows Server 2022 Standard (installation minimale)
```

Résultat : invite de commande uniquement, sans interface graphique.

### 3. Installation des drivers VirtIO

Après installation, monter l'ISO VirtIO comme second lecteur CD/DVD puis installer les drivers :

```cmd
# Driver réseau — indispensable pour la connectivité
pnputil /add-driver D:\NetKVM\2k22\amd64\*.inf /install

# Driver stockage
pnputil /add-driver D:\viostor\2k22\amd64\*.inf /install

# Driver ballon mémoire — optimisation RAM dynamique
pnputil /add-driver D:\Balloon\2k22\amd64\*.inf /install

# Driver série virtuel — communications Proxmox VM
pnputil /add-driver D:\vioserial\2k22\amd64\*.inf /install

# Installer tous les drivers restants en une seule commande PowerShell
Get-ChildItem D:\ -Recurse -Filter "*.inf" | ForEach-Object { pnputil /add-driver $_.FullName /install }
```

```powershell
# Vérifier les périphériques encore en erreur après installation
Get-PnpDevice | Where-Object {$_.Status -eq "Error"} | Select FriendlyName, InstanceId
```

### 4. Configuration initiale via sconfig

```cmd
# Outil de configuration de base en mode Core
sconfig
```

Éléments configurés :
- Nom du serveur : SRV-CORE-01
- Adresse IP statique
- Activation bureau à distance (RDP)
- Mises à jour Windows

### 5. Passage sur PowerShell

```cmd
powershell
```

### 6. Configuration réseau via PowerShell

```powershell
# Lister les interfaces réseau disponibles
Get-NetAdapter

# Configurer une IP statique
# -InterfaceAlias : nom de l'interface retourné par Get-NetAdapter
# -IPAddress : adresse IP à assigner
# -PrefixLength : masque réseau en notation CIDR (24 = 255.255.255.0)
# -DefaultGateway : passerelle par défaut
New-NetIPAddress -InterfaceAlias "Ethernet" `
  -IPAddress 192.168.1.10 `
  -PrefixLength 24 `
  -DefaultGateway 192.168.1.1

# Configurer le serveur DNS
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" `
  -ServerAddresses 192.168.1.1

# Renommer le serveur et redémarrer
Rename-Computer -NewName "SRV-CORE-01" -Restart
```

### 7. Installation du rôle AD DS

```powershell
# Installer le rôle Active Directory Domain Services
# -IncludeManagementTools : installe aussi les outils d'administration PowerShell AD
Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools

# Promouvoir le serveur en contrôleur de domaine
# -DomainName : nom FQDN du domaine à créer
# -DomainNetbiosName : nom NetBIOS court du domaine
# -ForestMode / -DomainMode : niveau fonctionnel (WinThreshold = Windows Server 2016+)
# -InstallDns : installe le rôle DNS en même temps
# -Force : pas de confirmation interactive, redémarre automatiquement
Install-ADDSForest `
  -DomainName "lab.local" `
  -DomainNetbiosName "LAB" `
  -ForestMode "WinThreshold" `
  -DomainMode "WinThreshold" `
  -InstallDns `
  -Force
```

```powershell
# Vérifier que les services AD sont bien démarrés après reboot
Get-Service ADWS, DNS, Netlogon, NTDS | Select Name, Status

# Vérifier les informations du domaine
Get-ADDomain
```

### 8. Structure Active Directory — OUs, Utilisateurs, Groupes

```powershell
# Créer les unités d'organisation (OUs)
# -Name : nom de l'OU
# -Path : emplacement dans l'arborescence AD
New-ADOrganizationalUnit -Name "Utilisateurs" -Path "DC=lab,DC=local"
New-ADOrganizationalUnit -Name "Informatique" -Path "DC=lab,DC=local"
New-ADOrganizationalUnit -Name "Direction"    -Path "DC=lab,DC=local"
New-ADOrganizationalUnit -Name "Serveurs"     -Path "DC=lab,DC=local"

# Vérifier les OUs créées
Get-ADOrganizationalUnit -Filter * | Select Name, DistinguishedName
```

```powershell
# Créer des utilisateurs de test
# -SamAccountName : identifiant de connexion
# -Path : OU de destination
# -Enabled : compte actif dès la création
# -AccountPassword : mot de passe initial converti en SecureString
New-ADUser -Name "Alice Martin" `
  -SamAccountName "amartin" `
  -Path "OU=Utilisateurs,DC=lab,DC=local" `
  -Enabled $true `
  -AccountPassword (ConvertTo-SecureString "P@ssw0rd123!" -AsPlainText -Force)

New-ADUser -Name "Bob Dupont" `
  -SamAccountName "bdupont" `
  -Path "OU=Utilisateurs,DC=lab,DC=local" `
  -Enabled $true `
  -AccountPassword (ConvertTo-SecureString "P@ssw0rd123!" -AsPlainText -Force)

New-ADUser -Name "Admin Infra" `
  -SamAccountName "ainfra" `
  -Path "OU=Informatique,DC=lab,DC=local" `
  -Enabled $true `
  -AccountPassword (ConvertTo-SecureString "P@ssw0rd123!" -AsPlainText -Force)

# Vérifier les utilisateurs créés
Get-ADUser -Filter * | Select Name, SamAccountName, Enabled
```

```powershell
# Créer des groupes de sécurité
# -GroupScope Global : visible dans tout le domaine
# -GroupCategory Security : groupe de sécurité (pas de distribution)
New-ADGroup -Name "GRP-Informatique" `
  -GroupScope Global `
  -GroupCategory Security `
  -Path "OU=Informatique,DC=lab,DC=local"

New-ADGroup -Name "GRP-Utilisateurs" `
  -GroupScope Global `
  -GroupCategory Security `
  -Path "OU=Utilisateurs,DC=lab,DC=local"

# Ajouter les utilisateurs dans les groupes
Add-ADGroupMember -Identity "GRP-Utilisateurs" -Members "amartin","bdupont"
Add-ADGroupMember -Identity "GRP-Informatique" -Members "ainfra"

# Vérifier les membres
Get-ADGroupMember -Identity "GRP-Utilisateurs" | Select Name
Get-ADGroupMember -Identity "GRP-Informatique" | Select Name
```

> 💡 Pour déplacer un objet AD mal placé sans le supprimer :
> ```powershell
> Move-ADObject -Identity "CN=GRP-Informatique,OU=Utilisateurs,DC=lab,DC=local" `
>   -TargetPath "OU=Informatique,DC=lab,DC=local"
> ```

### 9. Configuration des GPO

```powershell
# Installer la console de gestion des GPO
# sans ce rôle les cmdlets New-GPO et New-GPLink n'existent pas
Install-WindowsFeature -Name GPMC

# Créer une GPO vide — elle n'est pas encore appliquée à quoi que ce soit
New-GPO -Name "Securite-Postes" -Comment "Politique sécurité postes de travail"

# Lier la GPO à une OU — c'est ce qui l'active réellement
# -Enforced Yes : force l'application même si une GPO en conflit existe
# plus bas dans la hiérarchie — évite qu'un admin local la contourne
New-GPLink -Name "Securite-Postes" `
  -Target "OU=Utilisateurs,DC=lab,DC=local" `
  -Enforced Yes

# Verrouillage automatique après 5 minutes d'inactivité
# -Key : chemin registre correspondant au paramètre
# -Type DWord : valeur numérique entière
# -Value 300 : 300 secondes = 5 minutes
Set-GPRegistryValue -Name "Securite-Postes" `
  -Key "HKLM\Software\Microsoft\Windows\CurrentVersion\Policies\System" `
  -ValueName "InactivityTimeoutSecs" `
  -Type DWord `
  -Value 300

# Désactivation du CMD pour les utilisateurs standards
# -Value 1 : restriction activée (0 = désactivée)
Set-GPRegistryValue -Name "Securite-Postes" `
  -Key "HKCU\Software\Policies\Microsoft\Windows\System" `
  -ValueName "DisableCMD" `
  -Type DWord `
  -Value 1
```

### 10. Politique de mot de passe et verrouillage de compte

```powershell
# Configurer la politique de mot de passe du domaine
# Note : s'applique sur le domaine entier via Default Domain Policy
# -MinPasswordLength 12 : recommandation ANSSI
# -PasswordHistoryCount 10 : mémorise les 10 derniers mots de passe
# -MaxPasswordAge : validité maximale de 90 jours
# -MinPasswordAge : délai minimum avant de pouvoir rechanger
# -ComplexityEnabled : force maj + min + chiffres + caractères spéciaux
Set-ADDefaultDomainPasswordPolicy -Identity "lab.local" `
  -MinPasswordLength 12 `
  -PasswordHistoryCount 10 `
  -MaxPasswordAge 90.00:00:00 `
  -MinPasswordAge 1.00:00:00 `
  -ComplexityEnabled $true

# Configurer le verrouillage de compte
# -LockoutThreshold 5 : verrouille après 5 tentatives échouées
# -LockoutDuration 00:30:00 : verrouillage de 30 minutes puis déverrouillage auto
# -LockoutObservationWindow 00:30:00 : fenêtre de 30 min pour comptabiliser les échecs
Set-ADDefaultDomainPasswordPolicy -Identity "lab.local" `
  -LockoutThreshold 5 `
  -LockoutDuration 00:30:00 `
  -LockoutObservationWindow 00:30:00

# Vérifier la politique appliquée
Get-ADDefaultDomainPasswordPolicy -Identity "lab.local" | Format-List *
```

### 11. Configuration des sauvegardes Windows Server

```powershell
# Installer la fonctionnalité de sauvegarde
Install-WindowsFeature -Name Windows-Server-Backup
```

> ⚠️ Windows Server Backup refuse de sauvegarder sur le même disque que la source.
> Un second disque dédié est obligatoire. Dans ce lab : disque VirtIO Block de 10 Go.

```cmd
# Initialisation du disque de sauvegarde via diskpart
diskpart
select disk 1
attributes disk clear readonly
online disk
convert gpt
create partition primary
format fs=ntfs label="Backups" quick
assign letter=E
exit
```

```powershell
# Créer et configurer la politique de sauvegarde
$policy = New-WBPolicy

# Définir le volume source
$volume = Get-WBVolume -VolumePath "C:"
Add-WBVolume -Policy $policy -Volume $volume

# Définir le disque de destination
# [1] = second disque = disque Backups de 10 Go
$disk = (Get-WBDisk)[1]
$target = New-WBBackupTarget -Disk $disk
Add-WBBackupTarget -Policy $policy -Target $target

# Planifier la sauvegarde quotidienne à 22h00
Set-WBSchedule -Policy $policy -Schedule 22:00

# Enregistrer la politique sur le système
# sans cette étape elle disparaît à la fermeture de la session PowerShell
Set-WBPolicy -Policy $policy -Force

# Lancer une sauvegarde manuelle pour tester
Start-WBBackup -Policy $policy

# Vérifier le résultat — LastBackupResultHR: 0x00000000 = succès
Get-WBSummary
```

---

## Scripts utiles

| Script | Description |
|---|---|
| `export-users.ps1` | Export CSV de tous les utilisateurs AD |
| `inactive-accounts.ps1` | Rapport comptes inactifs depuis 90 jours |
| `reset-password.ps1` | Reset mdp en masse sur une OU |
| `backup-check.ps1` | Vérification état des sauvegardes |

---

## Commandes de diagnostic fréquentes

```powershell
# Vérifier la réplication AD
Get-ADReplicationFailure -Scope Domain

# Vérifier les services critiques
Get-Service ADWS, DNS, Netlogon, NTDS | Select Name, Status

# Table de routage
Get-NetRoute

# Tester la connectivité DNS
Resolve-DnsName lab.local

# Vérifier les GPO appliquées sur le poste
gpresult /r

# Lister les utilisateurs désactivés
Get-ADUser -Filter {Enabled -eq $false} -Properties LastLogonDate | Select Name, LastLogonDate

# Lister les comptes dont le mdp n'a pas changé depuis 90 jours
Get-ADUser -Filter {PasswordLastSet -lt (Get-Date).AddDays(-90)} `
  -Properties PasswordLastSet | Select Name, PasswordLastSet

# Vérifier les périphériques en erreur
Get-PnpDevice | Where-Object {$_.Status -eq "Error"} | Select FriendlyName, InstanceId
```

---

## Références

- [Documentation Microsoft Windows Server Core](https://docs.microsoft.com/fr-fr/windows-server/administration/server-core/server-core-overview)
- [Proxmox VE Documentation](https://pve.proxmox.com/wiki/Main_Page)
- [VirtIO Drivers](https://pve.proxmox.com/wiki/Windows_VirtIO_Drivers)
- [Guide ANSSI Hygiène Informatique](https://www.ssi.gouv.fr/guide/guide-dhygiene-informatique/)

---

## Auteur

**Thomas Pichot** — Administrateur Systèmes & Réseaux  
[LinkedIn](https://www.linkedin.com/in/p1ch0tth0m45) | Toulouse, France
