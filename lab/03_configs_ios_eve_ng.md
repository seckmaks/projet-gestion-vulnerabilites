# Configurations complètes — Lab EVE-NG (VMS) — **variante « hôte 16 GB / EVE-NG 8 GB »**

> **Baseline = état réel confirmé** : le campus accède **directement** (non chiffré) au cloud.
> **Topologie réseau complète conservée** : R-CE-1 + R-CE-2, cœur **R-CE**, SW-TOR, SW-MET, SW-SMS.
> **Lab réel (capture du 20/09 — réseau communicant)** : le cœur est nommé **R-CE** (IOSv L3,
> 5 ports **G0/0…G0/4**) ; le scanner **GBVA** est un **nœud du lab** (Kali Linux + GVM/OpenVAS) branché sur
> **SW-TOR G0/2** (accès VLAN 10) ; le 2ᵉ bâtiment = **SW-SMS (SMS/Scolarité, VLAN 31)** ;
> **SRV-CLOUD a 1 NIC (Ubuntu : ens3)** → cœur **G0/2** (172.16.90.0/30, seule liaison du
> cloud) + **10.9.1.10/24 en IP secondaire** (données ; 2ᵉ liaison G0/4/ens4 supprimée le
> 20/09) ; ports SW-MET vérifiés : G0/2 Win-7, G0/3 IP-Phone, **G1/0** Camera.
> **Segment opérateur** : R-CE-1 = **10.0.0.3**, R-CE-2 = 10.0.0.2, GW = **10.0.0.254 (cloud NAT)** ;
> liaisons FO en **/30** (255.255.255.252) ; distance admin de la route flottante = **252**.
> **Périmètre** : bâtiment MET/SATA/ILAMEL (VLAN 30) + bâtiment SMS/Scolarité (VLAN 31) ;
> bâtiment Rectorat (VLAN 34) retiré ; après le Lab 4, seul le segment scanner (VLAN 10)
> est autorisé au cloud.
> **§8 « Remédiation Lab 4 »** : config IPSec + ACL à appliquer APRÈS le constat F-01.
> Images : Cisco IOSv **AdvEnterprise 15.7(3)M2 (development test — build ≈ 180 j, fallback IOSv
> 15.9(3)M stable si expiration)** — R-CE-1/2 + R-CE (cœur ; crypto/IKEv2 validés) ·
> Cisco IOSvL2 15.2 (eval) — SW-TOR, SW-MET, SW-SMS · **Ubuntu 20.04 non patché (SRV-CLOUD —
> seule VM cloud, 1 NIC ens3)** · Ubuntu léger headless
> (Camera) · **GBVA (Kali + GVM/OpenVAS, scanner)** · Windows 10 (Windows-10, Windows-10-1) · Windows 7
> (Win-7, Win-7-1) · VPCS (IP-Phone, IP-Phone-1).
> **Mots de passe masqués** dans ce dépôt (voir coffre-fort du lab).
> **Note de nommage** : dans EVE-NG, les ports s'affichent **`Gi0/x`** = **`G0/x`
> (GigabitEthernet0/x) dans IOS** — le cœur a **5 ports G0/0…G0/4** (**G0/4 = libre/réserve**
> depuis le 20/09 : 2ᵉ liaison SRV-CLOUD supprimée), SW-MET a **G1/0** (Camera) ; côté Ubuntu,
> SRV-CLOUD n'a qu'**une NIC `ens3`** (uplink + 10.9.1.10/24 en IP secondaire).
> Vérifier avec `show ip interface brief` (IOS) / `ip a` (Ubuntu).

---

## Rappel des ports — câblage réel du lab (conforme à la capture EVE-NG)

| Équipement | G0/0 | G0/1 | G0/2 | G0/3 | G0/4 | G1/0 |
|---|---|---|---|---|---|---|
| R-CE-1 | CLOUD-OP « Net » (10.0.0.3/24) | R-CE cœur (172.16.255.0/30) | — | — | — | — |
| R-CE-2 | CLOUD-OP « Net » (10.0.0.2/24) | R-CE cœur (172.16.254.0/30) | — | — | — | — |
| R-CE (cœur) | R-CE-1 | R-CE-2 | **SRV-CLOUD ens3** (172.16.90.0/30, SEULE liaison cloud, **lien plain**, 10.9.1.10 sec.) | trunk SW-TOR (V10, V30, V31) | — (réserve) | — |
| SW-TOR | trunk R-CE (V10, V30, V31) | trunk SW-MET (V30) | **GBVA e0 — accès VLAN 10** (scanner) | trunk SW-SMS (V31) | — | — |
| SW-MET | trunk SW-TOR (V30) | Windows-10 (PC-MET) | Win-7 (WIN-VULN-01) | IP-Phone (TEL-MET) | — | **Camera (CAM-MET)** |
| SW-SMS | trunk SW-TOR (V31) | Windows-10-1 | IP-Phone-1 | Win-7-1 | — | — |

