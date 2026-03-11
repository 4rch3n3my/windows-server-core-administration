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
# Permet de configurer les paramètres essentiels sans interface graphique
sconfig
```

Via sconfig, l'IP statique et le DNS ont été configurés :
- IP : `192.168.1.60`
- DNS : `1.1.1.1`

```powershell
# Renommer le serveur
# -NewName : nom souhaité pour le serveur
# -Restart : redémarre automatiquement pour appliquer le changement
Rename-Computer -NewName "SRV-CORE" -Restart
PS C:\Users\Administrateur> hostname
SRV-CORE

```

```powershell
# Installer le module de gestion des mises à jour Windows
# -Force : installe sans confirmation même si déjà présent
Install-Module PSWindowsUpdate -Force

# Lancer et installer toutes les mises à jour disponibles
# -AcceptAll : accepte toutes les mises à jour sans confirmation manuelle
# -Install : installe directement sans étape intermédiaire
# -AutoReboot : redémarre automatiquement si une mise à jour l'exige
Get-WindowsUpdate -AcceptAll -Install -AutoReboot
```


### 5. Configuration réseau via PowerShell

```powershell
# Lister les interfaces réseau disponibles
PS C:\Users\Administrateur> Get-NetAdapter

Name                      InterfaceDescription                    ifIndex Status       MacAddress
----                      --------------------                    ------- ------       ---------- 
Ethernet                  Intel(R) PRO/1000 MT Network Connection       4 Up           BC-24-1... 



# Configurer une IP statique
# -InterfaceAlias : nom de l'interface retourné par Get-NetAdapter
# -IPAddress : adresse IP à assigner
# -PrefixLength : masque réseau en notation CIDR (24 = 255.255.255.0)
# -DefaultGateway : passerelle par défaut
New-NetIPAddress -InterfaceAlias "Ethernet" `
  -IPAddress 192.168.1.60 `
  -PrefixLength 24 `
  -DefaultGateway 192.168.1.1

# Configurer le serveur DNS
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" `
  -ServerAddresses 192.168.1.1

```

### 6. Installation du rôle AD DS

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
PS C:\Users\Administrateur> Get-Service ADWS, DNS, Netlogon, NTDS | Select Name, Status

Name      Status
----      ------
ADWS     Running
DNS      Running
Netlogon Running
NTDS     Running


# Vérifier les informations du domaine
PS C:\Users\Administrateur> Get-ADDomain


AllowedDNSSuffixes                 : {}
ChildDomains                       : {}
ComputersContainer                 : CN=Computers,DC=homelab,DC=local
DeletedObjectsContainer            : CN=Deleted Objects,DC=homelab,DC=local
DistinguishedName                  : DC=homelab,DC=local
DNSRoot                            : homelab.local
DomainControllersContainer         : OU=Domain Controllers,DC=homelab,DC=local
DomainMode                         : Windows2016Domain
DomainSID                          : S-1-5-21-20253185-3697107975-3713613610
ForeignSecurityPrincipalsContainer : CN=ForeignSecurityPrincipals,DC=homelab,DC=local
Forest                             : homelab.local
InfrastructureMaster               : SRV-CORE.homelab.local
LastLogonReplicationInterval       :
LinkedGroupPolicyObjects           : {CN={31B2F340-016D-11D2-945F-00C04FB984F9},CN=Policies,CN=Sy 
                                     stem,DC=homelab,DC=local}
LostAndFoundContainer              : CN=LostAndFound,DC=homelab,DC=local
ManagedBy                          :
Name                               : homelab
NetBIOSName                        : LAB
ObjectClass                        : domainDNS
ObjectGUID                         : 6c5b75a0-95d3-4398-9d81-a0f2f505396b
ParentDomain                       :
PDCEmulator                        : SRV-CORE.homelab.local
PublicKeyRequiredPasswordRolling   : True
QuotasContainer                    : CN=NTDS Quotas,DC=homelab,DC=local
ReadOnlyReplicaDirectoryServers    : {}
ReplicaDirectoryServers            : {SRV-CORE.homelab.local}
RIDMaster                          : SRV-CORE.homelab.local
SubordinateReferences              : {DC=ForestDnsZones,DC=homelab,DC=local,
                                     DC=DomainDnsZones,DC=homelab,DC=local,
                                     CN=Configuration,DC=homelab,DC=local}
SystemsContainer                   : CN=System,DC=homelab,DC=local
UsersContainer                     : CN=Users,DC=homelab,DC=local

```

### 7. Structure Active Directory — OUs, Utilisateurs, Groupes

```powershell
# Créer les unités d'organisation (OUs)
# -Name : nom de l'OU
# -Path : emplacement dans l'arborescence AD
New-ADOrganizationalUnit -Name "Utilisateurs" -Path "DC=lab,DC=local"
New-ADOrganizationalUnit -Name "Informatique" -Path "DC=lab,DC=local"
New-ADOrganizationalUnit -Name "Direction"    -Path "DC=lab,DC=local"
New-ADOrganizationalUnit -Name "Serveurs"     -Path "DC=lab,DC=local"

# Vérifier les OUs créées
PS C:\Users\Administrateur> Get-ADOrganizationalUnit -Filter * | Select Name, DistinguishedName

Name               DistinguishedName                        
----               -----------------
Domain Controllers OU=Domain Controllers,DC=homelab,DC=local
Utilisateurs       OU=Utilisateurs,DC=homelab,DC=local
Informatique       OU=Informatique,DC=homelab,DC=local
Direction          OU=Direction,DC=homelab,DC=local
Serveurs           OU=Serveurs,DC=homelab,DC=local

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
PS C:\Users\Administrateur> Get-ADUser -Filter * | Select Name, SamAccountName, Enabled

Name           SamAccountName Enabled
----           -------------- -------
Administrateur Administrateur    True
Invité         Invité           False
krbtgt         krbtgt           False
Alice MARTIN   amartin           True
Admin Infra    ainfra            True
Bob Dupont     bdupont           True

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
PS C:\Users\Administrateur> Get-ADGroupMember -Identity "GRP-Utilisateurs" | Select Name

Name        
----
Alice MARTIN
Bob Dupont


PS C:\Users\Administrateur> Get-ADGroupMember -Identity "GRP-Informatique" | Select Name

Name       
----
Admin Infra


```

