# Guide des labs — Déroulé pratique de la démonstration VMS (EVE-NG)

> Ce document est le **guide d'exécution** du mémoire : il détaille les 7 « campagnes »
> (Phase 0 + Labs 1→6) **plus 3 ateliers d'industrialisation (obj. 4-7 : scoring RBVM, workflow
> automatisé, non-régression)**, leurs commandes exactes, les résultats attendus et **les preuves à
> capturer** pour chaque lab. Les labs sont la démonstration pratique de la méthodologie
> présentée dans le livrable Semaine 1 (cycle de vie de la GdV : identification → évaluation
> → remédiation → vérification), appliquée au lab EVE-NG (15 nœuds + 2 clouds, scanner GBVA = GCE
> **dans le lab**, EVE-NG 8 GB + swap 8 GB, hôte 16 GB).

---

## 1. L'idée générale

Les labs ne sont pas des exercices : c'est la **série de démonstrations du mémoire**. Chaque lab
applique le cycle de gestion des vulnérabilités à **une famille d'actifs différente**, avec
toujours les mêmes 5 gestes :

1. **Qu'est-ce que j'ai ?** (inventaire)
2. **Qu'est-ce qui cloche ?** (scan)
3. **Qu'est-ce qui est le plus grave, et pourquoi ?** (priorisation)
4. **Je corrige quoi ?** (remédiation)
5. **C'est corrigé ?** (re-scan = preuve)

## 2. Vue d'ensemble — 7 labs + 3 ateliers (objectifs 4-7)

| Lab | Cible | Ce que vous faites | Constat | Preuve à capturer |
|---|---|---|---|---|
| **P0** | Tout le lab | Nmap depuis GBVA (GCE) sur 172.16.10.0/24 + 172.16.30.0/24 + 172.16.31.0/24 + 10.9.1.0/24 + IP IOS ; recensement des 15 nœuds + 2 clouds ; **1er scan GCE (découverte, non authentifié)** | « Je sais ce que je possède » (inventaire + baseline brute) | rapport Nmap + tableau d'inventaire + **rapport GCE baseline** |
| **L1** | Réseau (IOS) | Scan GCE avec credentials → durcissement (telnet off, SNMPv3, guest supprimé) → re-scan | V-03 : telnet, SNMP v1 `public`, compte `guest` | rapports GCE avant / après |
| **L2** | Windows + cloud + caméra | Full Scan authentifié (SMB (NTLM), SSH) → corrections → re-scan | V-01 Win7 EOL + MS17-010 (critique), V-02 Win10, V-04 Ubuntu périmé, V-05 caméra :80 | rapports GCE + Nmap SMBv1 avant/après |
| **L3** | **L'architecture** | `ping 10.9.1.10` depuis Windows-10 = succès ; tshark sur l'unique liaison (ens3 SRV-CLOUD) = clair | **F-01 : cloud accessible sans VPN** (fait de terrain) | capture tshark + recommandation écrite |
| **L4** | Remédiation de F-01 | Tunnel IPSec (IKEv2 + Libreswan) + ACL « scanner uniquement » | ping Windows-10 refusé + loggé ; GCE OK via tunnel ; ESP seul | `show crypto ipsec sa` + matrice avant/après |
| **L5** | Exploitation continue | Planification (fast quotidien, full hebdo), exports PDF, **KPI obj 6 (MTTR, top 5, âge moyen, taux auto)** | Le VMS tourne en « mode production » | dashboard + rapports exportés |
| **L6** | Disponibilité | `shut` **G0/0 du cœur R-CE** (coupure FO principale côté DC) → basculement R-CE-2 | La plateforme VMS tient la coupure | `show ip route` avant/après + scan pendant la coupure |
| **A** | **Obj 4 (S3)** — registre de risques | Grille de criticité métier + `Risque = CVSS × Criticité` sur les constats L1/L2 | Les priorités suivent **le métier**, pas seulement le CVSS (RBVM) | grille remplie + matrice P1/P2/P3 |
| **B** | **Obj 5 (S4)** — workflow | DefectDojo → ticket **Jira** auto (P1) → alerte **Mattermost** → playbook **Ansible** → clôture au re-scan | La remédiation devient un **processus outillé** | captures DefectDojo/Jira/Mattermost + run Ansible |
| **C** | **Obj 7 (S6)** — non-régression | cron **`gvm-cli`** + scan post-remédiation + finding **Re-opened** + contrôles hardening | Les failles corrigées **ne reviennent pas** | cron + capture Re-opened + KPI final |

## 3. L'ordre — un récit d'ingénieur sécurité

L'enchaînement raconte une histoire, c'est ce qui fait la qualité du mémoire :

