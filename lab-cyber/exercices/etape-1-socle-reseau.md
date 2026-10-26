# Étape 1 — Socle réseau segmenté (VirtualBox, lab 8 Go)

> Lab **isolé** à but d'entraînement. Aucun pont (Bridge) vers ton réseau réel.
> À la fin de cette étape : un pare-feu pfSense route et filtre le trafic entre
> deux segments isolés, prêts à accueillir le DC et la Kali à l'étape 2.

---

## 1. Objectif de l'étape

Construire la fondation : un routeur/pare-feu **pfSense** qui sépare deux réseaux
internes VirtualBox.

- `lab-corp` → futur réseau de l'entreprise (le contrôleur de domaine AD)
- `lab-red` → réseau de l'attaquant (la Kali)

C'est volontairement un réseau « à plat » au départ (l'attaquant peut atteindre
l'entreprise) : c'est le scénario réaliste qu'on **durcira** à l'étape 4 pour
mesurer l'apport de la segmentation.

---

## 2. Topologie logique

```
                 Internet (updates only)
                        │
                   [ NAT VBox ]
                        │ WAN (em0)
                  ┌───────────┐
                  │  pfSense  │  routeur / pare-feu
                  └─────┬─────┘
            LAN (em1)   │   OPT1 (em2)
        ┌───────────────┤   ├────────────────┐
        │  lab-corp     │   │   lab-red       │
        │ 10.10.10.0/24 │   │ 10.10.20.0/24   │
        │               │   │                 │
     [ DC Windows ]        [ Kali attaquant ]
      10.10.10.10            10.10.20.50
```

---

## 3. Plan d'adressage

| Segment   | Réseau VirtualBox        | Sous-réseau       | Passerelle (pfSense) |
|-----------|--------------------------|-------------------|----------------------|
| WAN       | NAT                      | fourni par VBox   | automatique          |
| lab-corp  | Réseau interne `lab-corp`| `10.10.10.0/24`   | `10.10.10.1`         |
| lab-red   | Réseau interne `lab-red` | `10.10.20.0/24`   | `10.10.20.1`         |

Hôtes prévus (étape 2) :

| Machine        | Segment   | IP (statique) | Passerelle    | DNS           |
|----------------|-----------|---------------|---------------|---------------|
| DC Windows Srv | lab-corp  | `10.10.10.10` | `10.10.10.1`  | `127.0.0.1` * |
| Kali           | lab-red   | `10.10.20.50` | `10.10.20.1`  | `10.10.10.10`**|

\* Le DC est son propre serveur DNS une fois AD installé.
\*\* La Kali pointe sur le DNS du DC pour résoudre le futur domaine.

---

## 4. Créer les réseaux internes dans VirtualBox

Dans VirtualBox, un « Réseau interne » (intnet) isole le trafic : seules les VM
qui portent **le même nom de réseau interne** se voient, et rien ne sort vers
l'hôte ou le vrai réseau. Pas besoin de les créer à l'avance — ils existent dès
qu'une VM les référence. On les nommera donc directement dans la config pfSense :
`lab-corp` et `lab-red`.

> Rappel : on n'utilise **jamais** le mode « Accès par pont (Bridge) » pour les
> machines du lab. Seul pfSense a un accès NAT, en sortie uniquement.

---

## 5. Créer et configurer la VM pfSense

### 5.1 Télécharger
- Récupère l'ISO pfSense CE (Community Edition) sur le site officiel
  (`pfsense.org`), architecture AMD64, support « CD Image (ISO) Installer ».
- Décompresse le `.gz` si besoin pour obtenir le `.iso`.

### 5.2 Créer la VM
- **Nouvelle** → Nom : `pfSense-lab` · Type : **BSD** · Version : **FreeBSD (64-bit)**
- RAM : **1024 Mo** (suffisant ; on pourra descendre à 512 Mo plus tard)
- Disque dur : créer, **8 Go** VDI dynamique
- 1 seul CPU

### 5.3 Configurer les 3 cartes réseau
Sélectionne la VM → **Configuration → Réseau** :

