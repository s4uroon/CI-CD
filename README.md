# Pipeline de Déploiement Sécurisé — Guide d'Architecture et d'Implémentation

> **Stack** : GitLab · Ansible · Ansible Semaphore
> **Modèle** : Pull en cascade (GitLab → Git HP → Git Prod → App Prod)
> **Déclenchement** : Manuel via Semaphore (Single Pane of Glass)

---

## Table des matières

1. [Vue d'ensemble](#1-vue-densemble)
2. [Infrastructure — 6 serveurs](#2-infrastructure--6-serveurs)
3. [Structure du dépôt](#3-structure-du-dépôt)
4. [Prérequis](#4-prérequis)
5. [Gestion des clés SSH](#5-gestion-des-clés-ssh)
6. [Playbooks Ansible](#6-playbooks-ansible)
7. [Configuration d'Ansible Semaphore](#7-configuration-dansible-semaphore)
8. [Stratégie de Rollback](#8-stratégie-de-rollback)
9. [Procédure opérationnelle standard](#9-procédure-opérationnelle-standard)

---

## 1. Vue d'ensemble

Le flux de déploiement suit un modèle **Pull en cascade strict** à travers
des serveurs Git dédiés (dépôts bare) qui servent de relais entre les
environnements. Aucun serveur ne "pousse" : chaque niveau tire depuis le
niveau supérieur, uniquement sur action humaine.

```
Développeur  ──push──►  GitLab Central
                               │
                     [Bouton 1 Semaphore]
                               │ git fetch
                               ▼
                       Git Server HP (bare)
                       /opt/git/myapp.git
                               │
                     [Bouton 2 Semaphore]
                               │ git fetch + checkout
                               ▼
                       App Server HP (working tree)
                       /opt/apps/myapp  → App Hors-Prod ✓
                               │
                     [Bouton 3 Semaphore]
                               │ git fetch (connexion Git Prod → Git HP)
                               ▼
                       Git Server Prod (bare)
                       /opt/git/myapp.git
                               │
                     [Bouton 4 Semaphore]
                               │ git fetch + checkout
                               ▼
                       App Server Prod (working tree)
                       /opt/apps/myapp  → App Production ✓
```

Les 5 boutons dans Semaphore correspondent aux 5 Task Templates :

| # | Bouton Semaphore    | Playbook                 | Cible Ansible | Action                                      |
|---|---------------------|--------------------------|---------------|---------------------------------------------|
| 1 | Synchro Git HP      | `01_sync_git_hp.yml`     | `git_hp`      | Git HP tire depuis GitLab (bare fetch)      |
| 2 | Déployer HP         | `02_deploy_hp.yml`       | `app_hp`      | App HP tire depuis Git HP + deploy          |
| 3 | Synchro Git Prod    | `03_sync_git_prod.yml`   | `git_prod`    | Git Prod tire depuis Git HP (bare fetch)    |
| 4 | Déployer Prod       | `04_deploy_prod.yml`     | `app_prod`    | App Prod tire depuis Git Prod + deploy      |
| 5 | Rollback Prod       | `05_rollback_prod.yml`   | `app_prod`    | Checkout tag antérieur + redéploiement      |

Pour les schémas détaillés, voir [docs/architecture.md](docs/architecture.md).

---

## 2. Infrastructure — 6 serveurs

| Serveur         | Hostname                   | Zone       | Rôle                                   |
|-----------------|----------------------------|------------|----------------------------------------|
| GitLab Central  | gitlab.company.com         | Externe    | Stockage source (pas de CI/CD actif)   |
| Git Server HP   | git-hp.company.com         | Hors-Prod  | Dépôt bare, miroir de GitLab           |
| App Server HP   | app-hp.company.com         | Hors-Prod  | Application + working tree             |
| Git Server Prod | git-prod.company.com       | Production | Dépôt bare, miroir de Git HP           |
| App Server Prod | app-prod.company.com       | Production | Application + working tree             |
| Orchestrateur   | semaphore.prod.company.com | Production | Ansible Semaphore, point d'entrée unique|

**Distinction clé** : les serveurs Git (HP et Prod) ne font tourner aucune
application. Ce sont des dépôts bare purs (`/opt/git/myapp.git`) qui servent
exclusivement de relais Git entre les niveaux du pipeline.

---

## 3. Structure du dépôt

```
CI-CD/
├── README.md
├── docs/
│   ├── architecture.md              ← Schémas Mermaid (6 serveurs, 8 clés SSH)
│   ├── ssh-key-management.md        ← Guide complet des 8 paires de clés
│   └── rollback-strategy.md         ← Stratégie de rollback par tag Git
├── ansible/
│   ├── ansible.cfg
│   ├── inventory/
│   │   ├── hosts.yml                ← 4 groupes : git_hp, app_hp, git_prod, app_prod
│   │   └── group_vars/
│   │       ├── all.yml              ← Variables communes
│   │       ├── git_hp.yml           ← Git HP : bare, remote = GitLab, Clé E
│   │       ├── app_hp.yml           ← App HP : working tree, remote = Git HP, Clé G
│   │       ├── git_prod.yml         ← Git Prod : bare, remote = Git HP, Clé F
│   │       └── app_prod.yml         ← App Prod : working tree, remote = Git Prod, Clé H
│   ├── playbooks/
│   │   ├── 01_sync_git_hp.yml       ← Bouton 1 : Git HP fetch depuis GitLab
│   │   ├── 02_deploy_hp.yml         ← Bouton 2 : App HP fetch + deploy
│   │   ├── 03_sync_git_prod.yml     ← Bouton 3 : Git Prod fetch depuis Git HP
│   │   ├── 04_deploy_prod.yml       ← Bouton 4 : App Prod fetch + deploy
│   │   └── 05_rollback_prod.yml     ← Bouton 5 : Rollback App Prod
│   └── roles/
│       ├── git_sync/
│       │   ├── tasks/
│       │   │   ├── main.yml         ← Dispatcher (bare vs working)
│       │   │   ├── sync_bare.yml    ← Logique dépôt bare (Git HP, Git Prod)
│       │   │   └── sync_working.yml ← Logique working tree (App HP, App Prod)
│       │   └── defaults/main.yml
│       └── deploy/
│           ├── tasks/
│           │   ├── main.yml
│           │   ├── install_deps.yml
│           │   ├── install_deps_python.yml
│           │   ├── run_migrations.yml
│           │   └── run_migrations_rollback.yml
│           ├── defaults/main.yml
│           └── templates/deploy.sh.j2
└── semaphore/
    └── project_config.md            ← Guide configuration Semaphore (5 templates)
```

---

## 4. Prérequis

### Sur l'Orchestrateur
- Ansible >= 2.14, Ansible Semaphore >= 2.9
- 4 clés SSH privées (A, B, C, D) dans `~/.ssh/`

### Sur Git HP et Git Prod
- Git >= 2.x, Python >= 3.8
- Répertoire `/opt/git/` pour les dépôts bare
- Utilisateur `deploy` avec authorized_keys configurés

### Sur App HP et App Prod
- Git >= 2.x, Python >= 3.8
- PHP >= 8.x (Composer) ou Python >= 3.8 (pip/venv) selon la stack
- Service systemd de l'application

---

## 5. Gestion des clés SSH

Voir le guide détaillé : [docs/ssh-key-management.md](docs/ssh-key-management.md)

### Résumé des 8 paires de clés

| Clé | Source        | Destination   | Type           |
|-----|---------------|---------------|----------------|
| A   | Orchestrateur | Git HP        | Ansible SSH    |
| B   | Orchestrateur | App HP        | Ansible SSH    |
| C   | Orchestrateur | Git Prod      | Ansible SSH    |
| D   | Orchestrateur | App Prod      | Ansible SSH    |
| E   | Git HP        | GitLab        | git (read-only)|
| F   | Git Prod      | Git HP        | git (read-only)|
| G   | App HP        | Git HP        | git (read-only)|
| H   | App Prod      | Git Prod      | git (read-only)|

Les clés E, F, G, H sont restreintes dans `authorized_keys` avec
`command="git-upload-pack ..."` — elles ne permettent pas d'ouvrir un shell.

---

## 6. Playbooks Ansible

Le rôle `git_sync` gère deux modes via la variable `git_repo_type` :

```
git_repo_type: bare     → sync_bare.yml    (Git HP, Git Prod)
               working  → sync_working.yml (App HP, App Prod)
```

Le rôle `deploy` gère les étapes applicatives (dépendances, migrations, reload).

---

## 7. Configuration d'Ansible Semaphore

Voir le guide détaillé : [semaphore/project_config.md](semaphore/project_config.md)

---

## 8. Stratégie de Rollback

Voir le guide détaillé : [docs/rollback-strategy.md](docs/rollback-strategy.md)

- Tags Git `v1.x.y` créés après validation HP, avant synchro Prod
- Bouton 5 "Rollback Prod" avec saisie de la version cible
- Tag de sauvegarde `backup/prod-pre-deploy-*` créé automatiquement avant chaque déploiement

---

## 9. Procédure opérationnelle standard

```
1. Dev push sur GitLab (branche main ou tag release)
        |
2. Responsable déploiement ouvre Semaphore
        |
3. [Bouton 1] Synchro Git HP    -> Git HP fetch depuis GitLab   (OK)
4. [Bouton 2] Déployer HP        -> App HP fetch depuis Git HP + deploy (OK)
        |
5. Validation QA sur Hors-Prod (tests fonctionnels)
        |
6. [Bouton 3] Synchro Git Prod   -> Git Prod fetch depuis Git HP (OK)
7. [Bouton 4] Déployer Prod      -> App Prod fetch depuis Git Prod + deploy (OK)
        |
8. Si problème -> [Bouton 5] Rollback Prod (saisir la version ex: v1.4.2)
```