> Ports **vérifiés sur le lab (20/09, réseau communicant)** : 5ᵉ port du cœur = **G0/4**
> (**libre/réserve** — 2ᵉ liaison SRV-CLOUD supprimée le 20/09) ; SW-MET : G0/2 = Win-7,
> G0/3 = IP-Phone, **G1/0 = Camera** ; SRV-CLOUD (Ubuntu) = **ens3** (seule NIC). Ports IOS
> affichés `Gi0/x` dans EVE-NG = `G0/x` dans IOS. Ports côté VMs (non gérés par IOS) :
> Windows-10 `e0` · Camera `e0` · IP-Phone `eth0` · Win-7 `e0` · GBVA `e0` (+ `e1` bridge hôte) ·
> SRV-CLOUD `ens3`.

---

## 1. R-CE-1 (Cisco IOSv) — liaison FO principale MADIYANA

```
!=================================================================
! R-CE-1 — Routeur CE — Liaison principale (central MADIYANA)
!=================================================================
version 15.9
service timestamps debug datetime msec
service timestamps log datetime msec
service password-encryption
!
hostname R-CE-1
!
username admin privilege 15 secret <masqué>
!
no ip domain-lookup
!
interface G0/0
 description >>> CLOUD-OP (Net - IP/MPLS SONATEL)
 ip address 10.0.0.3 255.255.255.0
 no shut
!
interface G0/1
 description >>> FO PRINCIPALE - R-CE (coeur)
 ip address 172.16.255.1 255.255.255.252
 no shut
!
ip route 0.0.0.0 0.0.0.0 10.0.0.254
ip route 172.16.0.0 255.255.0.0 172.16.255.2
!
ip ssh version 2
ip ssh time-out 60
ip ssh authentication-retries 3
no ip http server
logging buffered 16384 informational
banner motd #
 R-CE-1 - Liaison principale MADIYANA
 Acces restreint au personnel habilite.
#
!
line con 0
 logging synchronous
line vty 0 4
 login local
 transport input ssh
!
end
write memory
```

## 2. R-CE-2 (Cisco IOSv) — liaison FO secours NDAM MBACKÉ

```
!=================================================================
! R-CE-2 — Routeur CE — Liaison secours (central NDAM MBACKE)
!=================================================================
version 15.9
service timestamps debug datetime msec
service timestamps log datetime msec
service password-encryption
!
hostname R-CE-2
!
username admin privilege 15 secret <masqué>
!
no ip domain-lookup
!
interface G0/0
 description >>> CLOUD-OP (Net - IP/MPLS SONATEL)
 ip address 10.0.0.2 255.255.255.0
 no shut
!
interface G0/1
 description >>> FO SECOURS - R-CE (coeur)
 ip address 172.16.254.1 255.255.255.252
 no shut
!
ip route 0.0.0.0 0.0.0.0 10.0.0.254
ip route 172.16.0.0 255.255.0.0 172.16.254.2
!
ip ssh version 2
ip ssh time-out 60
ip ssh authentication-retries 3
no ip http server
logging buffered 16384 informational
banner motd #
 R-CE-2 - Liaison secours NDAM MBACKE
 Acces restreint au personnel habilite.
#
!
line con 0
 logging synchronous
line vty 0 4
 login local
 transport input ssh
!
end
write memory
```

## 3. R-CE (Cisco IOSv L3) — cœur : inter-VLAN + double accès opérateur + accès cloud DIRECT (sans VPN)

