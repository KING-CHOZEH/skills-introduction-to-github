# pfSense — jeux de règles de pare-feu

Référence des règles de filtrage du lab (Firewall → Rules). Deux jeux : le jeu
**de base** (réseau « à plat », étapes 1→3) et le jeu **durci** (étape 4). On
compare l'avant/après pour mesurer concrètement l'apport de la segmentation.

> Interfaces pfSense : `WAN` (NAT, em0), `LAN` = CORP (em1, `10.10.10.1`),
> `OPT1` = RED (em2, `10.10.20.1`).

---

## Jeu de base — réseau à plat (étapes 1 à 3)

Volontairement permissif : l'attaquant en `RED` peut atteindre `CORP`, ce qui
rend l'exercice de mouvement latéral possible.

| # | Interface  | Source         | Destination      | Service | Action | Log     |
|---|------------|----------------|------------------|---------|--------|---------|
| 1 | LAN (CORP) | `lab-corp` net | any              | any     | Pass   | non     |
| 2 | RED (OPT1) | `lab-red` net  | `lab-corp` net   | any     | Pass   | **oui** |
| 3 | RED (OPT1) | `lab-red` net  | WAN (internet)   | any     | Pass   | non     |

> **Active le log sur la règle 2 dès maintenant** : c'est la première source
> côté bleu (qui, depuis RED, parle à CORP et sur quels ports).
> Status → System Logs → Firewall.

---

## Jeu durci — moindre privilège (étape 4, ne pas appliquer avant)

On remplace la règle large RED→CORP par un filtrage de moindre privilège, puis on
observe ce qui « casse » dans la kill chain (SMB, Kerberos, RPC bloqués).

| # | Interface | Source        | Destination        | Service                   | Action    | Log |
|---|-----------|---------------|--------------------|---------------------------|-----------|-----|
| 1 | RED       | `lab-red` net | `10.10.10.10` (DC) | TCP 443 (appli autorisée) | Pass      | oui |
| 2 | RED       | `lab-red` net | `lab-corp` net     | any                       | **Block** | oui |

Chaque blocage se relie ensuite à la technique ATT&CK qu'il neutralise.

---

## Lien défense / cadre

- **NIST SP 800-53, SC-7 (Boundary Protection)** : le filtrage inter-zones est un
  point de contrôle et de journalisation.
- **MITRE ATT&CK — Lateral Movement (TA0008)** : le jeu de base facilite le
  mouvement latéral (voulu) ; le jeu durci le contraint.
- **Journal pfSense** : Status → System Logs → Firewall — première source bleue
  du lab avant l'ajout de Sysmon sur le DC (étape 3).