- **P0** → « Je connais mon patrimoine » *(identification)*
- **L1 / L2** → « Je sais ce qui est cassé, et c'est trié par gravité » *(évaluation)*
- **L3** → « J'ai trouvé un vrai problème d'architecture : pas de VPN » *(le constat terrain — cœur du projet)*
- **L4** → « Je le corrige, et je prouve avant/après » *(remédiation + vérification)*
- **L5** → « Ce n'est pas ponctuel : processus continu avec KPI »
- **L6** → « Même si une fibre coupe, la plateforme tient » *(disponibilité)*
- **A** → « Les priorités suivent le métier, pas juste le score » *(RBVM — obj 4)*
- **B** → « La correction devient un workflow : ticket, alerte, playbook » *(obj 5)*
- **C** → « On prouve que ça tient dans le temps : non-régression » *(obj 7)*

**Dépendances à respecter** :
- **P0 d'abord** — pas d'inventaire, pas de scan.
- **L4 seulement après L3** — on ne remédie que ce qui a été formellement constaté (F-01).
- **L6 tout dernier** — la coupure de fibre est le choc final : on montre que tout ce qui a été
  installé avant (VPN, ACL, scanner) tient.
- **L5 est indépendant** (configuration de planification) — peut se faire en parallèle de L4.
- **A (obj 4) après L1 + L2** — on ne priorise que des constats réellement remontés.
- **B (obj 5) après A** — les seuils de tickets utilisent la grille de criticité ; se fait en
  parallèle de L3/L4 (il tourne hors EVE-NG).
- **C (obj 7) en tout dernier** — la non-régression vérifie TOUTES les remédiations
  (L1, L2, L4, Atelier B).

## 4. Correspondance avec le chronogramme (6 semaines)

| Semaine | Objectif du mémoire | Labs associés |
|---|---|---|
| 1 | Obj. 1 — cycle de vie ; Obj. 2 — comparaison des scanners | *(livrable 1 déjà rendu)* |
| 2 | Obj. 3 — déploiement du scanner + 1er scan | **P0** (inventaire Nmap + 1er scan GCE découverte) |
| 3 | Obj. 4 — scoring et priorisation (CVSS + contexte) | **L1 + L2 + Atelier A** (matrice RBVM) |
| 4 | Obj. 5 — workflow de remédiation | **L3 + L4 + Atelier B** (DefectDojo / Jira / Mattermost / Ansible) |
| 5 | Obj. 6 — tableau de bord KPI | **L5** (dashboard DefectDojo + MTTR) |
| 6 | Obj. 7 — évaluation finale | **L6 + Atelier C** (non-régression) + rapport final |

---

## 5. Prérequis (une seule fois, avant P0)

- [ ] Lab up : baseline configurée (`03_configs_ios_eve_ng.md` §1–§6) et vulnérabilités IOS plantées (§7.5).
- [ ] VMs préparées : `04_scripts_preparation_windows.md` §1–§5 (Windows-10, Win-7, SRV-CLOUD, Camera).
- [ ] GBVA (GCE, **nœud du lab**) up : e0 = 172.16.10.10/24 (gw 172.16.10.1, route `172.16.0.0/16
      via 172.16.10.1`), e1 → cloud **bridge** LAN hôte (192.168.1.200/24 statique, NVT/internet),
      GSA via `https://192.168.1.200:9392` ; mot de passe admin **faible planté** (V-07) puis changé.
- [ ] Outillage **obj 5/7** prêt (Ateliers B/C — détail en **04 §10-§11**) : **DefectDojo** + **Mattermost**
      (Docker sur l'**hôte**), site **Jira Cloud** + token API, **Ansible** + **gvm-tools** sur GBVA.
- [ ] Chaîne de test (depuis GBVA) : `ping 172.16.10.1` OK · `ping 172.16.10.2` OK (SVI SW-TOR) · `ping 172.16.30.10` OK ·
      `ping 172.16.30.11` OK (Camera) · `ping 10.9.1.10` **OK (exposition attendue)**.
- [ ] Dossier « preuves » créé : **un sous-dossier par lab** (P0, L1… L6) — les captures horodatées
      vont y atterrir ; elles alimenteront directement le mémoire.

---

## 6. Détail de chaque lab

### Phase 0 — Baseline et inventaire

**But** : répondre à « que possède l'entreprise ? » — prérequis absolu de toute GdV.

1. Depuis GBVA (GCE), Nmap sur le campus et le cloud :
   ```bash
   nmap -sV …                        # nmap est préinstallé sur Kali
   nmap -sV 172.16.10.0/24           # VLAN 10 mgmt : SVI SW-TOR (.2) + GBVA
   nmap -sV 172.16.30.0/24           # bât. MET : Windows-10, Win-7, IP-Phone, Camera, SVI SW-MET (.2)
   nmap -sV 172.16.31.0/24           # bât. SMS : Windows-10-1, Win-7-1, IP-Phone-1, SVI SW-SMS (.2)
   nmap -sV 10.9.1.10 172.16.90.2    # cloud : SRV-CLOUD (uplink + données, même NIC ens3)
   nmap -sV 172.16.255.1 172.16.254.1 172.16.30.1   # équipements IOS joignables
   ```
2. Sur chaque équipement (console) : `show version` + `show ip interface brief`.
3. Remplir le **tableau d'inventaire** : actif · type · IP · OS/version · rôle · criticité métier ·
   exposition (interne / cloud ; la colonne « criticité métier » 1-5 sera formalisée par la
   **grille de l'objectif 4** — pour P0, une estimation suffit). L'actif SRV-CLOUD y figure sous
   son double rôle (passerelle + données — seule VM du cloud).
4. De l'inventaire naît le **règlement de priorité** (ordre de passage des scans) :
   1) SRV-CLOUD (données de l'entreprise) → 2) Win-7 / Win-7-1 (Win7 EOL) → 3) équipements IOS →
   4) Windows-10 / Windows-10-1 → 5) Camera / IP-Phone.