```
!=================================================================
! R-CE — Cœur du réseau (Rectorat) — IOSv L3
! Câblage réel : G0/0 R-CE-1 · G0/1 R-CE-2 · G0/2 SRV-CLOUD ens3 (SEULE liaison cloud :
!                uplink + données 10.9.1.0/24 en IP secondaire) · G0/3 SW-TOR · G0/4 = réserve
! BASELINE = état actuel : liaison cloud en clair (pas de VPN)
! (config IPSec : §8.2, à ajouter au Lab 4)
!=================================================================
version 15.9
service timestamps debug datetime msec
service timestamps log datetime msec
service password-encryption
!
hostname R-CE
!
username admin privilege 15 secret <masqué>
!
no ip domain-lookup
ip cef
!
!------------------- Interfaces L3 -------------------
interface G0/0
 description >>> FO PRINCIPALE - R-CE-1 (MADIYANA)
 ip address 172.16.255.2 255.255.255.252
 no shut
!
interface G0/1
 description >>> FO SECOURS - R-CE-2 (NDAM MBACKE)
 ip address 172.16.254.2 255.255.255.252
 no shut
!
interface G0/2
 description >>> LIEN CLOUD (UNIQUE) - SRV-CLOUD ens3 (ETAT ACTUEL : PAS DE VPN)
 ip address 172.16.90.1 255.255.255.252
 no shut
!
! G0/4 : LIBRE (reserve) - 2e liaison SRV-CLOUD supprimee (20/09)
!   (si ancienne config presente : no ip address + shutdown + write memory)
!
interface G0/3
 description >>> TRUNK - SW-TOR
 no shut
!
!------------------- Subinterfaces 802.1Q (inter-VLAN) -------------------
interface G0/3.10
 description MGMT - Plateforme VMS (GBVA, via SW-TOR G0/2)
 encapsulation dot1Q 10
 ip address 172.16.10.1 255.255.255.0
 no shut
!
interface G0/3.30
 description MET-SATA-ILAMEL
 encapsulation dot1Q 30
 ip address 172.16.30.1 255.255.255.0
 no shut
!
interface G0/3.31
 description SMS-SCOLARITE
 encapsulation dot1Q 31
 ip address 172.16.31.1 255.255.255.0
 no shut
!
!------------------- Routing -------------------
ip route 0.0.0.0 0.0.0.0 172.16.255.1
ip route 0.0.0.0 0.0.0.0 172.16.254.1 252
ip route 10.0.0.0 255.255.255.0 172.16.255.1
ip route 10.9.1.0 255.255.255.0 172.16.90.2
!
!------------------- Durcissement de base -------------------
ip ssh version 2
ip ssh time-out 60
ip ssh authentication-retries 3
no ip http server
no ip http secure-server
no ip bootp server
logging buffered 32768 informational
banner motd #
 R-CE - Cœur du réseau (Rectorat)
 Acces restreint au personnel habilite. Connexion suivie.
#
!
line con 0
 logging synchronous
line vty 0 4
 login local
 transport input ssh
!
end
write memory
```

## 4. SW-TOR (Cisco IOSvL2) — distribution 802.1Q (câblage réel)

```
!=================================================================
! SW-TOR — Distribution — câblage réel :
!   G0/0 trunk R-CE · G0/1 trunk SW-MET · G0/2 GBVA (VLAN 10) · G0/3 trunk SW-SMS
!   + SVI Vlan10 172.16.10.2/24 = management (scanner GCE, Lab 1)
!=================================================================
version 15.2
!
hostname SW-TOR
!
enable secret <masqué>
username admin privilege 15 secret <masqué>
!
vlan 10
 name MGMT
vlan 30
 name MET-SATA-ILAMEL
vlan 31
 name SMS-SCOLARITE
!
interface G0/0
 description >>> TRUNK - R-CE (coeur)
 switchport trunk encapsulation dot1q
 switchport trunk allowed vlan 10,30,31
 switchport mode trunk
 switchport nonegotiate
 no shut
!
interface G0/1
 description >>> FO - SW-MET
 switchport trunk encapsulation dot1q
 switchport trunk allowed vlan 30
 switchport mode trunk
 no shut
!
interface G0/2
 description >>> SCANNER - GBVA (plateforme GdV, VM du lab)
 switchport mode access
 switchport access vlan 10
 no shut
!
interface G0/3
 description >>> FO - SW-SMS (SMS / Scolarité)
 switchport trunk encapsulation dot1q
 switchport trunk allowed vlan 31
 switchport mode trunk
 no shut
!
interface Vlan10
 description >>> MGMT - SVI SW-TOR (scan GCE, Lab 1)
 ip address 172.16.10.2 255.255.255.0
 no shut
!
ip default-gateway 172.16.10.1
!
spanning-tree mode rapid-pvst
no cdp run
!
line con 0
 logging synchronous
line vty 0 4
 login local
!
end
write memory
```

## 5. SW-MET (Cisco IOSvL2) — accès bâtiment MET/SATA/ILAMEL (VLAN 30)

```
!=================================================================
! SW-MET — Accès bâtiment MET / SATA / ILAMEL (VLAN 30)
! Câblage réel (vérifié) : G0/0 trunk · G0/1 Windows-10 · G0/2 Win-7 ·
!                G0/3 IP-Phone · G1/0 Camera
!=================================================================
version 15.2
!
hostname SW-MET
!
enable secret <masqué>
username admin privilege 15 secret <masqué>
!
vlan 30
 name MET-SATA-ILAMEL
!
interface G0/0
 description >>> TRUNK - SW-TOR
 switchport trunk encapsulation dot1q
 switchport trunk allowed vlan 30
 switchport mode trunk
 no shut
!
interface G0/1
 description Accès poste Windows-10 (PC-MET)
 switchport mode access
 switchport access vlan 30
 no shut
!
interface G0/2
 description Accès poste Win-7 (WIN-VULN-01, legacy)
 switchport mode access
 switchport access vlan 30
 no shut
!
interface G0/3
 description Accès IP-Phone (TEL-MET, VPCS)
 switchport mode access
 switchport access vlan 30
 no shut
!
interface G1/0
 description Accès Camera (CAM-MET, Ubuntu, web :80)
 switchport mode access
 switchport access vlan 30
 no shut
!
interface Vlan30
 description >>> MGMT - SVI SW-MET (scan GCE, Lab 1)
 ip address 172.16.30.2 255.255.255.0
 no shut
!
ip default-gateway 172.16.30.1
!
spanning-tree portfast
no cdp run
!
line con 0
 logging synchronous
line vty 0 4
 login local
!
end
write memory
```