- **Adaptateur 1 (WAN)** : activé, attaché à **NAT**
- **Adaptateur 2 (LAN)** : activé, attaché à **Réseau interne**, nom : `lab-corp`
- **Adaptateur 3 (OPT1)** : activé, attaché à **Réseau interne**, nom : `lab-red`

Astuce : dans « Avancé » de chaque carte, note le type (Intel PRO/1000) — par
défaut ça donne les interfaces `em0`, `em1`, `em2` dans pfSense, dans l'ordre
des adaptateurs.

### 5.4 Installer pfSense
- **Configuration → Stockage** : monte l'ISO pfSense sur le lecteur optique.
- Démarre la VM, laisse l'installeur se faire (Install → clavier → disque auto
  ZFS ou UFS, les deux conviennent). À la fin, **retire l'ISO** et redémarre.

### 5.5 Assigner les interfaces (console pfSense)
Au premier boot, pfSense demande l'assignation. Réponds :

- Setup VLANs ? → **n**
- WAN interface → `em0`
- LAN interface → `em1`
- OPT1 interface → `em2`
- Valide (`y`).

Puis, menu console :
- **Option 2 (Set interface IP address)** → LAN :
  - IPv4 : `10.10.10.1` / `24`
  - Pas de passerelle amont sur LAN (Entrée)
  - Activer le serveur DHCP sur LAN ? → **n** (on met des IP statiques, plus propre
    pour l'AD)
- **Option 2** à nouveau → OPT1 :
  - IPv4 : `10.10.20.1` / `24`
  - DHCP → **n**

Le WAN reste en DHCP (il reçoit une IP de la NAT VBox automatiquement).

---

## 6. Finaliser via l'interface web (webConfigurator)

Le GUI pfSense n'est accessible que depuis une VM d'un des segments internes
(l'hôte ne voit pas les réseaux internes). Deux options :

- **Simple** : attendre l'étape 2, et administrer pfSense depuis le navigateur de
  la Kali/Windows (`https://10.10.10.1` ou `https://10.10.20.1`).
- **Tout de suite** : démarre temporairement une petite VM Linux Live dans
  `lab-corp` avec IP `10.10.10.2/24`, ouvre `https://10.10.10.1`
  (login `admin` / `pfsense`), change le mot de passe, puis éteins-la.

Dans le webConfigurator, **active l'interface OPT1** (Interfaces → OPT1 → Enable,
sans changer l'IP déjà posée en console) et donne-lui le nom `RED`.

### Règles de pare-feu (point pédagogique clé)
- **LAN** : règle par défaut « allow LAN to any » (laisse pour l'installation).
- **RED (OPT1)** : ajoute une règle **allow RED → lab-corp** (c'est ce qui permet
  à l'attaquant d'atteindre le DC pour l'exercice de l'étape 3).
- On garde ces règles larges volontairement. À l'**étape 4**, on les resserrera
  (micro-segmentation) et on observera ce que ça bloque dans la kill chain.

---

## 7. Vérifications avant de passer à l'étape 2

- [ ] pfSense démarre et montre bien 3 interfaces (WAN/LAN/OPT1) avec les bonnes IP.
- [ ] Depuis une VM de test dans `lab-corp`, tu pingues `10.10.10.1`.
- [ ] Depuis une VM de test dans `lab-red`, tu pingues `10.10.20.1`.
- [ ] Une VM `lab-red` peut atteindre une VM `lab-corp` à travers pfSense (routage OK).
- [ ] Le WAN a bien une IP et pfSense peut sortir (test ping `1.1.1.1` depuis sa
      console) — mais les VM internes ne sortent que selon tes règles.

---

## 8. Lien défense / cadre (on garde le réflexe dès le socle)

- **NIST / durcissement** : la segmentation est une mesure préventive (réduction
  de la surface de mouvement latéral). On la mesurera concrètement à l'étape 4.
- **MITRE ATT&CK** : le réseau à plat qu'on monte ici facilitera plus tard
  `Lateral Movement` (TA0008) — c'est voulu, pour voir l'avant/après durcissement.
- **Journalisation** : pfSense peut logguer les connexions bloquées/autorisées
  (Status → System Logs → Firewall). Ce sera notre première source côté bleu.

---

## 9. Approfondissement : segmentation, VLAN et règles de filtrage

### 9.1 Pourquoi segmenter (le fond)
Segmenter = découper le réseau en zones à qui on applique une politique de
confiance différente. Objectifs :
- **Réduire la surface de mouvement latéral** — une fois l'attaquant sur une
  machine, la segmentation limite ce qu'il peut atteindre ensuite
  (MITRE ATT&CK `Lateral Movement`, TA0008).
- **Défense en profondeur** — le filtrage inter-zones est un point de contrôle et
  de **journalisation** (NIST SP 800-53, contrôle **SC-7 Boundary Protection**).
- **Confiner un incident** — un poste compromis en zone `RED` ne doit pas pouvoir
  parler librement à la zone `CORP`.

Dans ce lab on part volontairement d'un réseau **à plat** (RED peut atteindre
CORP) pour pouvoir **mesurer l'écart** une fois durci à l'étape 4.

### 9.2 VLAN réel vs réseaux internes VirtualBox
En entreprise, la segmentation L2 se fait avec des **VLAN 802.1Q** (chaque trame
porte un *tag* d'ID VLAN), et un équipement L3 (pare-feu ou switch routeur)
applique les **règles inter-VLAN**. VirtualBox n'a pas de tag 802.1Q, mais un
**« réseau interne »** crée un domaine de broadcast séparé : fonctionnellement,
c'est la même isolation que deux VLAN distincts. pfSense joue le rôle du pare-feu
inter-VLAN.

Transposition vers ta stack pro (le raisonnement se transfère tel quel) :

| Lab (VirtualBox)        | Équivalent FortiGate / Cisco                         |
|-------------------------|------------------------------------------------------|
| Réseau interne `lab-corp` | VLAN 10 « CORP » — interface/zone FortiGate, ou SVI Cisco `int vlan 10` |
| Réseau interne `lab-red`  | VLAN 20 « RED » — interface/zone FortiGate, ou SVI `int vlan 20` |
| Interface pfSense `10.10.10.1` | Passerelle de VLAN 10 (SVI ou sous-interface taggée) |
| Règle pfSense RED→CORP  | **Policy FortiGate** : srcintf `RED` / dstintf `CORP` / service / action / log |

Si tu veux coller à FortiGate plus tard, on remplacera pfSense par une FortiGate
VM : mêmes zones, mêmes politiques, juste la syntaxe change.

### 9.3 Règles de filtrage — jeu de base (apprentissage)
À poser dans pfSense (Firewall → Rules). C'est le jeu « réseau à plat » de départ :

| # | Interface | Source        | Destination     | Service | Action | Log |
|---|-----------|---------------|-----------------|---------|--------|-----|
| 1 | LAN (CORP)| `lab-corp` net| any             | any     | Pass   | non |
| 2 | RED (OPT1)| `lab-red` net | `lab-corp` net  | any     | Pass   | **oui** |
| 3 | RED (OPT1)| `lab-red` net | WAN (internet)  | any     | Pass   | non |

> Active le **log sur la règle 2** dès maintenant : ce sera ta première trace
> côté bleu (qui, de RED, parle à CORP et sur quels ports). Status → System Logs
> → Firewall.

### 9.4 Règles durcies — aperçu de l'étape 4 (ne pas appliquer encore)
À l'étape 4, on remplacera la règle 2 par un filtrage de moindre privilège, pour
voir ce qui « casse » dans la kill chain :

| # | Interface | Source        | Destination     | Service        | Action | Log |
|---|-----------|---------------|-----------------|----------------|--------|-----|
| 1 | RED       | `lab-red` net | `10.10.10.10` (DC) | TCP 443 (appli autorisée) | Pass | oui |
| 2 | RED       | `lab-red` net | `lab-corp` net  | any            | **Block** | oui |

On comparera alors ce que l'attaquant pouvait faire avant/après (ex. SMB,
Kerberos, RPC bloqués), en reliant chaque blocage à la technique ATT&CK qu'il
neutralise.

---

## Prochaine étape

**Étape 2** : déployer le contrôleur de domaine Windows Server (AD DS) dans
`lab-corp` et la Kali dans `lab-red`, avec les IP statiques du tableau ci-dessus,
puis créer le domaine et quelques faiblesses réalistes à exploiter.
