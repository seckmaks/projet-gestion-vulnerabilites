# Cahier des charges — Lab EVE-NG « VMS Université » (baseline + remédiation)

> **Objectif** : matérialiser l'architecture réelle de l'université (Rectorat + bâtiments
> MET/SATA/ILAMEL + SMS/Scolarité + cloud IaaS SONATEL + double liaison FO) dans EVE-NG,
> avec une **plateforme de gestion des vulnérabilités (Greenbone/GCE)** pour couvrir les
> 4 phases du cycle de vie : **inventory → vuln scan → report → remediation**.
> **Topologie réseau complète conservée** (double FO, cœur, TOR, accès) ; la contrainte
> mémoire (16 GB) est traitée par **RAM allégées + swap + démarrage par vague** —
> **aucun équipement réseau n'est retiré**.
> **Périmètre** : Rectorat (DC) + bâtiment MET/SATA/ILAMEL (VLAN 30) + bâtiment
> SMS/Scolarité (VLAN 31, **SW-SMS**) + cloud IaaS (SRV-CLOUD) + scanner (GBVA) +
> opérateur (double FO). Bâtiment Rectorat « PC-RECTO » retiré ; TOUBA CA KANAM
> (VLAN 33) = **extension documentée, non câblée** dans la variante retenue.
> **Nom de lab conseillé : `VMS-2026`.**

## 1. Contexte & justification

- **Fidélité au réel** : R-CE-1/R-CE-2 (centraux MADIYANA / NDAM MBACKÉ), cœur **R-CE**
  (IOSv L3, équiv. du switch cœur du DC), SW-TOR, SW-MET, SW-SMS, cloud IaaS représenté
  par **une seule VM** (SRV-CLOUD — la passerelle IaaS managée est hors périmètre),
  terminaux du parc (parc réel = Windows 11 ; le lab utilise Windows 10/7 pour les
  constats de fin de support), **GBVA = Greenbone Community Edition importée dans le lab**.
- **Fait de terrain confirmé** : le cloud est accessible **SANS VPN** (lien clair) —
  c'est la **baseline** ; le tunnel IPSec est la **remédiation** (Lab 4) + ACL
  « seul le scanner (VLAN 10) accède au cloud ».
- **Pédagogie** : chaque élément a une vulnérabilité « cible » (Win7 EOL/MS17-010,
  IOS telnet/SNMP v1, Ubuntu 20.04 périmé, caméra par défaut, mdp GSA par défaut).
- **Extrapolation 8 sites** : le lab modélise Rectorat + 2 bâtiments ; le raisonnement
  (segmentation, priorisation, KPI) s'étend aux 8 sites via l'inventaire réel.

## 2. Périmètre fonctionnel

| Domaine | Contenu |
|---|---|
| Opérateur (2 FO) | Cloud « Net » (IP/MPLS, **NAT 10.0.0.254**) + R-CE-1 (10.0.0.3) + R-CE-2 (10.0.0.2) |
| Data center | R-CE (cœur IOSv L3), SW-TOR (distribution) |
| Bâtiment MET/SATA/ILAMEL | SW-MET (VLAN 30) + Windows-10 + Win-7 + IP-Phone (VPCS) + Camera (Ubuntu, web :80) |
| Bâtiment SMS/Scolarité | SW-SMS (VLAN 31) + Windows-10-1 + Win-7-1 + IP-Phone-1 (VPCS) |
| Cloud IaaS | **SRV-CLOUD (SEULE VM)** — Ubuntu 20.04 non patché, **1 NIC (ens3)** : 172.16.90.2/30 → cœur **G0/2** + **10.9.1.10/24 en IP secondaire** (données, config multi-IP façon Azure) |
| Plateforme GdV | **GBVA (Kali Linux + GVM/OpenVAS) = nœud du lab** (qcow2, 4 GB) : e0 → SW-TOR G0/2 (VLAN 10, 172.16.10.10/24) ; e1 → cloud **bridge** LAN hôte (192.168.1.200/24 statique : internet/NVT + GSA) |
| Lab 4 (remédiation) | Tunnel IPSec site-to-site R-CE ↔ SRV-CLOUD (IKEv2/AES-256/SHA-256/GRP14, PSK) + ACL d'isolation (VLAN 10 uniquement) |
| Lab 6 (disponibilité) | Basculement FO principale (shut **G0/0 du cœur R-CE**) → route flottante R-CE-2 (distance 252) |