## 6. SW-SMS (Cisco IOSvL2) — accès bâtiment SMS / Scolarité (VLAN 31)

```
!=================================================================
! SW-SMS — Accès bâtiment SMS / Scolarité (VLAN 31)
! Câblage réel : G0/0 trunk SW-TOR · G0/1 Windows-10-1 · G0/2 IP-Phone-1 · G0/3 Win-7-1
!=================================================================
version 15.2
!
hostname SW-SMS
!
enable secret <masqué>
username admin privilege 15 secret <masqué>
!
vlan 31
 name SMS-SCOLARITE
!
interface G0/0
 description >>> TRUNK - SW-TOR
 switchport trunk encapsulation dot1q
 switchport trunk allowed vlan 31
 switchport mode trunk
 no shut
!
interface G0/1
 description Accès poste Windows-10-1
 switchport mode access
 switchport access vlan 31
 no shut
!
interface G0/2
 description Accès IP-Phone-1 (VPCS)
 switchport mode access
 switchport access vlan 31
 no shut
!
interface G0/3
 description Accès poste Win-7-1 (legacy)
 switchport mode access
 switchport access vlan 31
 no shut
!
interface Vlan31
 description >>> MGMT - SVI SW-SMS (scan GCE, Lab 1)
 ip address 172.16.31.2 255.255.255.0
 no shut
!
ip default-gateway 172.16.31.1
!
spanning-tree portfast
no cdp run
!
line con 0
 logging synchronous
line vty 0 4
 login local
!
end
write memory
```

---

## 7. Le cloud (1 VM) + postes + terminaux + segment scanner (baseline, sans VPN)

### 7.1 SRV-CLOUD (Ubuntu 20.04 non patché) — le cloud en UNE VM, 1 NIC (ens3 : uplink + données)

