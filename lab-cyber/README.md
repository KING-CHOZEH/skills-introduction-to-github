# Lab Cybersécurité — pentest / blue team / durcissement

Dépôt de versionnement du lab d'entraînement de **KING** (réseau & cybersécurité,
environnement pro FortiGate / Cisco / Active Directory / Microsoft 365).
On y garde les **configs**, **schémas** et **fiches d'exercices** du lab, pas à
pas, en reliant chaque attaque à sa **détection** et à sa **remédiation**
(MITRE ATT&CK, cyber kill chain, NIST).

> ⚠️ **Cadre impératif.** Tout se fait **exclusivement** dans des environnements
> possédés ou explicitement autorisés : VM personnelles, plateformes légales,
> labs isolés. **Aucun pont (Bridge) vers un réseau réel.** Les faiblesses posées
> dans les exercices sont **volontaires et pédagogiques** : elles n'ont de sens
> que dans ce lab coupé du monde.

---

## Vue d'ensemble

Hyperviseur : **VirtualBox** (hôte ~8 Go RAM). Segmentation via *réseaux internes*
VirtualBox (pas de VLAN matériel), pare-feu **pfSense** jouant le rôle de routeur
inter-zones. Le raisonnement se transpose directement vers une FortiGate / un
Cisco en entreprise.

| Segment    | Réseau interne VBox | Sous-réseau       | Passerelle pfSense |
|------------|---------------------|-------------------|--------------------|
| WAN        | NAT                 | fourni par VBox   | automatique        |
| `lab-corp` | `lab-corp`          | `10.10.10.0/24`   | `10.10.10.1`       |
| `lab-red`  | `lab-red`           | `10.10.20.0/24`   | `10.10.20.1`       |

| Machine          | Segment    | IP statique   | Rôle                               |
|------------------|------------|---------------|------------------------------------|
| `pfSense-lab`    | WAN+2 int. | `.1` de chaque| Routeur / pare-feu / journal       |
| `DC01`           | `lab-corp` | `10.10.10.10` | Contrôleur de domaine `sodeco.lab` |
| `kali-red`       | `lab-red`  | `10.10.20.50` | Poste attaquant                    |

Schéma détaillé : [`schemas/topologie.md`](schemas/topologie.md).

---

## Arborescence du dépôt

```
lab-cyber/
├── README.md                     ← ce fichier (index + cadre + topologie)
├── schemas/
│   └── topologie.md              ← topologie logique et plan d'adressage
├── configs/
│   └── pfsense/
│       └── pare-feu-regles.md    ← jeux de règles de base et durcies
└── exercices/
    ├── etape-1-socle-reseau.md   ← socle réseau segmenté (pfSense)
    └── etape-2-cibles-et-ad.md   ← cibles + Active Directory sodeco.lab
```

---

## Progression (plan en 4 étapes)

| Étape | Objectif                                                        | État      |
|-------|-----------------------------------------------------------------|-----------|
| 1     | Socle réseau segmenté — pfSense + 2 réseaux internes            | ✅ rédigée |
| 2     | Cibles + Active Directory — DC `sodeco.lab` + Kali, faiblesses  | ✅ rédigée |
| 3     | Premier cycle rouge → bleu mappé sur la kill chain (+ Sysmon)   | à venir   |
| 4     | Durcissement (GPO, tiering AD) + détection (Sysmon + règles)    | à venir   |

Chaque fiche d'exercice relie l'attaque posée à sa trace défensive et à sa
contre-mesure. La fiche **blue team** transversale (chaque technique ↔ détection
↔ remédiation, indexée sur MITRE ATT&CK) est suivie à part.

---

## Convention de versionnement

- Une branche par étape ou par chantier (`etape-3-...`, `durcissement-...`).
- Les artefacts lourds (ISO, disques VM, exports) **ne sont pas** versionnés
  (voir le `.gitignore` à la racine) — on versionne la **doc** et les **configs
  texte**, pas les images disque.
- Les secrets réels n'ont pas leur place ici : les mots de passe présents dans
  les fiches sont **fictifs** et propres au lab isolé.