## 3. Plan d'adressage (résumé)

| Réseau | Usage | Attributions |
|---|---|---|
| 10.0.0.0/24 (CLOUD-OP) | Segment opérateur « Net » (cloud NAT) | GW 10.0.0.254 · R-CE-1 **10.0.0.3** · R-CE-2 10.0.0.2 |
| 172.16.255.0/30 | Lien R-CE-1 ↔ R-CE | .1 / .2 |
| 172.16.254.0/30 | Lien R-CE-2 ↔ R-CE | .1 / .2 |
| 172.16.90.0/30 | Lien R-CE **G0/2** ↔ SRV-CLOUD **ens3** — SEULE liaison cloud : uplink + données (plain = F-01) | .1 / .2 (+ 10.9.1.10 sec.) |
| ~~172.16.91.0/30~~ | **Supprimé (20/09)** — 2ᵉ liaison SRV-CLOUD retirée ; cœur **G0/4** = libre (réserve) | — |
| 10.9.255.0/30 | Tunnel IPSec (Lab 4) | .1 (R-CE) / .2 (SRV-CLOUD) |
| 10.9.1.0/24 | Données cloud | 10.9.1.10 (SRV-CLOUD, **IP secondaire sur ens3**) |
| 172.16.10.0/24 (VLAN 10) | MGMT — scanner GBVA | .1 (SVI R-CE) · .2 (SVI SW-TOR) · .10 (GBVA e0) |
| 192.168.1.0/24 (LAN hôte, bridge) | Internet/GSA de GBVA | .1 (box) · .200 (GBVA e1, statique) |
| 172.16.30.0/24 (VLAN 30) | MET/SATA/ILAMEL | .1 (SVI cœur) · .2 (SVI SW-MET) · .10 Windows-10 · .11 Camera · .50 Win-7 · .60 IP-Phone |
| 172.16.31.0/24 (VLAN 31) | SMS/SCOLARITÉ | .1 (SVI cœur) · .2 (SVI SW-SMS) · .10 Win-7-1 · .11 Windows-10-1 · .12 IP-Phone-1 |
| 172.16.33.0/24 (VLAN 33) | TOUBA CA KANAM — extension non câblée | (réserve) |

## 4. Liste des nœuds (15 nœuds + 2 clouds) — variante hôte 16 GB / EVE-NG 8 GB