> Le cloud IaaS est représenté par **une seule VM** (SRV-CLOUD) : la passerelle managée de
> l'IaaS est **hors périmètre** du lab. Câblage réel (20/09, **1 NIC**) : **ens3 → cœur G0/2**
> (172.16.90.0/30, l'**unique** liaison du cloud) + segment métier **10.9.1.10/24 en IP
> secondaire sur la même NIC** (comme une config multi-IP Azure) — la 2ᵉ liaison
> (G0/4/ens4, 172.16.91.0/30) a été **supprimée**. Le terminus VPN du Lab 4 (Libreswan)
> est installé sur **ens3**.

```bash
# ens3 (SEULE NIC : uplink + données) — vérifier le nom réel avec `ip a` si l'image diffère
sudo ip addr add 172.16.90.2/30 dev ens3
# segment métier en IP secondaire sur la MÊME NIC
sudo ip addr add 10.9.1.10/24  dev ens3

# Routage : le campus est joignable via l'uplink
sudo ip route add 172.16.0.0/16 via 172.16.90.1

# Vérification : depuis R-CE ou un poste
ping 10.9.1.10    # OK : ACCÈS DIRECT EN CLAIR, SANS VPN = C'EST LE CONSTAT F-01
# (Libreswan non installé — ajouté au Lab 4, §8.3)
```

**État à maintenir** (constats V-04) : Ubuntu 20.04 **hors support standard (2025)**,
**aucune mise à jour appliquée**, page métier en **HTTP clair** (preuve tshark F-01),
SSH ouvert, **compte service faible** `svc-app / <masqué>`, compte `scan` (sudo) pour le
scan authentifié. **Script de préparation** : `04_scripts_preparation_windows.md` §4.

### 7.2 PC Windows 10 — Windows-10 (bât. MET) + Windows-10-1 (bât. SMS)

- **Réseau** : Windows-10 `172.16.30.10/24` (gw `172.16.30.1`) ; Windows-10-1 `172.16.31.11/24` (gw `172.16.31.1`).
- **État à maintenir** : WinUpdate **désactivé** (constat V-02 — et Win10 est **hors support depuis
  oct. 2025**), compte `student / Win10-Lab-2026` (mdp court = constat), droits administrateurs
  (lab : requis pour le scan authentifié ; en prod = constat « droits excessifs »), WinRM activé.
- **Script de préparation** : `04_scripts_preparation_windows.md` §1 (à exécuter sur les deux postes).

### 7.3 Windows 7 legacy — Win-7 (bât. MET) + Win-7-1 (bât. SMS)

**Win-7** (`172.16.30.50`) — machine « très vulnérable » (cible d'exploitation optionnelle, Lab 2) :

```
Windows 7 SP1, NON patché, avec :
  - SMBv1 (natif Win7) : RequireSecuritySignature = 0  (exploit MS17-010 possible)
  - Compte admin/admin
  - RDP activé
  - Antivirus désactivé
  - WinUpdate désactivé, WinRM ouvert
→ Cible d'exploitation optionnelle (Lab 2) : EternalBlue / MS17-010 via Metasploit.
```
**Script de préparation** : `04_scripts_preparation_windows.md` §3.
**(Bât. SMS) Win-7-1** (`172.16.31.10`) : Win7 SP1, WinUpdate off, `olduser / Win7-Lab-2026`
(script §2 du 04) — 2ᵉ machine EOL pour étendre le constat V-01 au 2ᵉ bâtiment.

### 7.4 Terminaux — IP-Phone / IP-Phone-1 (VPCS) + Camera (Ubuntu, page web :80)

**IP-Phone** — console du **VPCS** (Virtual PC natif d'EVE-NG, aucune image à importer),
une ligne par machine :
```
ip 172.16.30.60/24 172.16.30.1     # IP-Phone (bât. MET, port SW-MET G0/3)
ip 172.16.31.12/24 172.16.31.1     # IP-Phone-1 (bât. SMS, port SW-SMS G0/2)
```

**Vérifications VPCS** (console IP-Phone, une fois le réseau up — complément des vérifs GCE, 04 §8) :
```
ipconfig /all                        # MAC + IP du terminal (utile pour l'inventaire / câblage)
ping 172.16.30.1 -n 3                # passerelle (SVI R-CE)
http get http://172.16.30.11/        # page caméra par défaut Camera (preuve V-05 « depuis le bâtiment »)
http get http://10.9.1.10/app/       # page « données métier » joignable EN CLAIR (preuve F-01)
```

**Camera** (`172.16.30.11`) — **Ubuntu serveur headless** (0,5 GB, port VM `e0`, câblée sur
**SW-MET G1/0**) simulant une caméra IP : page web **par défaut sur le port 80**
(interface « constructeur ») + compte par défaut **`admin/1234`** = constat **V-05** décelable
au scan.
**Script de préparation** : `04_scripts_preparation_windows.md` §5.

> Représentation IP des IOCs ; constats « mdp par défaut constructeur » (V-05/V-06) = **inventaire**.

### 7.5 VULNÉRABILITIES IOS À PLANTER (après validation de la baseline RÉSEAU §1–§6 — donc AVANT tout scan GCE)

> **Ordre des opérations** : « baseline » = la configuration réseau fonctionnelle (§1–§6 : adressage,
> routage, SVI de management), validée par les pings — PAS le rapport de scan. Le bloc ci-dessous se pose
> **avant la baseline de scan P0** (exigé par 06 §5, prérequis « avant P0 ») et donc **avant la relance
> élargie Full & fast** : le rapport « avant » doit contenir telnet 23/tcp, SNMP v1 `public`/`private`
> et le compte `guest`. Le bloc « Après remédiation » ne s'applique qu'au **Lab 1 (obj 5)**.

**Sur R-CE-1, R-CE-2, R-CE (cœur), SW-TOR, SW-MET, SW-SMS :**
```
! --- Telnet réactivé ---
line vty 0 4
 transport input telnet
!
! --- SNMP v1/v2 en clair ---
snmp-server community public ro
snmp-server community private rw
!
! --- Compte faible ---
username guest privilege 5 secret guest
```
**Après remédiation (Lab 1) :**
```
no snmp-server community public
no snmp-server community private
no username guest
!
snmp-server group scanv3 v3 priv
snmp-server user gvm-scan scanv3 auth sha <masqué> priv aes 128 <masqué>
line vty 0 4
 transport input ssh
```

> **⚠️ Pièges de ce bloc (retours du lab 05/10)** : ① coller le bloc d'un coup dans la console EVE-NG
> **coupe les lignes longues** (`% Invalid input detected at '^' marker` + écho avec `$` en début de
> ligne) → coller **ligne par ligne** (ou retaper à la main la ligne `snmp-server user gvm-scan …`,
> qui est correcte) ; le `[confirm]` de `no username guest` : **Entrée** suffit. Vérification :
> `show snmp user` (gvm-scan, SHA/AES-128) + `show run | include snmp-server user`, puis **`write memory`**.
> ② **ORDRE** : ce bloc ne se pose qu'**APRÈS** le scan L1 « AVANT » (credential SNMPv1 `public`) —
> sinon plus de preuve authentifiée V-03 sur l'équipement. Restauration de l'état AVANT si besoin :
> réexécuter le bloc « à planter » ci-dessus (community public/private + username guest, telnet vty).
(le profile `gvm-scan` = credential CISCO dans GBVA pour les re-scans ;
SW-TOR / SW-MET / SW-SMS sont scannés **par GCE** via leurs SVI de management
172.16.10.2 / 172.16.30.2 / 172.16.31.2 — §4-§6 ; `show running-config` en complément console)

### 7.6 SEGMENT SCANNER (VLAN 10) — GBVA, nœud du lab branché sur SW-TOR G0/2

**Principe (lab réel)** : le scanner **GBVA** (Kali Linux + GVM/OpenVAS) est un **nœud du lab EVE-NG**,
branché en **accès sur SW-TOR G0/2 (VLAN 10)**, routé par la subinterface `G0/3.10` du cœur
(172.16.10.1). Plus de bridge CLOUD-SCAN ni de réseau hôte-only : le VLAN 10 reste le segment
management isolé, **fidèle au réel**.

**1) NIC de GBVA (deux interfaces)**

a. **e0 — le lab** (câblée sur SW-TOR G0/2) : IP statique **172.16.10.10/24**,
   passerelle **172.16.10.1** (SVI R-CE).
b. **e1 — ajout d'une 2ᵉ NIC** → cloud **bridge** (adaptateur LAN physique de l'hôte) vers
   **192.168.1.0/24** : IP **statique (ex. 192.168.1.200/24**, hors plage DHCP de la box),
   passerelle = la box (ex. **192.168.1.1**), DNS = 192.168.1.1 — c'est elle qui donne
   l'**internet** à GBVA (sync quotidienne des NVT) et permet d'ouvrir le **GSA depuis l'hôte**
   sur `https://192.168.1.200:9392`.
   *(Alternative : cloud NAT si le bridge pose problème — le GSA se reachera alors via le NAT.)*

**2) Routes GBVA** (ou via l'UI Greenbone → Routes) :
```bash
# NIC e0 (lab) : le lab est joignable via le SVI VLAN 10
sudo ip route add 172.16.0.0/16 via 172.16.10.1
# NIC e1 (LAN) : passerelle par défaut = la box (internet / NVT)
sudo ip route add default via 192.168.1.1
```

**3) Vérifications (baseline)** :
```bash
# depuis GBVA :
ping 172.16.10.1        # SVI R-CE (VLAN 10)
ping 172.16.30.10       # Windows-10 — OK
ping 172.16.30.11       # Camera — OK
ping 172.16.31.11       # Windows-10-1 — OK
ping 10.9.1.10          # SRV-CLOUD — OK = EXPOSITION ATTENDUE (constat F-01)
ping 192.168.1.1        # box (internet NVT) — OK
# depuis R-CE (cœur) :
ping 172.16.10.10       # GBVA — OK (le DC « voit » le scanner de son VLAN 10)
```

