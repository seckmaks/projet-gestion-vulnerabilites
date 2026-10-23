# VMS — Système de Gestion des Vulnérabilités pour l'Université Cheikh Ahmadoul Khadim (UCAK)

**Projet de Fin de Cycle** — Makane SECK
UFR des Métiers et Technologies (MET) — Département Informatique et Télécommunications
Option : Administration et Sécurité des Réseaux — Année académique 2025-2026
Encadreur : Dr Abdoul Aziz CISS (EPT)

## 🎯 Le projet

Conception et mise en place d'un laboratoire complet de gestion des vulnérabilités (RBVM)
pour l'université, depuis la découverte réseau jusqu'à la preuve de non-régression :

1. **Objectif 1-2** : lab EVE-NG (architecture MADIYANA/NDAM-MBACKÉ, VLAN 30/31), scanner Greenbone (GSA), premier scan
2. **Objectif 3-4** : scans authentifiés, centralisation DefectDojo, scoring RBVM (Risque = CVSS × Criticité métier)
3. **Objectif 5** : workflow complet DefectDojo → Jira → Slack → Ansible → clôture automatique
4. **Objectif 6** : pilotage par 9 KPI (couverture, MTTR, tendance, taux d'automatisation, régression…)
5. **Objectif 7** : planifications GSA, tests de non-régression, piège telnet, bascule de double liaison

## 📁 Structure du dépôt

```
├── lab/                  Lab EVE-NG : architecture, cahier des charges, configs IOS, guide
├── ansible/              Inventaire + playbooks (remédiation caméra IP, rotation, idempotence)
├── scripts/              Générateurs Python des livrables (reportlab / python-docx)
├── docs/
│   ├── livrables/        Les 6 livrables hebdomadaires (PDF + DOCX)
│   ├── preuves/          Rapports GSA, tableur KPI, captures d'écran du lab
│   └── memoire/          Le mémoire de fin de cycle (PDF)
```

## 🔧 Régénérer les livrables

```bash
pip install reportlab python-docx pillow pypdf
python3 scripts/build_livrable2.py            # PDF (idem pour 3-6)
python3 scripts/build_livrable2_docx.py       # DOCX jumeau
```

## 🛠️ Outils du lab

EVE-NG · Greenbone Vulnerability Management (OpenVAS/GSA 25.04) · DefectDojo 3.4.0 (Community)
· Jira Cloud · Slack (API bot) · Ansible · nmap · Cisco IOS · Windows 7/10 · Ubuntu (SRV-CLOUD)

## ⚠️ Éthique et sécurité

- Toutes les vulnérabilités ont été **plantées volontairement** dans un laboratoire isolé (EVE-NG) à des fins pédagogiques.
- Les **mots de passe sont masqués** dans ce dépôt (`<masqué>`, `Cam-********`, variables `{{ cam_password }}` /
  `{{ hash_cam_password }}` à fournir via `ansible-vault`). Pensez à **faire une rotation** de tout secret de lab après usage.
- Aucun jeton réel (Slack `xoxb`, Atlassian `ATATT`, API DefectDojo) n'est présent dans ce dépôt.

## 📜 Mémoire

Le mémoire complet (36 pages) est disponible dans [`docs/memoire/`](docs/memoire/).