5. **Premier scan GCE (découverte, non authentifié)** — c'est le **livrable « 1er scan » de
   l'objectif 3** : une Task OpenVAS couvrant les **3 familles d'actifs** — réseau (172.16.255.1,
   172.16.254.1, 172.16.30.1 + SVI mgmt 172.16.10.2 / 172.16.30.2 / 172.16.31.2), postes
   (172.16.30.0/24 + 172.16.31.0/24), serveur (10.9.1.10). Résultat attendu : les futurs constats
   se dessinent déjà (23/tcp telnet, SNMP v1, SMBv1, HTTP :80 caméra/cloud). **Exporter le rapport
   PDF baseline** : c'est la référence « avant » de tout le mémoire — l'objectif 4 (S3) la notera
   et la priorisera.

**Preuves** : sorties Nmap (5), `show version` des 6 équipements, tableau d'inventaire rempli,
**rapport GCE baseline exporté (PDF)**.

---

### Lab 1 — Audit des équipements réseau (IOS)

**But** : identifier → remédier → vérifier sur la couche réseau (constat V-03).

1. **État initial** (vulnérabilités déjà plantées, 03 §7.5) : telnet actif,
   `snmp-server community public ro` / `private rw`, compte `guest`.
2. **Scan authentifié GCE** :
   - Credentials GSA (« Credentials » → « + », Type SNMP) : **CISCO-SNMPv1-public** (scan AVANT) :
     **GSA 25.04 exige login ET password non vides** (erreurs « credential_login/password was
     invalid » sinon) : **Community** `public` · **Username** `public` · **Password** `public` (bidon,
     inutilisé en v1/v2c) · Auth/Privacy vides (sinon bascule en v3). Refus persistant → créer par GMP :
     `sudo -u _gvm gvm-cli --gmp-username admin --gmp-password admin socket
     --socketpath /run/gvmd/gvmd.sock -X '<create_credential><name>CISCO-SNMPv1-public</name>
     <type>snmp</type><login>public</login><community>public</community></create_credential>'`
     (le credential apparaît alors dans GSA). Plan B : credential SSH admin/<masqué> + le SNMP
     public reste détecté par les NVT « communautés par défaut » du Full scan. À créer aussi dès maintenant : **CISCO-SNMPv3-gvm-scan** (re-scan
     APRÈS) : Username `gvm-scan` · Password `<masqué>` · **Auth** SHA-1 · **Privacy password**
     `<masqué>` · **Privacy algorithm** AES-128 · communauté vide. Option complémentaire AVANT :
     SSH `admin` / `<masqué>`. Pièges : communauté sensible à la casse ; SNMP = UDP 161 → rester
     en politique **Full Scan** ; capturer les 2 credentials pour preuves/L1.
   - Cibles (les **6 équipements IOS**, tous joignables depuis GBVA) : R-CE-1 (172.16.255.1),
     R-CE-2 (172.16.254.1), R-CE cœur (172.16.30.1), SW-TOR (SVI **172.16.10.2**), SW-MET
     (SVI **172.16.30.2**), SW-SMS (SVI **172.16.31.2**) — SVI de management ajoutés à la
     baseline (03 §4-§6) pour rendre les switchs **scannables** (exigence obj. 3).
   - `show running-config` sur console (SW-TOR, SW-MET, SW-SMS) : **complément de preuve** —
     vérification à l'œil des mêmes vulnérabilités plantées, en plus du scan.
   - Politique : **Full Scan** → les constats attendus : port 23 ouvert (telnet), SNMP v1 en clair,
     compte faible `guest`.
3. **Remédiation** sur les 6 équipements IOS (03 §7.5, bloc « Après remédiation ») :
   `transport input ssh` · suppression des communautés SNMP v1 · `no username guest` ·
   migration **SNMPv3** (profile `gvm-scan` : auth SHA + priv AES-128, `<masqué>`).
4. **Re-scan GCE** avec la **nouvelle credential CISCO SNMPv3** → les constats telnet/SNMPv1/guest
   doivent avoir **disparu**.