> **Pièges classiques** : port SW-TOR G0/2 mal mis en accès VLAN 10 ; subinterface G0/3.10 down
> (`show ip int brief`) ; NIC e1 de GBVA absente (ajouter la 2ᵉ NIC dans les propriétés du nœud) ;
> IP statique 192.168.1.x en conflit avec la plage DHCP de la box.

---

## 8. REMÉDIATION LAB 4 — Tunnel IPSec site-to-site + ACL d'isolation (APRÈS le constat F-01)

### 8.1 Démonstration avant remédiation (Lab 3) — à capturer pour le mémoire

```
# Depuis Windows-10 (bâtiment) :
ping 10.9.1.10         # => succès : le campus a accès direct au cloud (exposition)

# Depuis GBVA :
ping 10.9.1.10         # => succès : le scanner aussi (direct)

# Sur SRV-CLOUD (capture sur l'UPLINK ens3, l'unique liaison du cloud) :
sudo tshark -i ens3 -f "net 10.9.1.0/24" -c 20   # => trafic en clair (page métier HTTP, ping…)
```

### 8.2 Côté R-CE (cœur, IOSv) — ajouter à la config baseline

```
!------------------- IPSec site-to-site (ajouté au Lab 4) -------------------
crypto ikev2 keyring CLOUD
 pre-shared-key address 172.16.90.2 key <masqué>
!
crypto ikev2 policy 10
 hash sha256
 authentication pre-share
 group 14
!
crypto ipsec transform-set TS-CLOUD esp-aes 256 esp-sha256-hmac
!
crypto ipsec profile CLOUD-PROF
 set transform-set TS-CLOUD
 set ikev2 keyring CLOUD
!
interface Tunnel90
 description TUNNEL IPSec -> CLOUD (SRV-CLOUD, donnees)
 tunnel source G0/2
 tunnel destination 172.16.90.2
 tunnel mode ipsec ipv4
 tunnel protection ipsec profile CLOUD-PROF
 ip address 10.9.255.1 255.255.255.252
!
! Route cloud via le tunnel (remplace la route plain du §3)
no ip route 10.9.1.0 255.255.255.0 172.16.90.2
ip route 10.9.1.0 255.255.255.0 10.9.255.2
!
!------------------- ACL d'isolation : qui peut accéder au cloud ? -------------------
! Politique cible : VLAN 10 (scanner) UNIQUEMENT
ip access-list extended CLOUD-ACL
 remark Politique d'acces au cloud - Lab 4
 permit ip 172.16.10.0 0.0.0.255 10.9.1.0 0.0.0.255
 deny ip any 10.9.1.0 0.0.0.255 log
!
interface Tunnel90
 ip access-group CLOUD-ACL out     ! out = filtre les paquets allant VERS le cloud
```