> 💡 Pour déplacer un objet AD mal placé sans le supprimer :
> ```powershell
> Move-ADObject -Identity "CN=GRP-Informatique,OU=Utilisateurs,DC=lab,DC=local" `
>   -TargetPath "OU=Informatique,DC=lab,DC=local"
> ```

### 8. Configuration des GPO

```powershell
# Installer la console de gestion des GPO
# sans ce rôle les cmdlets New-GPO et New-GPLink n'existent pas
Install-WindowsFeature -Name GPMC

# Créer une GPO vide — elle n'est pas encore appliquée à quoi que ce soit
New-GPO -Name "Securite-Postes" -Comment "Politique sécurité postes de travail"

# Lier la GPO à une OU pour l'activer
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

```
### 8. Configuration du ssh pour l'administration distante

```powershell
## Prérequis

- Windows Server Core 2022 installé et à jour
- **⚠️ Redémarrer le serveur après les mises à jour avant de commencer**

---

## 1. Installation

```powershell
Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0
```

---

## 2. Démarrage du service

```powershell
Start-Service sshd
```

---

## 3. Activation au démarrage

```powershell
Set-Service -Name sshd -StartupType Automatic
```

---

## 4. Connexion

```bash
┌──(arch3n3my㉿kali)-[~]
└─$ ssh administrateur@192.168.1.60                                   
The authenticity of host '192.168.1.60 (192.168.1.60)' can't be established.
ED25519 key fingerprint is: SHA256:WTKo/Ik80zUjwBCvlAmuyISbzmAvM4sOZoPnav3b2Fg
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.1.60' (ED25519) to the list of known hosts.
** WARNING: connection is not using a post-quantum key exchange algorithm.
** This session may be vulnerable to "store now, decrypt later" attacks.
** The server may need to be upgraded. See https://openssh.com/pq.html
administrateur@192.168.1.60's password: 
Windows PowerShell
Copyright (C) Microsoft Corporation. Tous droits réservés.

Installez la dernière version de PowerShell pour de nouvelles fonctionnalités et améliorations ! https://aka.ms/PSWindowsttps://aka.ms/PSWindows

PS C:\Users\Administrateur>  

```


```


### 9. Politique de mot de passe et verrouillage de compte

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
PS C:\Users\Administrateur> Get-ADDefaultDomainPasswordPolicy -Identity "homelab.local" | Format-List *


ComplexityEnabled           : True
DistinguishedName           : DC=homelab,DC=local
LockoutDuration             : 00:30:00
LockoutObservationWindow    : 00:30:00
LockoutThreshold            : 5
MaxPasswordAge              : 90.00:00:00
MinPasswordAge              : 1.00:00:00
MinPasswordLength           : 12
objectClass                 : {domainDNS}
objectGuid                  : 6c5b75a0-95d3-4398-9d81-a0f2f505396b
PasswordHistoryCount        : 10
ReversibleEncryptionEnabled : False
PropertyNames               : {ComplexityEnabled, DistinguishedName, LockoutDuration,
                              LockoutObservationWindow...}
AddedProperties             : {}
RemovedProperties           : {}
ModifiedProperties          : {}
PropertyCount               : 12

```

### 10. Configuration des sauvegardes Windows Server

```powershell
# Installer la fonctionnalité de sauvegarde
Install-WindowsFeature -Name Windows-Server-Backup
```

> ⚠️ Windows Server Backup refuse de sauvegarder sur le même disque que la source.
> Un second disque dédié est ajouté depuis proxmox. Dans ce lab : disque VirtIO Block de 10 Go.

```cmd
# Initialisation du disque de sauvegarde via diskpart
diskpart
select disk 1
attributes disk clear readonly
online disk
convert gpt
create partition primary
format fs=ntfs label="Backups" quick
assign letter=F
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
PS C:\Users\Administrateur> Get-WBSummary  


NextBackupTime                  : 11/03/2026 22:00:00
NumberOfVersions                : 1
LastSuccessfulBackupTime        : 10/03/2026 20:16:00
LastSuccessfulBackupTargetPath  : \\?\Volume{efb0c152-7d1a-4a8c-88f9-f27c47a58bf3}
LastSuccessfulBackupTargetLabel : WIN-R 10/03/2026 20:15:03 Disk01
LastBackupTime                  : 11/03/2026 01:12:26
LastBackupTarget                : WIN-R 10/03/2026 20:15:03 Disk01
DetailedMessage                 : Un périphérique qui n’existe pas a été spécifié
LastBackupResultHR              : 2155348165
LastBackupResultDetailedHR      : 2147942833
CurrentOperationStatus          : NoOperationInProgress
```

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


---

## Auteur

**Thomas Pichot** — Administrateur Systèmes & Réseaux  
[LinkedIn](https://www.linkedin.com/in/p1ch0tth0m45) | Toulouse, France
