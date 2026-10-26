# Étape 2 — Cibles et annuaire Active Directory

> Prérequis : l'Étape 1 est terminée et sa checklist passe (pfSense route entre
> `lab-corp` 10.10.10.0/24 et `lab-red` 10.10.20.0/24).
> Lab **isolé** uniquement. Les faiblesses posées ici sont **volontaires et
> pédagogiques** : elles n'ont de sens que dans ce lab coupé du monde.

---

## 1. Objectif de l'étape

- Déployer un **contrôleur de domaine Windows Server (AD DS)** dans `lab-corp`.
- Déployer la **Kali attaquante** dans `lab-red`.
- Créer un domaine, des comptes et quelques **faiblesses réalistes** qui serviront
  de support aux exercices rouge → bleu de l'étape 3.

Budget RAM (rappel 8 Go) : pfSense (~1 Go) + DC (2 Go) + Kali (2 Go) ≈ 5 Go en
simultané. Éteins les VM dont tu ne te sers pas.

---

## 2. VM Windows Server (le futur DC)

### 2.1 ISO
- Windows Server 2022 **Evaluation** (180 jours, gratuit, site Microsoft) — ou 2019.
- Choisis l'édition **Standard (Desktop Experience)** pour avoir l'interface
  graphique au début.

### 2.2 Créer la VM
- **Nouvelle** → Nom : `DC01` · Type : **Microsoft Windows** · Version : **Windows 2022 (64-bit)**
- RAM : **2048 Mo** (minimum viable 1536 Mo)
- Disque : **50 Go** VDI dynamique
- 2 vCPU si possible
- **Réseau — Adaptateur 1** : Réseau interne, nom **`lab-corp`** (une seule carte)

### 2.3 Installer et configurer l'IP
- Installe Windows, mot de passe administrateur **fort**.
- Panneau réseau → IPv4 **statique** :
  - Adresse : `10.10.10.10` / masque `255.255.255.0`
  - Passerelle : `10.10.10.1`
  - DNS préféré : `127.0.0.1` (le DC sera son propre DNS)
- Renomme la machine en `DC01`, redémarre.
- Test : `ping 10.10.10.1` (pfSense) doit répondre.

### 2.4 Promouvoir en contrôleur de domaine (AD DS)
- **Gestionnaire de serveur → Ajouter des rôles → AD DS**, installe le rôle.
- Puis **Promouvoir ce serveur en contrôleur de domaine** :
  - **Ajouter une nouvelle forêt**
  - Nom de domaine racine : **`sodeco.lab`** (le suffixe `.lab` évite tout conflit
    avec un vrai domaine ; ne jamais utiliser un nom de domaine réel)
  - Mot de passe **DSRM** (mode restauration) — note-le
  - Laisse DNS s'installer automatiquement, termine, redémarre.
- Après reboot, connecte-toi en `SODECO\Administrateur`.

---

## 3. Peupler l'annuaire + faiblesses réalistes (pédagogiques)

Ouvre **PowerShell en administrateur** sur `DC01`. Ces commandes créent une
structure d'OU, des comptes, et des faiblesses classiques qu'on exploitera **et**
qu'on détectera ensuite.

### 3.1 Structure et comptes de base
```powershell
# OU métier
New-ADOrganizationalUnit -Name "SODECO-Users"  -Path "DC=sodeco,DC=lab"
New-ADOrganizationalUnit -Name "SODECO-Admins" -Path "DC=sodeco,DC=lab"

# Quelques utilisateurs standard
$pw = ConvertTo-SecureString "Ete2026!Soleil" -AsPlainText -Force
New-ADUser -Name "Jean Koffi"  -SamAccountName "jkoffi"  -AccountPassword $pw -Enabled $true -Path "OU=SODECO-Users,DC=sodeco,DC=lab"
New-ADUser -Name "Awa Traore"  -SamAccountName "atraore" -AccountPassword $pw -Enabled $true -Path "OU=SODECO-Users,DC=sodeco,DC=lab"
```