### 8.3 Côté SRV-CLOUD (Libreswan sur Ubuntu 20.04 — tunnel installé sur ens3, l'uplink)

```bash
sudo apt update && sudo apt install -y libreswan

# Sous-réseau tunnel (le cœur route 10.9.1.0/24 via 10.9.255.2)
sudo ip addr add 10.9.255.2/30 dev ens3

# /etc/ipsec.conf  (fin de fichier)
config setup
    uniqueids=no

# /etc/swanctl/conf.d/site.conf
connections {
  campus {
    version = 2
    local  = {
      auth = "psk"
      id   = "172.16.90.2"
    }
    remote = {
      auth = "psk"
      id   = "172.16.90.1"
    }
    children = {
      net = {
        local    = "10.9.1.0/24"
        remote   = "172.16.0.0/16"
        life_time  = 3h
        rekey_time = 2h30m
      }
    }
  }
}

# /etc/ipsec.d/psk.txt  (PSK identique au keyring IOS)
cat > /etc/ipsec.d/psk.txt <<EOF
172.16.90.1 172.16.90.2 : <masqué>
EOF
chmod 600 /etc/ipsec.d/psk.txt

sudo systemctl restart ipsec
ipsec statusall    # => child 'campus#1' INSTALLED, selectors 10.9.1.0/24 <> 172.16.0.0/16

# Retour trafic vers le campus par le tunnel
sudo ip route replace 172.16.0.0/16 via 10.9.255.1
```
> **Note pratique** : les dépôts d'Ubuntu 20.04 sont en archive depuis l'EOL (2025) — si
> `apt install libreswan` échoue, prendre le paquet `.deb` Libreswan correspondant, ou réaliser
> le Lab 4 **après** le scan initial (les constats V-04 sont alors déjà capturés par GBVA).

### 8.4 Preuves après remédiation (Lab 4)

```
# 1) Isolation : depuis Windows-10 (VLAN 30, non autorisé)
ping 10.9.1.10       # => REFUS (timeout) + log dans la console IOS (show log | include CLOUD-ACL)
#    depuis GBVA (VLAN 10, autorisé) : ping OK (via tunnel)

# 2) Chiffrement : tshark sur l'uplink de SRV-CLOUD (le tunnel est sur ens3)
sudo tshark -i ens3 -f "esp" -c 10    # => uniquement des paquets ESP sur l'unique liaison :
#    le trafic métier (10.9.1.0/24) est désormais chiffré dans le tunnel

# 3) Session IPSec
show crypto ipsec sa          # (R-CE cœur) => session UP, SPI, débit
ipsec statusall               # (SRV-CLOUD) => INSTALLED
```

---

## 9. Matrice de vérification (à copier dans le mémoire)

| # | Test | Baseline (avant) | Après Lab 4 |
|---|---|---|---|
| 1 | Campus interne (`ping 172.16.30.11` depuis Windows-10) | OK | OK |
| 2 | Internet simulé (`ping 10.0.0.254` depuis Windows-10) | OK via CE-1 | OK |
| 3 | **Cloud depuis le bâtiment** (`ping 10.9.1.10` depuis Windows-10) | **OK — exposition (F-01)** | **REFUS (ACL)** |
| 4 | Cloud depuis le scanner (GBVA, VLAN 10) | OK en clair | OK via tunnel (ESP) |
| 5 | `tshark` uplink ens3 SRV-CLOUD (172.16.90.0/30) | Trafic **clair** (HTTP métier 10.9.1.0/24 sur la même liaison) | Uniquement des paquets **ESP** |
| 6 | `show ip route` (cœur) | `10.9.1.0/24 via 172.16.90.2` | `10.9.1.0/24 via 10.9.255.2` (tunnel) |
| 7 | **Basculement** (shut **G0/0 du cœur R-CE**) — Lab 6 | défaut via 172.16.254.1, flux scanner maintenus | idem, session IPSec non impactée |
| 8 | Scan GBVA du cloud | direct | via tunnel (mêmes constats, accès sécurisé) |

---

## 10. Dépannage courant