| # | Nom | Type / image | RAM | Rôle |
|---|---|---|---|---|
| 1 | Net | cloud (opérateur) | 0 | IP/MPLS SONATEL (GW 10.0.0.254) |
| 2 | R-CE-1 | IOSv 15.7M2 (AdvEnt, dev test) | 0,5 GB | CE — FO principale MADIYANA |
| 3 | R-CE-2 | IOSv 15.7M2 (AdvEnt, dev test) | 0,5 GB | CE — FO secours NDAM MBACKÉ |
| 4 | R-CE | IOSv 15.7M2 (AdvEnt, L3) | 0,75 GB | Cœur : inter-VLAN, double FO, accès cloud, tunnel IPSec |
| 5 | SW-TOR | IOSvL2 15.2 | 0,5 GB | Distribution 802.1Q |
| 6 | SW-MET | IOSvL2 15.2 | 0,5 GB | Accès bâtiment MET (VLAN 30) |
| 7 | SW-SMS | IOSvL2 15.2 | 0,5 GB | Accès bâtiment SMS/SCOLARITÉ (VLAN 31) |
| 8 | GBVA | **Kali Linux + GVM/OpenVAS (qcow2)** | **4 GB** | Scanner GdV — e0 VLAN 10 (lab) / e1 bridge LAN hôte |
| 9 | SRV-CLOUD | Ubuntu 20.04 (non patché) | 0,5 GB | **Seule VM cloud** : 1 NIC ens3 — 172.16.90.2/30 + 10.9.1.10/24 (IP secondaire) |
| 10 | Windows-10 | Win10 qcow2 (legacy BIOS) | 1,0 GB | Poste MET — WinRM `student` |
| 11 | Camera | Ubuntu headless | 0,5 GB | Caméra IP : page web :80, `admin/1234` (V-05) |
| 12 | IP-Phone | **VPCS** (natif EVE-NG) | ~0,1 GB | Terminal MET (VPCS) |
| 13 | Win-7 | Win7 SP1 qcow2 | 1,0 GB | Poste legacy — MS17-010 (V-01) |
| 14 | Windows-10-1 | Win10 qcow2 | 1,0 GB | Poste SMS |
| 15 | Win-7-1 | Win7 SP1 qcow2 | 1,0 GB | Poste legacy SMS (2ᵉ EOL) |
| 16 | IP-Phone-1 | **VPCS** (natif EVE-NG) | ~0,1 GB | Terminal SMS (VPCS) |
| — | CLOUD - AZURE (cloud rouge) | cloud EVE-NG | 0 | Enveloppe visuelle du cloud (SRV-CLOUD) |

**Total « tout up » ≈ 12,4 GB** → EVE-NG plafonné à **8 GB** : **swap file 8 GB** +
**démarrage par vague** (voir §6) ; les RAM Windows sont allégées (1,0 GB au lieu de 1,5) ;
les 6 IOS/IOSvL2 ne dépassent pas 0,75 GB chacun.

## 5. Câblage (reproduire dans EVE-NG — conforme au lab réel)

| De | Port | Vers | Port | Type |
|---|---|---|---|---|
| Net | (cloud) | R-CE-1 | G0/0 | access 10.0.0.0/24 |
| Net | (cloud) | R-CE-2 | G0/0 | access 10.0.0.0/24 |

> **Réglage du cloud « Net » (EVE-NG → Cloud Settings)** : Type = **NAT**, IP = **10.0.0.254**,
> masque 255.255.255.0 → devient la passerelle des CEs + « internet simulé » (masquerade via
> l'internet de l'hôte EVE-NG). Vérif : `ping 10.0.0.254` puis `ping 8.8.8.8` depuis R-CE-1.
> (Rappel : le label texte « GW: 10.0.0.254 » affiché sur le canvas ne configure rien — c'est le
> **Type = NAT + IP** dans Cloud Settings qui crée la passerelle.)
| R-CE-1 | G0/1 | R-CE (cœur) | G0/0 | 172.16.255.0/30 |
| R-CE-2 | G0/1 | R-CE (cœur) | G0/1 | 172.16.254.0/30 |
| R-CE (cœur) | G0/2 | SRV-CLOUD | ens3 | 172.16.90.0/30 — SEULE liaison cloud (plain = F-01 ; 10.9.1.10 sec. sur ens3) |
| R-CE (cœur) | G0/3 | SW-TOR | G0/0 | trunk VLAN 10,30,31 |
| SW-TOR | G0/1 | SW-MET | G0/0 | trunk VLAN 30 |
| SW-TOR | G0/2 | GBVA | e0 | **accès VLAN 10** (scanner) |
| SW-TOR | G0/3 | SW-SMS | G0/0 | trunk VLAN 31 |
| SW-MET | G0/1 | Windows-10 | e0 | accès 30 |
| SW-MET | G0/2 | Win-7 | e0 | accès 30 |
| SW-MET | G0/3 | IP-Phone | eth0 | accès 30 |
| SW-MET | G1/0 | Camera | e0 | accès 30 |
| SW-SMS | G0/1 | Windows-10-1 | e0 | accès 31 |
| SW-SMS | G0/2 | IP-Phone-1 | eth0 | accès 31 |
| SW-SMS | G0/3 | Win-7-1 | e0 | accès 31 |
| GBVA | e1 | cloud **bridge** (LAN hôte 192.168.1.0/24) | — | 192.168.1.200/24 statique |