### 3.2 Faiblesses volontaires (chacune = un exercice ciblé)
```powershell
# (A) Compte de service avec SPN -> cible de Kerberoasting (ATT&CK T1558.003)
$svcpw = ConvertTo-SecureString "Service#2021" -AsPlainText -Force
New-ADUser -Name "svc_sql" -SamAccountName "svc_sql" -AccountPassword $svcpw -Enabled $true -Path "OU=SODECO-Users,DC=sodeco,DC=lab"
Set-ADUser svc_sql -ServicePrincipalNames @{Add="MSSQLSvc/db01.sodeco.lab:1433"}
Set-ADUser svc_sql -PasswordNeverExpires $true

# (B) Compte sans pré-auth Kerberos -> AS-REP Roasting (ATT&CK T1558.004)
New-ADUser -Name "Paul Mensah" -SamAccountName "pmensah" -AccountPassword $pw -Enabled $true -Path "OU=SODECO-Users,DC=sodeco,DC=lab"
Set-ADUser pmensah -DoesNotRequirePreAuth $true

# (C) Droits excessifs : un utilisateur "normal" mis dans Admins du domaine
Add-ADGroupMember -Identity "Domain Admins" -Members atraore
```

### 3.3 Un partage SMB trop permissif
```powershell
New-Item -Path "C:\Partage-RH" -ItemType Directory
New-SmbShare -Name "RH" -Path "C:\Partage-RH" -FullAccess "Everyone"
"Fiches de paie - CONFIDENTIEL" | Out-File C:\Partage-RH\paie.txt
```

> Chaque faiblesse ci-dessus a une **détection** et une **remédiation** associées
> qu'on construira aux étapes 3 et 4 (ex. : désactiver `DoesNotRequirePreAuth`,
> mots de passe de service longs + gMSA, retrait des droits excessifs, ACL de
> partage au moindre privilège).

---

## 4. Kali attaquante (dans `lab-red`)

### 4.1 Créer la VM
- ISO Kali Linux (image Installer 64-bit, site officiel `kali.org`).
- **Nouvelle** → Nom : `kali-red` · Type : **Linux** · Version : **Debian (64-bit)**
- RAM : **2048 Mo** · Disque : **25 Go** dynamique
- **Réseau — Adaptateur 1** : Réseau interne, nom **`lab-red`**

### 4.2 IP statique
Édite `/etc/network/interfaces` (ou via NetworkManager) :
- Adresse : `10.10.20.50/24`
- Passerelle : `10.10.20.1`
- DNS : `10.10.10.10` (le DC, pour résoudre `sodeco.lab`)

### 4.3 Vérifier la chaîne
```bash
ping 10.10.20.1          # pfSense (passerelle RED)
ping 10.10.10.10         # le DC, à travers pfSense (routage OK)
nslookup sodeco.lab 10.10.10.10   # résolution du domaine
```

---

## 5. Vérifications avant l'étape 3

- [ ] `DC01` est promu, domaine `sodeco.lab` opérationnel, DNS répond.
- [ ] Les comptes `jkoffi`, `atraore`, `svc_sql`, `pmensah` existent.
- [ ] Le SPN de `svc_sql` est visible : `setspn -L svc_sql`.
- [ ] Le partage `\\10.10.10.10\RH` est accessible.
- [ ] Depuis la Kali : ping du DC OK + `nslookup sodeco.lab` OK.
- [ ] Les trois VM (pfSense, DC01, kali-red) tiennent en RAM ensemble.

---

## 6. Lien attaque ↔ détection (ce qu'on branchera ensuite)

| Faiblesse posée        | Technique ATT&CK     | Détection (étape 3/4)                         | Remédiation (étape 4)                     |
|------------------------|----------------------|-----------------------------------------------|-------------------------------------------|
| SPN sur `svc_sql`      | T1558.003 Kerberoasting | Event 4769 (demande de ticket TGS chiffré RC4) | Mot de passe long / gMSA, AES only        |
| `pmensah` sans pré-auth| T1558.004 AS-REP roasting | Event 4768 sans pré-auth                     | Retirer `DoesNotRequirePreAuth`           |
| `atraore` Domain Admin | T1078 comptes valides / escalade | Audit des membres de groupes privilégiés | Tiering AD, retrait des droits            |
| Partage `RH` Everyone  | T1135 découverte de partages | Event 5140/5145 accès partage              | ACL au moindre privilège                  |

> On installera **Sysmon** sur le DC à l'étape 3 pour enrichir ces traces et
> commencer à écrire des règles de détection.

---

## Prochaine étape

**Étape 3** : premier cycle rouge → bleu de bout en bout, mappé sur la kill chain
(reconnaissance → accès initial → mouvement latéral), avec pour **chaque**
technique sa trace côté défense et sa contre-mesure.