| Symptôme | Cause probable / correctif |
|---|---|
| GBVA ne voit pas le lab | Port **SW-TOR G0/2** mal mis en accès **VLAN 10** ; subinterface G0/3.10 down (`show ip int brief`) ; GBVA e0 = 172.16.10.10/24 gw **172.16.10.1** ; route `172.16.0.0/16 via 172.16.10.1` présente sur GBVA ; tester : 172.16.10.1 → 172.16.30.10 |
| VLAN 30/31 ne passe pas sur un trunk | `trunk allowed vlan` incomplet (R-CE G0/3 : 10,30,31 ; SW-TOR G0/0 : 10,30,31 ; G0/1 : 30 ; G0/3 : 31) ; subinterfaces G0/3.x down |
| SVI SW-MET (172.16.30.2) / SW-SMS (172.16.31.2) injoignable depuis GBVA (ping .30.1/.31.1 OK) | Le switch reçoit le ping mais ne sait pas **répondre** hors de son /24. **Cause confirmée au lab (05/10) : `ip routing` activé sur l'IOSvL2** → `ip default-gateway` **ignorée** → correctif : `no ip routing` puis `ip default-gateway 172.16.30.1` / `172.16.31.1` (variante : garder `ip routing` + `ip route 0.0.0.0 0.0.0.0 <gw>`) ; autres causes : `ip default-gateway` **absente** (03 §5/§6), SVI **down/down** → VLAN absent (`show vlan brief`) ou trunk non formé (`show interfaces trunk`) ; test : depuis le switch `ping 172.16.30.1` (marche sans gateway) puis `ping 172.16.10.10` (échoue sans gateway) |
| GBVA sans internet (NVT) / GSA injoignable de l'hôte | NIC **e1** absente (ajouter la 2ᵉ NIC du nœud GBVA → cloud bridge hôte) ; IP statique **192.168.1.200/24** hors plage DHCP ; gw + DNS **192.168.1.1** (test : `ping 192.168.1.1`) ; GSA : `https://192.168.1.200:9392` |
| SRV-CLOUD (10.9.1.10) injoignable | **ens3** (seule NIC) = 172.16.90.2/30 + **10.9.1.10/24 en IP secondaire** (`ip addr show ens3`) ; route retour `172.16.0.0/16 via 172.16.90.1` ; depuis le cœur : `ping 172.16.90.2` puis `ping 10.9.1.10` |
| Ping GBVA → lab OK mais scan vide | WinRM/SSH inactif sur la cible, credential erroné, ou pare-feu — tester `Test-WSMan` (Windows) / `ssh scan@10.9.1.10` |
| Scan GBVA très lent | RAM du nœud GBVA ; scans cibles restreints (Fast Scan d'abord) ; vérifier la RAM/swap d'EVE-NG |
| Windows lent / freeze dans EVE-NG | qcow2, 1 vCPU / 1,5 GB (Win10) / 1 GB (Win7), `q35`, animations off ; **ne pas tout démarrer d'un coup** ; vérifier le swap d'EVE-NG |
| Image Windows expirée (eval) | Réimporter une image fraîche (limite ~180 j) — à anticiper avant la soutenance |
| MS17-010 non détecté sur Win-7 | Vérifier `RequireSecuritySignature=0` (LanmanServer\Parameters) et service `Server` actif |
| Camera (port 80) injoignable | `systemctl status nginx` sur Camera ; IP 172.16.30.11/24 ; port d'accès SW-MET **G1/0** en VLAN 30 |
| Tunnel IPSec ne s'installe pas | PSK différent des 2 côtés ; `id` IKEv2 incohérent ; hash/group différents (sha256 / group 14) ; 10.9.255.2/30 manquant sur SRV-CLOUD |
| Ping tunnel OK mais app ko | MTU IPsec → `ip tcp adjust-mss 1300` sur Tunnel90 |
| Ping 0 % entre 2 routeurs alors que les 2 ports sont **up/up** | Sur IOSv, les ports virtuels **ne détectent pas le carrier** : `up/up` même sans câble → vérifier le **wire dans EVE-NG** (clic sur le wire : extrémités R-CE-1 G0/1 ↔ R-CE G0/0 ?) + `show cdp neighbors` (CDP ne répond que si le wire existe) ; le cas échéant, delete wire + redessiner |
| `show ip route static` vide (« Gateway of last resort is not set ») | La ligne `ip route 0.0.0.0 0.0.0.0 …` n'a pas été collée → la ressaisir (`ip route 0.0.0.0 0.0.0.0 10.0.0.254` sur R-CE-1/2 ; `via 172.16.255.1` sur R-CE) puis `write memory` |
| `ping 10.0.0.254` en 0 % alors que le /24 est connecté | Le .254 est **directement connecté** (aucune route n'est en cause) : c'est le **cloud** qui ne répond pas → vérifier le **Type du cloud dans EVE-NG = NAT** avec IP 10.0.0.254/24 (un cloud **Empty** ne répond à rien ; le label texte « GW: 10.0.0.254 » ne configure rien) |
| Route flottante ne bascule pas (Lab 6) | `distance 252` absente sur la 2ᵉ route ; **le `shut` doit être sur G0/0 du cœur R-CE** (pas sur R-CE-1 : les ports virtuels ne propagent pas la rupture de liaison) ; vérifier `show ip route` avant/après |
| IOSvL2 refuse `switchport trunk encapsulation dot1q` | Utiliser l'image IOSvL2 ; sinon supprimer la commande (dot1q par défaut) |
