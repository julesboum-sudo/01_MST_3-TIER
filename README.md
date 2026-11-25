# 🌐 Architecture Réseau d'Entreprise Haute Disponibilité (MSTP, HSRP, OSPF, LACP)

Ce dépôt présente la conception, la configuration et la validation d'une architecture réseau hiérarchique à trois couches (**Cœur, Distribution, Accès**) assurant la redondance des liens, la répartition de charge et l'accès à Internet sous Cisco IOS.

---

## 📐 Topologie du Réseau

![Topologie du réseau](TOPOLOGIE.jpg)

---

## 🛠️ Technologies et Protocoles Déployés

* **Commutation & Redondance L2 :**
  * **MSTP (Multiple Spanning Tree Protocol - 802.1s) :** Domaine `CIN4_MST_LAB` (Révision 1) avec répartition de charge par instances :
    * *Instance 1 (Root : AG1)* : VLANs 211, 212, 225, 235
    * *Instance 2 (Root : AG2)* : VLANs 245, 255
  * **EtherChannel (LACP - 802.3ad) :** Agrégation de liens entre la couche Distribution et Accès (`Po19`, `Po21`) ainsi qu'interconnexion L3 entre `AG1` et `AG2` (`Po20`).
  * **VTPv2 :** Synchronisation centralisée des VLANs (Domaine `CIN4ENSPY`) avec `AG1`/`AG2` en mode *Server* et `SW1`/`SW2` en mode *Client*.
  * **Sécurité des ports :** Activation de `PortFast`, `BPDU Guard` et `Port-Security` (MAC Sticky, max 2 adresses, violation *shutdown*) sur les ports d'accès.

* **Routage & Haute Disponibilité L3 :**
  * **HSRP (Hot Standby Router Protocol) :** Redondance de passerelle par défaut avec répartition de charge active/passif (`AG1` actif pour les VLANs 225 et 235 ; `AG2` actif pour les VLANs 245 et 255).
  * **OSPF (Area 0) :** Routage dynamique interne entre les routeurs cœurs (`FAI`, `RO-1`, `RO-2`) et les commutateurs de distribution (`AG1`, `AG2`), avec injection de route par défaut (`default-information originate`).

* **Services Réseau :**
  * **DHCP & Relais DHCP :** Serveurs DHCP redondants configurés sur `RO-1` et `RO-2` avec relais (`ip helper-address`) sur les interfaces SVI de la couche distribution.
  * **NAT / PAT :** Translation d'adresses (`ip nat inside source overload`) sur le routeur `FAI` pour fournir un accès Internet aux postes clients internes.

---

## 📊 Plan d'Adressage IP

