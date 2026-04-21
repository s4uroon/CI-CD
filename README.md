# Pipeline de Déploiement Sécurisé — Guide d'Architecture et d'Implémentation

> **Stack** : GitLab · Ansible · Ansible Semaphore
> **Modèle** : Pull en cascade (GitLab → Git+Ansible HP → Git+Ansible Prod → App Prod)
> **Déclenchement** : Manuel via 2 instances Semaphore indépendantes

---

## Table des matières

1. [Vue d'ensemble](#1-vue-densemble)
2. [Infrastructure — 2 zones indépendantes](#2-infrastructure--2-zones-indépendantes)
3. [Structure du dépôt](#3-structure-du-dépôt)
4. [Prérequis](#4-prérequis)
5. [Gestion des clés SSH](#5-gestion-des-clés-ssh)
6. [Configuration par dépôt](#6-configuration-par-dépôt)
7. [Playbooks Ansible](#7-playbooks-ansible)
8. [Configuration d'Ansible Semaphore](#8-configuration-dansible-semaphore)
9. [Flag de validation HP](#9-flag-de-validation-hp)
10. [Stratégie de Rollback](#10-stratégie-de-rollback)
11. [Procédure opérationnelle standard](#11-procédure-opérationnelle-standard)

---

## 1. Vue d'ensemble

Le flux de déploiement suit un modèle **Pull en cascade strict** à travers
deux zones indépendantes, chacune avec son propre Semaphore et son serveur
Git+Ansible. Aucun serveur ne "pousse" : chaque niveau tire depuis le niveau
supérieur, uniquement sur action humaine.

```
Développeur  ──push──►  GitLab Central
                               │
                   ╔═══════════════════════╗
                   ║    ZONE HORS-PROD     ║
                   ║                       ║
                   ║  [Bouton 1 SEM-HP]    ║
                   ║    │ git fetch         ║
                   ║    ▼                   ║
                   ║  Git+Ansible HP (bare) ║
                   ║  /opt/git/myapp.git   ║
                   ║    │                   ║
                   ║  [Bouton 2 SEM-HP]    ║
                   ║    │ git fetch         ║
                   ║    ▼                   ║
                   ║  App Server(s) HP     ║  ← Recette ✓
                   ║  /opt/apps/myapp      ║
                   ║    │                   ║
                   ║  [Bouton 3 SEM-HP]    ║
                   ║    └─ tag git         ║  ← VALIDATION HUMAINE
                   ║       validated/hp/.. ║
                   ╚═══════════════════════╝
                               │
                   ╔═══════════════════════╗
                   ║    ZONE PRODUCTION    ║
                   ║                       ║
                   ║  [Bouton 1 SEM-PROD]  ║
                   ║    │ check flag + fetch║
                   ║    ▼                   ║
                   ║  Git+Ansible Prod      ║
                   ║  /opt/git/myapp.git   ║
                   ║    │                   ║
                   ║  [Bouton 2 SEM-PROD]  ║
                   ║    │ BDD backup + fetch║
                   ║    ▼                   ║
                   ║  App Server(s) Prod   ║  ← Production ✓
                   ║  /opt/apps/myapp      ║
                   ╚═══════════════════════╝
```

### Les 7 boutons Semaphore

**Semaphore HP** (4 boutons) :

| # | Bouton          | Playbook                   | Action                                              |
|---|-----------------|----------------------------|-----------------------------------------------------|
| 1 | Synchro Git HP  | `hp/01_sync_git_hp.yml`    | Git+Ansible HP fetche depuis GitLab (bare)          |
| 2 | Déployer HP     | `hp/02_deploy_hp.yml`      | App HP fetche depuis Git+Ansible HP + déploiement   |
| 3 | Valider HP      | `hp/03_validate_hp.yml`    | Pose le tag `validated/hp/<repo>` sur bare HP       |
| 4 | Rollback HP     | `hp/04_rollback_hp.yml`    | Retourne l'App HP à un tag ou commit précédent      |

**Semaphore Prod** (3 boutons) :

| # | Bouton           | Playbook                    | Action                                              |
|---|------------------|-----------------------------|-----------------------------------------------------|
| 1 | Synchro Git Prod | `prod/01_sync_git_prod.yml` | Vérifie flag HP, puis fetche depuis Git+Ansible HP  |
| 2 | Déployer Prod    | `prod/02_deploy_prod.yml`   | Sauvegarde BDD + déploiement sur App Prod           |
| 3 | Rollback Prod    | `prod/03_rollback_prod.yml` | Sauvegarde BDD + retour à une version précédente    |

> Le **Bouton 1 Prod est bloqué** si le Bouton 3 HP n'a pas été exécuté.
> Voir [docs/validation-flag.md](docs/validation-flag.md).

---

## 2. Infrastructure — 2 zones indépendantes

| Serveur              | Hostname                      | Zone          | Rôle                                              |
|----------------------|-------------------------------|---------------|---------------------------------------------------|
| GitLab Central       | gitlab.company.com            | Externe/DMZ   | Stockage source de vérité                         |
| Semaphore HP         | semaphore-hp.company.com      | Hors-Prod     | Semaphore UI + Ansible control node HP            |
| Git+Ansible HP       | git-ansible-hp.company.com    | Hors-Prod     | Dépôt bare miroir + Ansible installé              |
| App Server(s) HP     | app[n]-hp.company.com         | Hors-Prod     | N serveurs applicatifs HP (PHP/Python/Node.js/Java/Go) |
| Semaphore Prod       | semaphore-prod.company.com    | Production    | Semaphore UI + Ansible control node Prod          |
| Git+Ansible Prod     | git-ansible-prod.company.com  | Production    | Dépôt bare miroir + Ansible installé              |
| App Server(s) Prod   | app[n]-prod.company.com       | Production    | N serveurs applicatifs Prod (PHP/Python/Node.js/Java/Go) |

**Point clé** : Semaphore HP est dans la zone HP (pas dans la zone Prod).
Les deux Semaphore sont des instances indépendantes, chacune avec son propre
inventory et ses propres clés SSH.

---

## 3. Structure du dépôt

```
CI-CD/
├── README.md
├── docs/
│   ├── procedure-creation-application.md   ← Procédure standardisée de mise en service d'une nouvelle application
│   ├── validation-flag.md                  ← Mécanisme flag git HP → Prod (transverse aux deux zones)
│   │
│   ├── architecture_hp.md                  ← Architecture zone Hors-Prod (serveurs, clés A-D, boutons 1-4)
│   ├── architecture_prod.md                ← Architecture zone Production (serveurs, clés E-H, boutons 1-3)
│   │
│   ├── ldap-https-configuration_hp.md      ← Configuration LDAP + Nginx HTTPS pour Semaphore HP
│   ├── ldap-https-configuration_prod.md    ← Configuration LDAP + Nginx HTTPS pour Semaphore PROD
│   │
│   ├── notifications_hp.md                 ← Notifications (email/Slack/Teams) pour la zone HP
│   ├── notifications_prod.md               ← Notifications (email/Slack/Teams) pour la zone PROD
│   │
│   ├── offline-installation-rhel9_hp.md    ← Installation hors-ligne RHEL9 — serveurs zone HP
│   ├── offline-installation-rhel9_prod.md  ← Installation hors-ligne RHEL9 — serveurs zone PROD
│   │
│   ├── rollback-strategy_hp.md             ← Stratégie de rollback HP (Bouton 4 HP, sans BDD)
│   ├── rollback-strategy_prod.md           ← Stratégie de rollback PROD (Bouton 3 PROD, avec sauvegarde BDD)
│   │
│   ├── ssh-key-management_hp.md            ← Clés SSH A, B, C, D — génération, distribution, tests HP
│   └── ssh-key-management_prod.md          ← Clés SSH E, F, G, H — génération, distribution, tests PROD
├── ansible/
│   ├── ansible.cfg
│   ├── inventory/
│   │   ├── hp/
│   │   │   ├── hosts.yml            ← git_ansible_hp + app_hp (pour Semaphore HP)
│   │   │   └── group_vars/
│   │   │       ├── all.yml          ← Variables communes zone HP
│   │   │       ├── git_ansible_hp.yml ← Clé B, gitlab_ssh_host, git_bare_base
│   │   │       └── app_hp.yml       ← Clé D, git_server_host, app_env=recette
│   │   └── prod/
│   │       ├── hosts.yml            ← git_ansible_prod + app_prod (pour Semaphore Prod)
│   │       └── group_vars/
│   │           ├── all.yml          ← Variables communes zone Prod
│   │           ├── git_ansible_prod.yml ← Clé F, git_hp_ssh_host
│   │           └── app_prod.yml     ← Clé H, git_server_host, app_env=production
│   ├── repos/
│   │   └── myapp.yml                ← Exemple de config par dépôt
│   ├── playbooks/
│   │   ├── hp/
│   │   │   ├── 01_sync_git_hp.yml   ← Bouton 1 HP : fetch GitLab → bare HP
│   │   │   ├── 02_deploy_hp.yml     ← Bouton 2 HP : fetch + deploy App HP
│   │   │   ├── 03_validate_hp.yml   ← Bouton 3 HP : tag validated/hp/<repo>
│   │   │   └── 04_rollback_hp.yml   ← Bouton 4 HP : rollback App HP vers un tag
│   │   └── prod/
│   │       ├── 01_sync_git_prod.yml ← Bouton 1 Prod : check flag + fetch HP → bare Prod
│   │       ├── 02_deploy_prod.yml   ← Bouton 2 Prod : BDD backup + deploy App Prod
│   │       └── 03_rollback_prod.yml ← Bouton 3 Prod : BDD backup + rollback App Prod
│   └── roles/
│       ├── git_sync/
│       │   ├── tasks/
│       │   │   ├── main.yml         ← Dispatcher (bare vs working)
│       │   │   ├── sync_bare.yml    ← Miroir bare avec block/rescue détaillé
│       │   │   └── sync_working.yml ← Working tree avec block/rescue détaillé
│       │   └── defaults/main.yml
│       └── deploy/
│           └── tasks/
│               ├── main.yml
│               ├── install_deps.yml              ← PHP (Composer)
│               ├── install_deps_python.yml       ← Python (pip/venv)
│               ├── install_deps_nodejs.yml       ← Node.js (npm)
│               ├── install_deps_java.yml         ← Java (Maven/Gradle)
│               ├── install_deps_go.yml           ← Go (go build)
│               ├── run_migrations.yml            ← PHP / Python
│               ├── run_migrations_nodejs.yml     ← Node.js
│               ├── run_migrations_java.yml       ← Java
│               └── run_migrations_rollback.yml
└── semaphore/
    ├── hp_project_config.md         ← Guide configuration Semaphore HP (3 boutons)
    └── prod_project_config.md       ← Guide configuration Semaphore Prod (2 boutons)
```

---

## 4. Prérequis

### Sur Semaphore HP
- Ansible >= 2.14, Ansible Semaphore >= 2.9
- Clés SSH A et C dans `~/.ssh/`

### Sur Semaphore Prod
- Ansible >= 2.14, Ansible Semaphore >= 2.9
- Clés SSH E et G dans `~/.ssh/`

### Sur Git+Ansible HP et Prod
- Git >= 2.x, Ansible installé (pour être utilisé comme target Ansible)
- Répertoire `/opt/git/` pour les dépôts bare
- Utilisateur `deploy` avec authorized_keys configurés

### Sur App HP et App Prod
- Git >= 2.x
- Selon la stack applicative (un ou plusieurs) :
  - PHP >= 8.x (Composer)
  - Python >= 3.8 (pip/venv)
  - Node.js >= 18.x (npm)
  - Java >= 17 (Maven ou Gradle)
  - Go >= 1.21 (go build)
- Service systemd de l'application
- PostgreSQL client (`pg_dump`) pour les sauvegardes avant déploiement Prod

---

## 5. Gestion des clés SSH

Voir les guides détaillés : [docs/ssh-key-management_hp.md](docs/ssh-key-management_hp.md) · [docs/ssh-key-management_prod.md](docs/ssh-key-management_prod.md)

### Résumé des 8 paires de clés

| Clé | Source             | Destination        | Type                   |
|-----|--------------------|--------------------|------------------------|
| A   | Semaphore HP       | Git+Ansible HP     | Ansible SSH (complet)  |
| B   | Git+Ansible HP     | GitLab             | git fetch (read-only)  |
| C   | Semaphore HP       | App Server(s) HP   | Ansible SSH (complet)  |
| D   | App Server(s) HP   | Git+Ansible HP     | git fetch (read-only)  |
| E   | Semaphore Prod     | Git+Ansible Prod   | Ansible SSH (complet)  |
| F   | Git+Ansible Prod   | Git+Ansible HP     | SSH + git fetch        |
| G   | Semaphore Prod     | App Server(s) Prod | Ansible SSH (complet)  |
| H   | App Server(s) Prod | Git+Ansible Prod   | git fetch (read-only)  |

Les clés D et H sont restreintes dans `authorized_keys` avec
`command="git-upload-pack ..."` — elles ne permettent pas d'ouvrir un shell.

---

## 6. Configuration par dépôt

Chaque application dispose d'un fichier `ansible/repos/<repo_name>.yml`
qui définit les paramètres spécifiques au dépôt, notamment la **liste des
serveurs cibles** (configurable indépendamment pour HP et Prod).

```yaml
# ansible/repos/myapp.yml
repo_name: myapp
git_remote_url: "git@gitlab.company.com:mygroup/myapp.git"
git_branch: main
app_type: php          # "php", "python", "nodejs", "java", "go" ou "static"

app_root: "/opt/apps/myapp"
app_service_name: myapp

# Serveurs cibles HP (autant que nécessaire)
app_servers_hp:
  - app1-hp.company.com
  - app2-hp.company.com

# Serveurs cibles Prod (autant que nécessaire)
app_servers_prod:
  - app1-prod.company.com
  - app2-prod.company.com

run_migrations: true
db_host_prod: prod-db.company.com
db_name_prod: myapp_prod
```

Les playbooks reçoivent `repo_name` comme **Survey Variable** dans Semaphore
et chargent dynamiquement ce fichier via `include_vars`.

---

## 7. Playbooks Ansible

Le rôle `git_sync` gère deux modes via la variable `git_repo_type` :

```
git_repo_type: bare     → sync_bare.yml    (Git+Ansible HP, Git+Ansible Prod)
               working  → sync_working.yml (App HP, App Prod)
```

Les playbooks 02 et 02 (deploy) utilisent `add_host` pour construire
dynamiquement le groupe de serveurs cibles à partir de la liste dans le
fichier de configuration du dépôt.

Le rôle `deploy` gère les étapes applicatives (dépendances, migrations, reload).

Toutes les étapes utilisent des blocs `block/rescue` avec messages d'erreur
explicites et résumé visuel par étape.

---

## 8. Configuration d'Ansible Semaphore

Voir les guides détaillés :
- [semaphore/hp_project_config.md](semaphore/hp_project_config.md) — Semaphore HP (4 boutons)
- [semaphore/prod_project_config.md](semaphore/prod_project_config.md) — Semaphore Prod (3 boutons)

---

## 9. Flag de validation HP

Voir le guide détaillé : [docs/validation-flag.md](docs/validation-flag.md) (document transverse HP/PROD)

Le **Bouton 3 HP** crée un tag git `validated/hp/<repo_name>` sur le dépôt
bare HP, signalant que le déploiement HP a été validé par l'équipe.

Le **Bouton 1 Prod** vérifie ce flag via SSH avant d'autoriser tout fetch.
Si le flag est absent, le playbook échoue avec le message :

```
BLOQUE : Actions requises sur Semaphore HP :
  1. Bouton 1 — Synchro Git HP
  2. Bouton 2 — Deployer HP
  3. Bouton 3 — Valider HP
```

---

## 10. Stratégie de Rollback

Voir les guides détaillés : [docs/rollback-strategy_hp.md](docs/rollback-strategy_hp.md) · [docs/rollback-strategy_prod.md](docs/rollback-strategy_prod.md)

**Rollback HP (Bouton 4 HP)** :
- Retourne l'application HP à un tag ou commit précédent (sans sauvegarde BDD)
- Variables Semaphore : `repo_name` + `rollback_tag` (tag sémantique, tag backup, ou hash court)
- Après un rollback HP, re-exécuter le **Bouton 3 HP** pour re-valider avant tout déploiement Prod

**Rollback Prod (Bouton 3 Prod)** :
- Tag de sauvegarde `backup/prod-pre-deploy-<date>-<heure>` créé automatiquement
  avant chaque déploiement Prod (dans le Bouton 2 Prod)
- Dump PostgreSQL créé avant chaque migration et avant chaque rollback Prod
- Pour rollback : lancer le **Bouton 3 Prod** avec `repo_name` et `rollback_tag`
  (ex: `backup/prod-pre-deploy-2026-04-09-14-30-00` ou `v1.x.y`)

---

## 11. Procédure opérationnelle standard

```
Dev push sur GitLab (branche main ou tag release)
         │
         ▼
═══════ ZONE HP (Semaphore HP) ════════════════════

[Bouton 1 HP] Synchro Git HP
  → Git+Ansible HP fetche depuis GitLab
  → Affiche : commit hash, branches disponibles

[Bouton 2 HP] Déployer HP
  → Survey Variable : repo_name = myapp
  → App HP fetche + checkout + deps + migrations + reload
  → Affiche état de chaque étape (OK / ERREUR)

═══ Validation QA (tests fonctionnels sur HP) ═════

[Bouton 3 HP] Valider HP
  → Survey Variable : repo_name = myapp
  → Pose le tag git validated/hp/myapp
  → DÉBLOQUE le déploiement en Production

[Bouton 4 HP] Rollback HP  ← si problème après déploiement HP (optionnel)
  → Survey Variables : repo_name = myapp, rollback_tag = v1.2.0
  → App HP revenue à la version cible
  → Si déploiement Prod souhaité : re-exécuter Bouton 3 HP

═══════ ZONE PROD (Semaphore Prod) ════════════════

[Bouton 1 Prod] Synchro Git Prod
  → Survey Variable : repo_name = myapp
  → Vérifie le flag validated/hp/myapp (bloquant si absent)
  → Git+Ansible Prod fetche depuis Git+Ansible HP
  → Affiche : commit validé, cohérence HP/Prod

[Bouton 2 Prod] Déployer Prod
  → Survey Variable : repo_name = myapp
  → Sauvegarde BDD (pg_dump), git tag backup/prod-pre-deploy-*
  → App Prod fetche + checkout + deps + migrations + reload + health check
  → Rolling deploy (serial: 1) — un serveur à la fois

[Bouton 3 Prod] Rollback Prod  ← si problème après déploiement Prod (optionnel)
  → Survey Variables : repo_name = myapp, rollback_tag = backup/prod-pre-deploy-*
  → Sauvegarde BDD pré-rollback, checkout + deps + rollback migrations + reload
```
