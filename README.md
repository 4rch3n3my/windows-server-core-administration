# windows-server-core-administration

> Déploiement et administration complète d'un Windows Server 2022 en mode Core sur Proxmox VE, sans interface graphique. Administration 100% PowerShell/CLI, de l'installation à l'intégration cloud Microsoft 365 / Azure AD.

---

## Objectif

Simuler un environnement Windows Server de production réaliste pour monter en compétences sur les sujets demandés en entreprise :
- Administration système Windows Server sans GUI
- Active Directory, GPO, gestion des identités
- Sécurité SI et conformité
- Sauvegardes et supervision
- Intégration Microsoft 365 / Azure AD / Intune

---

## Environnement

| Composant | Détail |
|---|---|
| Hyperviseur | Proxmox VE 8.x |
| OS | Windows Server 2022 Standard Core |
| Domaine | lab.local |
| Outils | PowerShell, diskpart, sconfig |

---

## Structure du projet

| Étape | Contenu | Statut |
|---|---|---|
| [Étape 1 — Déploiement](./etape1-deploiement/) | VM Proxmox, drivers VirtIO, AD DS, GPO, sauvegardes | ✅ Terminé |
| [Étape 2 — AD Avancé](./etape2-ad-avance/) | RODC, délégation de contrôle, jonction de poste, GPO avancées | 🔄 En cours |
| [Étape 3 — Sécurité SI](./etape3-securite/) | Firewall Windows, audit, supervision, conformité ANSSI | ⏳ À venir |
| [Étape 4 — Azure AD & M365](./etape4-azure-ad/) | Azure AD Connect, Intune, Exchange Online, Teams | ⏳ À venir |

---

## Auteur

**Thomas Pichot** — Administrateur Systèmes & Réseaux  
[LinkedIn](https://www.linkedin.com/in/p1ch0tth0m45) | Toulouse, France