**Preuves** : rapport GCE avant (constats V-03) · `show run | section snmp` après ·
rapport GCE après (constats disparus).

---

### Lab 2 — Parc Windows, serveur cloud et terminaux

**But** : les constats « postes et données » (V-01, V-02, V-04, V-05) + option exploitation.

1. **Credentials GCE déjà créées** (04 §6.5) : type **SMB (NTLM)** `student / Win10-Lab-2026` (Windows-10),
   `admin / admin` (Win-7), `olduser / Win7-Lab-2026` (Win-7-1), SSH `scan / <masqué>`
   (SRV-CLOUD). Camera et Windows-10-1 : **aucun credential** (constats au scan non authentifié).
2. **Full Scan GCE** sur 172.16.30.0/24 + **172.16.31.0/24** (les 2 bâtiments) + 10.9.1.10
   (avec credentials) → constats attendus :

   | Actif | Constat | Sévérité |
   |---|---|---|
   | Win-7 | Win7 **EOL 2020** non patché, **MS17-010 / SMBv1**, `admin/admin`, RDP | **Critique** |
   | Windows-10 | Win10 **EOL 10/2025**, WinUpdate off, mdp court, droits excessifs, WinRM non chiffré | Moyenne |
   | Win-7-1 (bât. SMS) | **Même profil que V-01** (EOL + SMBv1) — authentifié (WIN-TC2) | **Critique** |
   | Windows-10-1 (bât. SMS) | Win10 EOL — constat non authentifié | Moyenne |
   | SRV-CLOUD | Ubuntu **20.04 EOL (2025)** non patché, `svc-app` faible, SSH ouvert, HTTP clair | Haute |
   | Camera | Page web **par défaut :80**, compte `admin/1234` | Moyenne |

   > **Résultat réel (scan authentifié SRV-CLOUD du 07/10, auth SSH `scan` OK)** : l’EOL Ubuntu est
   > **non détecté** par GCE (aucun OS consolidé → NVT « OS EOL » silencieuse) ; constats réels :
   > **Sudo CVE-2021-3156 « Baron Samedit » 7.8 (High)**, 8 × mitigations CPU manquantes (Moyen — noyau figé),
   > TCP/ICMP timestamps + weak MAC SSH (Bas). **:80 fermé** (nginx de 04 §3 non démarré) → le démarrer si la
   > preuve « HTTP clair » / tshark F-01 est voulue, sinon assumer la sortie de cet aspect de V-04.

   Complément Nmap (argumentaire) :
   ```bash
   nmap --script smb-vuln-ms17-010,smb2-security-mode 172.16.30.50 172.16.31.10   # => MS17-010 VULNERABLE (les 2 Win-7)
   ```
3. **(Optionnel) Exploitation PoC** — Metasploit depuis l'hôte : `eternalblue` contre
   172.16.30.50 → preuve que le constat critique est **exploitable** (usage strictement local).
4. **Remédiation** (simule les actions de production) :
   - Win-7 : patch MS17-010 (KB4012212) + **SMBv1 désactivée** (`RequireSecuritySignature`
     remis à 1 + désactivation du sous-système) + changement `admin/admin` → mot de passe fort ;
     recommandation écrite : **migration** (Win7 EOL = à remplacer, pas à patcher indéfiniment).
   - Windows-10 : mot de passe fort, WinUpdate réactivé, restriction des droits admin.
   - → **Commandes exactes d'exécution : 04 §12** (SMBv1, mots de passe forts, mdp `administrator` VIDE
     de .30.10, durcissement SSH SRV-CLOUD, Camera) + **mettre à jour les credentials GSA avant re-scan**.
   - SRV-CLOUD : mot de passe fort `svc-app`, (recommandation : mise à niveau de l'OS —
     hors lab, OS volontairement conservé ancien pour le scénario).
   - Camera : changement `admin/1234` → mot de passe fort.
5. **Re-scan GCE** + Nmap SMBv1 → MS17-010 et constats corrigés **disparaissent**.

**Preuves** : rapports GCE avant/après · Nmap `smb-vuln-ms17-010` avant/après ·
capture PoC (si réalisé) · tableau de risques CVSS finalisé (livrable semaine 3).

---

### Atelier A — Objectif 4 : scoring & priorisation RBVM (S3, après L1 + L2)

**But** : transformer les constats bruts GCE en **risques métier priorisés** — une 9.8 sur un
poste de test n'a pas le poids d'une 9.8 sur le serveur de données (RBVM).