Ports **vérifiés sur le lab (20/09, réseau communicant)** : 5ᵉ port du cœur = **G0/4**
(**libre/réserve** — 2ᵉ liaison SRV-CLOUD supprimée le 20/09) ; SW-MET : G0/2 Win-7,
G0/3 IP-Phone, **G1/0 Camera** ; SRV-CLOUD (Ubuntu) = **ens3** (seule NIC). Ports IOS affichés
**`Gi0/x`** dans EVE-NG = **`G0/x`** dans IOS.

## 6. Contrainte mémoire (hôte 16 GB, EVE-NG 8 GB)

- **Partition hôte** : OS 3–4 GB · EVE-NG **8 GB + swap file 8 GB** (réglage dans les
  paramètres de l'appliance, fichier `swapfile` sur le disque hôte) · reste 4–5 GB.
- **Démarrage par vague** (recommandé, évite le swap permanent) :
  1. **Vague 1** (Labs 0/1) : 6 IOS + GBVA + SRV-CLOUD ≈ **7,75 GB** — tient dans les 8 GB.
  2. **Vague 2** (Lab 2) : + Windows-10 + Win-7 (bât. MET) ≈ 9,75 GB → swap 2 GB (OK).
  3. **Vague 3** (Lab 3/4) : SRV-CLOUD + GBVA + 1 poste ≈ 9–10 GB → OK.
  4. **Lab 6** : pauser le bâtiment SMS (−2,5 GB) si nécessaire.
  5. « Tout up » (captures/soutenance) ≈ 12,4 GB → swap 4–5 GB : **lâcher le lab 1 h avant**,
     ne pas lancer les 15 nœuds en un seul clic.
- **Si l'hôte devient trop lent** : GBVA 4→3 GB (GCE fonctionne à 3 GB, scans plus lents),
  Windows 1,0→0,75 GB ; **jamais de retrait d'équipement réseau** (SW-TOR, R-CE-2, R-CE…).

## 7. Matrice vulnérabilités cibles

| # | Actif | Vulnérabilité plantée | Sévérité cible (CVSS) |
|---|---|---|---|
| V-01 | Win-7 / Win-7-1 (172.16.30.50 / .31.10) | Win7 EOL 2020, non patché, **MS17-010/SMBv1** | **Critique** |
| V-02 | Windows-10 / -1 | Win10 EOL (oct. 2025), WinUpdate off, mdp court, droits admin | Moyenne |
| V-03 | R-CE-1/2, R-CE, SW-TOR, SW-MET, SW-SMS | telnet, SNMP v1 `public`/v2 `private`, compte `guest` | Moyenne-Haute |
| V-04 | SRV-CLOUD (10.9.1.10) | Ubuntu **20.04 périmé** (EOL 2025) non patché, ssh, http | Haute |
| V-05 | Camera (172.16.30.11) | page web :80 par défaut, `admin/1234` | Moyenne |
| V-06 | IP-Phone / IP-Phone-1 (VPCS) | terminaux gérés, mdp par défaut (représentation) | Faible |
| V-07 | GBVA (GSA) | compte admin GSA à mot de passe faible (`admin/admin`) planté au déploiement | Moyenne-Haute |
| F-01 | Réseau (constat) | **Cloud accessible SANS VPN** (lien 172.16.90.0/30 en clair) | Haute (remédié Lab 4) |

## 8. Règles de gestion du lab

1. **Démarrage par vague** (§6) — jamais les 15 nœuds en un clic.
2. **Baseline = état réel** : pas de VPN avant le Lab 4 ; les vulnérabilités IOS sont
   plantées **après** la validation de la baseline, et **maintenues** jusqu'au Lab 1
   (remédiation) — jamais « corrigées » avant le scan correspondant.
3. **Re-scan systématique après chaque remédiation** (preuve du workflow).
4. **Mots de passe** : à remplacer avant la soutenance (`<masqué>`, `<masqué>`,
   `<masqué>`, `admin/1234`…).
5. **Snapshot EVE-NG** après la baseline (`VMS-2026-baseline`) + snapshot après Lab 4
   (`VMS-2026-remedie`) → reproductibilité.
6. **GSA (GBVA)** : changer le mdp par défaut immédiatement après la capture du constat
   V-07 (capture d'abord, correction ensuite).

## 9. Limites du lab (documenter dans le mémoire)

1. **RAM allégées + swap** : le scanner (GCE 4 GB) et les postes tournent dans l'enveloppe
   EVE-NG 8 GB — le placement est un choix de dimensionnement documenté, pas une contrainte
   de la topologie logique.
2. **1 seul bâtiment par site** en baseline (MET + SMS) ; 8 sites extrapolés depuis
   l'inventaire réel.
3. **Windows 7/10** à la place du parc réel Windows 11 (constats EOL/MS17-010 non
   disponibles sur Win11 ; logique d'inventaire identique).
4. **Cloud = 1 VM (SRV-CLOUD)** : la passerelle IaaS managée est hors périmètre ; SRV-CLOUD
   joue double rôle sur **une seule NIC** (ens3 : uplink 172.16.90.0/30 + 10.9.1.10/24 en IP
   secondaire — config multi-IP façon Azure) ; le VPN y est ajouté au Lab 4 (Libreswan, tunnel sur ens3).
5. **IOCs représentés** (VPCS/Camera) : constats = inventaire, pas d'audit profond.
6. **Images IOSv dev/eval** (~180 j, build « DEVELOPMENT TEST ») : réimporter avant la
   soutenance si expirées (fallback : IOSv 15.9(3)M stable).
7. **Outillage obj. 5-7 hors EVE-NG** : DefectDojo + Mattermost = Docker sur l'**hôte** (2-3 GB,
   démarrés seulement pendant les ateliers), Jira = Cloud, **Ansible + gvm-tools = GBVA** —
   aucun nœud ajouté au lab, RAM du lab inchangée (06, ateliers B/C ; 04 §10-§11).

## 10. Images à préparer (checklist)

- [ ] **IOSv AdvEnterprise 15.7(3)M2 (dev test, ~180 j)** : converti `.qcow2` pour EVE-NG
      (R-CE-1, R-CE-2, R-CE) — fallback IOSv 15.9(3)M stable si expiration.
- [ ] **IOSvL2 15.2 (eval)** : converti `.qcow2` (SW-TOR, SW-MET, SW-SMS).
- [ ] **Kali Linux qcow2 (pré-construite)** : nœud GBVA dans EVE-NG (4 GB, 2 NIC, BIOS) +
      `apt install gvm` + `gvm-setup` (sync NVT) — plan B : OVA GCE convertie qcow2.
- [ ] **Ubuntu 20.04 (SRV-CLOUD)** : image non patchée, SSHd, nginx (page `/app` HTTP clair),
      compte `scan/<masqué>` (sudo) + `svc-app/<masqué>`.
- [ ] **Ubuntu headless (Camera)** : nginx, page web par défaut port 80, `admin/1234`.
- [ ] **Windows 10 qcow2 ×2** (legacy BIOS) : WinRM, WinUpdate off, comptes lab.
- [ ] **Windows 7 SP1 qcow2 ×2** : SMBv1 actif, `admin/admin`, RDP.
- [ ] **VPCS (×2 — natif EVE-NG, aucune image à importer)** : IP-Phone, IP-Phone-1.
- [ ] **Cloud « bridge »** (adaptateur LAN hôte 192.168.1.0/24) : pour la NIC e1 de GBVA.
- [ ] **Swap file EVE-NG : 8 GB** + RAM appliance 8 GB.

> **Références** : chronogramme 6 semaines · Explications.pdf (DC, double FO, 8 sites) ·
> `03_configs_ios_eve_ng.md` · `04_scripts_preparation_windows.md`.
