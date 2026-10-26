# Topologie du lab

Topologie **logique** du lab isolé (VirtualBox). pfSense route et filtre entre
deux réseaux internes ; seul pfSense a une sortie (NAT) vers Internet, pour les
mises à jour uniquement.

```
                 Internet (updates only)
                        │
                   [ NAT VBox ]
                        │ WAN (em0)
                  ┌───────────┐
                  │  pfSense  │  routeur / pare-feu / journal
                  └─────┬─────┘
            LAN (em1)   │   OPT1 (em2)
        ┌───────────────┤   ├────────────────┐
        │  lab-corp     │   │   lab-red       │
        │ 10.10.10.0/24 │   │ 10.10.20.0/24   │
        │               │   │                 │
     [ DC01 Windows ]       [ kali-red ]
      10.10.10.10            10.10.20.50
      sodeco.lab (AD DS)     attaquant
```

## Plan d'adressage

| Segment   | Réseau VirtualBox         | Sous-réseau     | Passerelle (pfSense) |
|-----------|---------------------------|-----------------|----------------------|
| WAN       | NAT                       | fourni par VBox | automatique          |
| lab-corp  | Réseau interne `lab-corp` | `10.10.10.0/24` | `10.10.10.1`         |
| lab-red   | Réseau interne `lab-red`  | `10.10.20.0/24` | `10.10.20.1`         |

| Machine        | Segment  | IP (statique) | Passerelle   | DNS            |
|----------------|----------|---------------|--------------|----------------|
| DC01 (Win Srv) | lab-corp | `10.10.10.10` | `10.10.10.1` | `127.0.0.1` *  |
| kali-red       | lab-red  | `10.10.20.50` | `10.10.20.1` | `10.10.10.10`**|

\* Le DC est son propre serveur DNS une fois AD installé.
\*\* La Kali pointe sur le DNS du DC pour résoudre le domaine `sodeco.lab`.

## Transposition vers la stack pro

| Lab (VirtualBox)              | Équivalent FortiGate / Cisco                         |
|-------------------------------|------------------------------------------------------|
| Réseau interne `lab-corp`     | VLAN 10 « CORP » — zone FortiGate, ou SVI `int vlan 10` |
| Réseau interne `lab-red`      | VLAN 20 « RED » — zone FortiGate, ou SVI `int vlan 20`  |
| Interface pfSense `10.10.10.1`| Passerelle de VLAN 10 (SVI ou sous-interface taggée) |
| Règle pfSense RED→CORP        | Policy FortiGate : srcintf `RED` / dstintf `CORP` / service / action / log |

VirtualBox n'a pas de tag 802.1Q, mais un « réseau interne » crée un domaine de
broadcast séparé : fonctionnellement, la même isolation que deux VLAN distincts.