1. **Grille de criticité métier (université)** — chaque actif noté de 1 (mineur) à 5 (vital),
   **une fois pour toutes au moment de l'inventaire P0** (la colonne « criticité » existait déjà) :

   | Critère | Question | Notation (exemples lab) |
   |---|---|---|
   | Confidentialité | Que perd-on si les données fuient ? | SRV-CLOUD (dossiers étudiants) = 5 · poste = 2 |
   | Disponibilité | Quel service s'arrête si l'actif tombe ? | cœur / FO = 5 · un bâtiment = 3 · un poste = 1 |
   | Exposition | Où se trouve l'actif ? | cloud = 5 · VLAN métier = 3 · VLAN mgmt 10 = 1 |
   | Rôle | Serveur, réseau, poste, IoT ? | serveur = 5 · équipement réseau = 4 · poste = 2 · IoT = 2 |

   **Criticité métier = moyenne des 4 critères (arrondie à l'entier).**
2. **Formule** : `Risque = Sévérité CVSS (GCE) × Criticité métier` —
   **P1 ≥ 25 · P2 = 10-24 · P3 < 10**.
3. **Matrice appliquée au lab** (ordres de grandeur — remplacer par les scores réels des rapports) :

   | Constat | CVSS | Actif (criticité) | Risque | Priorité |
   |---|---|---|---|---|
   | V-04 Ubuntu EOL + HTTP clair | ~9 | SRV-CLOUD (5) | ~45 | **P1 — immédiate** |
   | **F-01 cloud accessible sans VPN** (constat L3, architecture) | ~7 | SRV-CLOUD / données métier (5) | ~35 | **P1 — remédiation IPSec (L4)** |
   | V-03 telnet + SNMPv1 + guest | ~7 | réseau (4) | ~28 | **P1** |
   | V-01 MS17-010 + Win7 EOL | ~8 | Win-7 / Win-7-1 (3) | ~24 | **P1/P2** |
   | V-05 caméra `admin/1234` :80 | ~6 | Camera (2) | ~12 | P2 |
   | V-02 Win10 EOL, WinRM clair | ~5 | Windows-10 (2) | ~10 | P2 |
   | V-06 téléphones (VPCS) | ~3 | IP-Phone (1) | ~3 | P3 |

   → lecture métier : **SRV-CLOUD et le réseau se corrigent AVANT les postes**, même si un constat
   poste paraît « critique » au sens CVSS seul.
4. **Preuves** : grille remplie par actif, matrice de risques exportée (tableur) — la criticité
   est reportée comme attribut des findings dans **DefectDojo (Atelier B)**, qui calcule le risque
   et déclenche les tickets P1.

---

### Lab 3 — Constat F-01 : le cloud sans VPN (le fait de terrain)

> **Pourquoi ce lab reste dans le mémoire** : OpenVAS ne voit pas les failles d'**architecture** —
> L3 prouve que le cycle de GdV (obj 1 : *identification*) dépasse l'outil. F-01 est ensuite
> **scoré et priorisé comme les autres constats** (Atelier A, obj 4), puis remédié (L4) et
> surveillé en non-régression (Atelier C) : il traverse TOUT le cycle de la figure 1 du mémoire.

**But** : **constater et qualifier** le problème d'architecture, avant toute remédiation.

1. **Depuis Windows-10** (console Windows) : `ping 10.9.1.10` → **succès** (le campus atteint le cloud).
2. **Depuis GCE** : `ping 10.9.1.10` → **succès** (le scanner aussi).
3. **Depuis IP-Phone** (VPCS) : `http get http://10.9.1.10/app/` → la page « données métier »
   s'affiche **en clair, depuis un poste du bâtiment**.
4. **Sur SRV-CLOUD** (capture de l'**uplink ens3**, l'unique liaison du cloud) :
   ```bash
   sudo tshark -i ens3 -f "net 10.9.1.0/24" -c 20
   ```
   → on voit les trames en clair : ping, HTTP, **le contenu de la page métier est lisible**
   sur la liaison d'interconnexion (les données partagent le lien du cloud) — aucune confidentialité.
5. **Qualification du risque** (page du mémoire) :
   - Actif exposé : données de l'entreprise (10.9.1.10) + liaison 172.16.90.0/30.
   - Menace : interception (liaison non chiffrée), accès non restreint (tout le campus + le segment
     scanner y accèdent).
   - Conséquence métier : exposition des dossiers étudiants.
   - **Recommandation écrite** : tunnel IPSec site-to-site entre le cœur et le cloud +
     restriction d'accès au cloud au seul segment management (VLAN 10).

**Preuves** : 4 captures (ping Windows-10, ping GCE, http get VPCS, tshark clair) +
la page « recommandation » rédigée. **C'est le pivot du mémoire.**

---

### Lab 4 — Remédiation : tunnel IPSec + ACL d'isolation

> **Pourquoi ce lab reste dans le mémoire** : c'est la **remédiation d'ingénierie** — elle prouve
> que le cycle de GdV se termine par une correction **vérifiée avant/après** (obj 1 : *remédiation
> → vérification*). L'automatisation des correctifs (obj 5) est couverte par l'**Atelier B**
> (Ansible) ; L4 complète : une vulnérabilité d'**architecture** ne se corrige pas par un paquet,
> mais par une conception (chiffrement + politique d'accès). L6 montre que cette remédiation
> **tient** en cas de coupure FO ; l'Atelier C la surveille en non-régression.

**But** : implémenter la recommandation du Lab 3 et **prouver avant/après**.

1. **Côté R-CE (cœur)** (ajouter à la baseline, 03 §8.2) : keyring PSK (`<masqué>`),
   politique IKEv2 (sha256 / group 14), transform-set AES-256, `interface Tunnel90`
   (source G0/2 → destination 172.16.90.2, IP 10.9.255.1/30),
   route `10.9.1.0/24 via 10.9.255.2`, **ACL CLOUD-ACL** (permit 172.16.10.0/24 → 10.9.1.0/24,
   deny any **log**) appliquée `out` sur Tunnel90.
2. **Côté SRV-CLOUD** (Ubuntu 20.04, 03 §8.3) : `apt install libreswan` (dépôts archives 20.04 :
   voir note du 03 §8.3), `ip addr add 10.9.255.2/30 dev ens3`, configuration swanctl
   (ids 172.16.90.2 ↔ 172.16.90.1, selectors 10.9.1.0/24 ↔ 172.16.0.0/16, PSK identique),
   `ip route replace 172.16.0.0/16 via 10.9.255.1`.
3. **Preuves avant/après** (le cœur du lab) :

   | Test | Avant (baseline) | Après (Lab 4) |
   |---|---|---|
   | `ping 10.9.1.10` depuis Windows-10 | **succès** | **timeout — refusé** (+ log IOS `CLOUD-ACL`) |
   | `ping 10.9.1.10` depuis GCE (VLAN 10) | succès (clair) | **succès via le tunnel** |
   | tshark uplink ens3 SRV-CLOUD | HTTP/ping **clairs** (métier 10.9.1.0/24) | **uniquement des paquets ESP** |
   | `show crypto ipsec sa` (R-CE (cœur)) | — | session **UP** (SPI, débit) |
   | `ipsec statusall` (SRV-CLOUD) | — | child **INSTALLED** |

   ```bash
   # R-CE (cœur) (refus loggé) :
   show log | include CLOUD-ACL
   ```
4. **Re-scan GCE du cloud via le tunnel** : mêmes constats V-04, mais l'accès du scanner passe
   désormais par le VPN — l'ACL autorise uniquement le VLAN 10.

**Preuves** : les 5 lignes du tableau ci-dessus (captures) + rapport de re-scan.

---

### Atelier B — Objectif 5 : workflow de remédiation automatisé (S4, en parallèle de L3/L4)

**But** : relier le scanner aux outils d'exploitation — **scan → risque → ticket → alerte →
correctif → clôture** (architecture cible : DefectDojo / Jira / Mattermost / Ansible).
**Tout tourne hors EVE-NG** (détail d'installation : **04 §10**) — la RAM du lab est intacte.

1. **DefectDojo (centralisation)** — Docker **sur l'hôte** (GBVA = 4 GB, déjà chargée par GCE) :
   import des rapports GCE (P0, L1, L2 — parser *Greenbone OpenVAS CSV*). La **criticité métier**
   de l'Atelier A est saisie par actif → le **Risque = CVSS × Criticité** est calculé par finding.
2. **Jira (tickets)** — site **Jira Cloud** gratuit + token API → DefectDojo *Integrations → Jira* :
   règle « **finding P1 (Risque ≥ 25) → ticket auto** » dans le projet `VMS-LAB` (priorité Highest,
   assigné « équipe système »). Le lien OpenVAS→Jira passe par DefectDojo, pas par un script maison.
3. **Mattermost (alertes)** — conteneur Docker sur l'hôte, canal `#sec-alerts` : message immédiat
   « P1 détectée sur <actif> » avec le lien du ticket (webhook).
4. **Ansible (correctif)** — **control node = GBVA** (seul nœud routé vers tout le lab) ;
   playbook de remédiation (**04 §10.4**) contre la **Camera** (cible de démo) :
   scénario « une caméra a été réinstallée avec `admin/1234` » → `ansible-playbook remediation-camera.yml`
   → mot de passe fort appliqué, sortie `changed=1` capturée. **Jamais contre SRV-CLOUD**
   (l'OS doit rester non patché — scénario V-04).
5. **Boucle de clôture** : re-scan ciblé de la Camera (Fast Scan, ou `gvm-cli` Atelier C) →
   import DefectDojo → finding **mitigé** → **ticket clôturé automatiquement**. Capturer la chaîne
   complète : ticket créé → playbook → re-scan → ticket fermé.

**Preuves** : capture dashboard DefectDojo (imports + risque), ticket Jira auto, message
`#sec-alerts`, sortie `ansible-playbook`, ticket fermé après re-scan.

---

### Lab 5 — Exploitation continue du VMS (KPI)

**But** : montrer que la GdV est un **processus continu**, pas un scan unique.

1. Dans GCE : créer les **scans planifiés** — *Fast Scan* quotidien + *Full Scan* hebdomadaire
   sur les deux cibles (campus / cloud), avec les credentials.
2. Générer les **exports PDF** (rapports datés).
3. Construire le **tableau de bord KPI** (feuille de calcul ou vue GCE) :
   - **Couverture** : actifs scannés / actifs inventoriés (P0) ;
   - **Critiques par VLAN** (VLAN 30 vs cloud) ;
   - **MTTM** : délai moyen détection → correction (mesuré sur le registre de risques,
     alimenté depuis L1/L2/L4) ;
   - **Tendance** : nombre de constats critiques par scan (doit décroître L1→L2→L4).
4. **Lien objectif 6** — consolider dans le **dashboard DefectDojo** (Atelier B) : le **MTTR** y est
   calculé automatiquement (`date de clôture - date de première détection`), avec les tendances par
   criticité. Compléter au tableur : **top 5 des serveurs les plus vulnérables**, **âge moyen des
   constats non corrigés** (critique > 30 j = alerte) et **taux de remédiation automatisée**
   (Ansible vs manuel).

**Preuves** : capture des planifications GCE, 2 rapports PDF exportés, tableau KPI rempli,
**capture dashboard DefectDojo (MTTR + tendances)**.

---

### Lab 6 — Continuité de service : basculement double liaison FO

**But** : démontrer que **la plateforme VMS reste disponible** même en cas de coupure
l'argument « double liaison FO » de l'architecture réelle devient celui du projet.

1. **Avant** (R-CE (cœur)) :
   ```
   show ip route
   ```
   → défaut via **172.16.255.1** (CE-1) + flottante via 172.16.254.1 **distance 252**.
   Depuis Windows-10 : `ping 10.0.0.254` (internet simulé) = OK.
2. **Coupure** (**R-CE, cœur** — le port côté DC) :
   ```
   configure terminal
   interface G0/0
   shutdown
   ```
   (= rupture de la FO principale MADIYANA).
   ⚠ **Important (EVE-NG)** : le `shutdown` doit être fait sur le port **du cœur**, pas sur
   R-CE-1 — un port virtuel ne propage pas la rupture de liaison : si vous coupez seulement
   R-CE-1, la route directe /30 du cœur reste active et **le basculement ne se déclenche pas**.
3. **Après** (R-CE (cœur)) : `show ip route` → défaut désormais via **172.16.254.1** (CE-2,
   flottante montée). Depuis Windows-10 : `ping 10.0.0.254` = **redevient OK** (quelques pings perdus
   au basculement — normal et à documenter).
4. **La plateforme tient la coupure** (le point important) :
   - Depuis GCE : `ping 10.9.1.10` = **OK** (le tunnel IPSec est sur le lien 172.16.90.0/30,
     pas sur la FO — il ne bouge pas).
   - Lancer un **Fast Scan pendant la coupure** → il se termine normalement (segment VLAN 10
     indépendant des liaisons opérateur).
5. **Restitution** : `no shutdown` sur G0/0 **du cœur** → observer le retour à 172.16.255.1.

**Preuves** : `show ip route` avant/après · pings Windows-10 pendant la coupure ·
Fast Scan lancé et terminé pendant la coupure · (bonus) capture des secondes de basculement.

---

### Atelier C — Objectif 7 : évaluation de l'efficacité & non-régression (S6, après L6)

**But** : prouver que les correctifs **tiennent dans le temps** — scans réguliers, détection des
réapparitions (*Re-opened*), indicateurs d'efficacité. Scripts : **04 §11**.

1. **Routine (cron sur GBVA)** : tâche **hebdomadaire** `gvm-cli` → Full Scan parc + export/import
   DefectDojo automatique (le « pouls » du parc, sans intervention).
2. **Scan post-remédiation** : dès qu'un ticket passe « Corrigé » (Atelier B), `gvm-cli` relance le
   scan de la **seule cible concernée** → import → finding **mitigé** → ticket clôturé.
3. **Test de réapparition (démonstration maîtrisée)** : replanter volontairement une faille corrigée
   — ex. `transport input telnet` sur SW-MET (via son SVI 172.16.30.2) ou recréer le compte `guest`
   pendant 10 min → re-scan → DefectDojo repasse le finding en **Re-opened** + alerte → le KPI
   **taux de régression** se dégrade → re-correction → re-scan → fermé. *(C'est la démo
   « non-régression » de la soutenance : les failles fermées ne se rouvrent pas.)*
   **+ Non-régression F-01 (remédiation L4)** : le contrôle hebdomadaire vérifie que la correction
   tient — tunnel **UP** (`ipsec statusall` sur SRV-CLOUD, `show crypto ipsec sa` sur le cœur),
   GCE → 10.9.1.10 toujours **autorisé** (via tunnel), Windows-10 toujours **refusé** (ACL).
   Toute réapparition de F-01 (cloud joignable en clair) = alerte P1.
4. **Contrôles de configuration (hardening)** : après chaque correctif, vérifier que rien n'a cassé —
   `sshd -T` (algos/auth), `systemd-analyze security nginx.service`, Nmap de contrôle des ports
   attendus. **Lynis**/InSpec sur la cible si un dépôt est accessible — sinon ces contrôles
   natifs suffisent au lab.
5. **Lecture des KPIs d'efficacité** (dashboard Atelier B + tableur L5) :

   | KPI | Attendu après les 6 semaines |
   |---|---|
   | Taux de remédiation | fermés > ouverts (tendance croissante) |
   | MTTR | diminue entre L1/L2 (manuel) et Atelier B (automatisé) |
   | Taux de régression | ≈ 0 % (hors test volontaire de l'étape 3) |
   | Failles critiques | décroissantes (baseline P0 → L4) |

6. **Preuves** : crontab + log d'exécution `gvm-cli`, capture finding **Re-opened**, tableau KPI
   final — alimente directement le chapitre « Évaluation de la solution » du mémoire.

---

## 7. Rappels et pièges pratiques

- **Un lab = un dossier de preuves** : chaque commande importante est capturée (console, capture
  d'écran horodatée) et déposée dans le sous-dossier du lab. Rien n'est « dit » dans le mémoire
  sans sa capture.
- **GCE = 4 GB** : un seul Full Scan à la fois ; Fast Scan pour les vérifications rapides.
- **Windows 7 sous EVE-NG** : image legacy BIOS (non UEFI) ; si freeze → 2 GB temporaires
  (le swap EVE-NG absorbe).
- **Ubuntu 20.04 (SRV-CLOUD)** : dépôts en archives (EOL 2025) — si `apt install libreswan` échoue,
  prendre le `.deb` ou faire le Lab 4 après le scan initial (les constats V-04 sont déjà capturés).
- **ACL direction** : `CLOUD-ACL` s'applique **`out` sur Tunnel90** (filtre ce qui va **vers** le
  cloud) — pas `in` (ce serait bloquer les réponses du cloud).
- **Ordre sacré** : P0 → L1 → L2 → L3 → L4 → L5 → L6 (+ A après L2, B en S4, C en dernier).
  Ne jamais faire L4 avant d'avoir écrit et capturé L3.
- **Scan non authentifié ≠ « tout le réseau »** — la différence est le **credential**, pas la portée :
  - **non authentifié** = le scanner voit comme un **attaquant extérieur**, sans aucun compte (ports,
    versions, failles visibles de dehors). Ça peut être **large** (P0 : tout l'inventaire) comme
    **ciblé** (Camera et Windows-10-1, volontairement sans compte : la faiblesse du credential
    — ou son absence — EST le constat V-05/V-02) ;
  - **authentifié** = le scanner **se connecte** avec les credentials (04 §6.5) et audite **de
    l'intérieur** : patchs manquants, OS EOL, WinRM non chiffré, SNMP v1 — des failles **invisibles
    de l'extérieur**. Là aussi ça peut être **large** (L2 balaye les deux /24 avec SMB/SSH, GVM
    utilise automatiquement le bon compte sur chaque hôte) comme **ciblé** (L1 : les 6 IOS via SNMP) ;
  - les deux se **complètent** : P0 non authentifié (baseline large) → L1/L2 authentifiés (profondeur).
    Phrase de soutenance : *« le non authentifié montre ce que voit l'attaquant ; l'authentifié montre
    ce que vous ne devez pas ignorer. »*
- **Ateliers B/C = hors EVE-NG** : DefectDojo + Mattermost en Docker **sur l'hôte** (2-3 GB, up
  uniquement pendant les démos), Jira = Cloud, Ansible + gvm-tools = **GBVA**. Ne rien installer
  de plus dans GBVA (4 GB déjà chargés par GCE).

## 8. L'analogie (pour expliquer en soutenance)

Le VMS (GCE/OpenVAS) est le **médecin** ; les labs sont les temps des soins :

| Lab | Analogie |
|---|---|
| P0 | Radiographie + dossier du patient (inventaire) |
| L1 / L2 | Diagnostic par service (réseau, postes, cloud, IoT) |
| L3 | Découverte de la maladie de fond : **pas de VPN** |
| L4 | Opération + contrôle après (preuves avant/après) |
| L5 | Suivi de routine : contrôles planifiés + constantes (KPI) |
| L6 | « Le patient marche encore sur une jambe » : disponibilité en cas de coupure |
| A | Triage des urgences : gravité de la plaie × état du patient (pas que le score) |
| B | Ordonnance automatisée : le pharmacien (Ansible) délivre, le contrôle (re-scan) valide |
| C | Visite de contrôle : la maladie ne revient pas |

Et la phrase-clé à retenir : **chaque lab est démontrable en 5 minutes** — une commande, un
rapport, une capture. Le mémoire ne contient jamais d'affirmation sans preuve horodatée.