| Équipement | Interface | Adresse IP | Masque | Rôle / Description |
| :--- | :--- | :--- | :--- | :--- |
| **FAI** | `Loopback1` | `2.2.2.1` | `/32` | Router-ID OSPF |
| | `e0/0` | `DHCP` | — | Accès Internet (NAT Outside) |
| | `e0/1` | `24.40.2.2` | `/30` | Interconnexion vers `RO-1` |
| | `e0/2` | `25.40.2.2` | `/30` | Interconnexion vers `RO-2` |
| **RO-1** | `Loopback1` | `2.2.2.2` | `/32` | Router-ID & Serveur DHCP |
| | `e0/0` | `24.40.2.1` | `/30` | Interconnexion vers `FAI` |
| | `e0/1` | `192.168.10.18` | `/30` | Interconnexion vers `AG1` |
| | `e0/2` | `192.168.10.22` | `/30` | Interconnexion vers `AG2` |
| **RO-2** | `Loopback1` | `2.2.2.3` | `/32` | Router-ID & Serveur DHCP |
| | `e0/0` | `25.40.2.1` | `/30` | Interconnexion vers `FAI` |
| | `e0/1` | `192.168.10.6` | `/30` | Interconnexion vers `AG2` |
| | `e0/2` | `192.168.10.26` | `/30` | Interconnexion vers `AG1` |
| **AG1** | `Loopback0` | `2.2.2.4` | `/32` | Router-ID OSPF |
| | `e0/0` | `192.168.10.17` | `/30` | Interconnexion vers `RO-1` |
| | `e0/1` | `192.168.10.25` | `/30` | Interconnexion vers `RO-2` |
| | `Port-channel 20` | `192.168.10.9` | `/30` | Lien L3 LACP vers `AG2` |
| | `SVI 225` (HN) | `192.168.225.1` | `/24` | HSRP Active (VIP : `.254`) |
| | `SVI 235` (AN) | `192.168.235.1` | `/24` | HSRP Active (VIP : `.254`) |
| | `SVI 245` (CIN) | `192.168.245.1` | `/24` | HSRP Standby (VIP : `.254`) |
| | `SVI 255` (ICD) | `192.168.255.1` | `/24` | HSRP Standby (VIP : `.254`) |
| **AG2** | `Loopback0` | `2.2.2.5` | `/32` | Router-ID OSPF |
| | `e0/0` | `192.168.10.5` | `/30` | Interconnexion vers `RO-2` |
| | `e0/1` | `192.168.10.21` | `/30` | Interconnexion vers `RO-1` |
| | `Port-channel 20` | `192.168.10.10` | `/30` | Lien L3 LACP vers `AG1` |
| | `SVI 225` (HN) | `192.168.225.2` | `/24` | HSRP Standby (VIP : `.254`) |
| | `SVI 235` (AN) | `192.168.235.2` | `/24` | HSRP Standby (VIP : `.254`) |
| | `SVI 245` (CIN) | `192.168.245.2` | `/24` | HSRP Active (VIP : `.254`) |
| | `SVI 255` (ICD) | `192.168.255.2` | `/24` | HSRP Active (VIP : `.254`) |

### Récapitulatif des VLANs

* **VLAN 200 (`NATIF`) :** VLAN natif des liens trunks 802.1Q
* **VLAN 211 (`MGMT`) & VLAN 212 (`PARC`) :** Administration et gestion de parc
* **VLAN 225 (`HN`) :** `192.168.225.0/24` — Passerelle : `192.168.225.254` (`VPC8`)
* **VLAN 235 (`AN`) :** `192.168.235.0/24` — Passerelle : `192.168.235.254` (`VPC9`)
* **VLAN 245 (`CIN`) :** `192.168.245.0/24` — Passerelle : `192.168.245.254` (`VPC10`)
* **VLAN 255 (`ICD`) :** `192.168.255.0/24` — Passerelle : `192.168.255.254` (`VPC11`)

---

## 📁 Contenu du Dépôt

Le dépôt est organisé de la manière suivante :

```text
├── README.md                                                 # Documentation générale du projet
├── TOPOLOGIE.jpg                                             # Schéma complet de l'architecture réseau
├── Rapport_BOUM_LIKAK_JULES_DAROLD_23P065_TPE_01_MST.pdf     # Rapport détaillé (configurations et vérifications)
└── configs/                                                  # Fichiers de configuration Cisco IOS par couche
    ├── 01-Coeur/
    │   ├── FAI.txt                                           # Routeur FAI (OSPF, NAT/PAT)
    │   ├── RO-1.txt                                          # Routeur Cœur 1 (OSPF, Serveur DHCP)
    │   └── RO-2.txt                                          # Routeur Cœur 2 (OSPF, Serveur DHCP)
    ├── 02-Distribution/
    │   ├── AG1.txt                                           # Switch L3 Distribution 1 (MSTP Root Inst. 1, HSRP, VTP Server, LACP, OSPF)
    │   └── AG2.txt                                           # Switch L3 Distribution 2 (MSTP Root Inst. 2, HSRP, VTP Server, LACP, OSPF)
    └── 03-Acces/
        ├── SW1.txt                                           # Switch Accès 1 (MSTP, VTP Client, LACP, Port-Security)
        └── SW2.txt                                           # Switch Accès 2 (MSTP, VTP Client, LACP, Port-Security)
