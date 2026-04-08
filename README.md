# Pipeline de Déploiement Sécurisé — Guide d'Architecture et d'Implémentation

> **Stack** : GitLab · Ansible · Ansible Semaphore
> **Modèle** : Pull en cascade (GitLab → Hors-Prod → Prod)
> **Déclenchement** : Manuel via Semaphore (Single Pane of Glass)

---

## Table des matières

1. [Vue d'ensemble](#1-vue-densemble)
2. [Structure du dépôt](#2-structure-du-dépôt)
3. [Prérequis](#3-prérequis)
4. [Configuration Git sur les serveurs cibles](#4-configuration-git-sur-les-serveurs-cibles)
5. [Gestion des clés SSH](#5-gestion-des-clés-ssh)
6. [Playbooks Ansible](#6-playbooks-ansible)
7. [Configuration d'Ansible Semaphore](#7-configuration-dansible-semaphore)
8. [Stratégie de Rollback](#8-stratégie-de-rollback)
9. [Procédure opérationnelle standard](#9-procédure-opérationnelle-standard)

---

## 1. Vue d'ensemble

Le flux de déploiement suit un modèle **Pull en cascade strict** :

```
Développeur  ──push──►  GitLab Central
                              │
                    [Bouton 1 Semaphore]
                              │ git fetch/checkout
                              ▼
                      Serveur Hors-Prod
                      (Git local #1)
                              │
                    [Bouton 2 Semaphore]
                              │ scripts déploiement
                              ▼
                     App Hors-Prod ✓
                              │
                    [Bouton 3 Semaphore]
                              │ git fetch/checkout (SSH)
                              ▼
                        Serveur Prod
                        (Git local #2)
                              │
                    [Bouton 4 Semaphore]
                              │ scripts déploiement
                              ▼
                       App Prod ✓
```

Les 4 boutons dans Semaphore correspondent aux 4 Task Templates :

| # | Bouton Semaphore       | Playbook Ansible              | Action                                          |
|---|------------------------|-------------------------------|-------------------------------------------------|
| 1 | Synchro Hors-Prod      | `01_sync_hors_prod.yml`       | Hors-Prod pull depuis GitLab                    |
| 2 | Déployer Hors-Prod     | `02_deploy_hors_prod.yml`     | Exécution des scripts de déploiement Hors-Prod  |
| 3 | Synchro Prod           | `03_sync_prod.yml`            | Prod pull depuis Hors-Prod (via SSH)            |
| 4 | Déployer Prod          | `04_deploy_prod.yml`          | Exécution des scripts de déploiement Prod       |

Pour le schéma d'architecture détaillé (avec diagrammes Mermaid), voir [docs/architecture.md](docs/architecture.md).

---

## 2. Structure du dépôt

```
CI-CD/
├── README.md                          ← Ce fichier
├── docs/
│   ├── architecture.md                ← Schéma et explication détaillée
│   ├── ssh-key-management.md          ← Gestion et répartition des clés SSH
│   └── rollback-strategy.md           ← Procédure de rollback
├── ansible/
│   ├── ansible.cfg                    ← Configuration Ansible globale
│   ├── inventory/
│   │   ├── hosts.yml                  ← Inventaire des serveurs
│   │   └── group_vars/
│   │       ├── all.yml                ← Variables communes
│   │       ├── hors_prod.yml          ← Variables Hors-Prod
│   │       └── prod.yml               ← Variables Production
│   ├── playbooks/
│   │   ├── 01_sync_hors_prod.yml      ← Bouton 1 : Synchro Hors-Prod
│   │   ├── 02_deploy_hors_prod.yml    ← Bouton 2 : Déployer Hors-Prod
│   │   ├── 03_sync_prod.yml           ← Bouton 3 : Synchro Prod
│   │   └── 04_deploy_prod.yml         ← Bouton 4 : Déployer Prod
│   └── roles/
│       ├── git_sync/                  ← Rôle : synchronisation Git
│       │   ├── tasks/main.yml
│       │   └── defaults/main.yml
│       └── deploy/                    ← Rôle : déploiement applicatif
│           ├── tasks/main.yml
│           ├── defaults/main.yml
│           └── templates/
│               └── deploy.sh.j2
└── semaphore/
    └── project_config.md              ← Guide de configuration Semaphore
```

---

## 3. Prérequis

### Sur l'Orchestrateur (Semaphore)
- Ansible >= 2.14
- Ansible Semaphore >= 2.9
- Git >= 2.x
- Accès SSH (sans mot de passe) vers Hors-Prod et Prod

### Sur Hors-Prod et Prod
- Git >= 2.x
- Python >= 3.8 (pour le module Ansible)
- Utilisateur de déploiement dédié (ex: `deploy`)
- Accès SSH depuis l'Orchestrateur

### Sur Prod (supplémentaire)
- Accès SSH vers Hors-Prod (pour git fetch)

### Sur GitLab
- Un utilisateur dédié de lecture (ex: `ci-reader`) avec accès aux dépôts
- Clé SSH de lecture autorisée

---

## 4. Configuration Git sur les serveurs cibles

### Principes clés

- **Fetch + Checkout** (jamais `git pull`) : évite les conflits de merge automatique.
- **Branches fixes** : on déploie toujours depuis une branche définie (`main`, `release`...).
- **Tags de version** : la synchro Prod checkout sur un tag signé ou un hash précis.
- Le dépôt local est **en mode non-bare** (working tree actif) pour exécuter le code.

### Hors-Prod : Remote = GitLab Central

```bash
# Initialisation (jouée une seule fois par Ansible au premier déploiement)
git clone --origin gitlab git@gitlab.company.com:mygroup/myapp.git /opt/apps/myapp
cd /opt/apps/myapp
git remote set-url gitlab git@gitlab.company.com:mygroup/myapp.git
```

### Prod : Remote = Hors-Prod (via SSH)

```bash
# La Prod tire depuis le dépôt local de la Hors-Prod via SSH
git clone --origin staging \
  deploy@hors-prod.company.com:/opt/apps/myapp \
  /opt/apps/myapp
```

> Le module `ansible.builtin.git` gère l'initialisation et les mises à jour.
> Voir les rôles dans `ansible/roles/git_sync/`.

---

## 5. Gestion des clés SSH

Voir le guide détaillé : [docs/ssh-key-management.md](docs/ssh-key-management.md)

### Résumé des paires de clés

| Paire | De               | Vers            | Usage                              | Fichier clé privée                   |
|-------|------------------|-----------------|------------------------------------|--------------------------------------|
| A     | Orchestrateur    | Hors-Prod       | Ansible SSH (playbooks 1 & 2)      | `/home/semaphore/.ssh/id_orch_hp`    |
| B     | Orchestrateur    | Prod            | Ansible SSH (playbooks 3 & 4)      | `/home/semaphore/.ssh/id_orch_prod`  |
| C     | Prod             | Hors-Prod       | Git fetch Prod <- Hors-Prod        | `/home/deploy/.ssh/id_prod_hp_git`   |
| D     | Hors-Prod        | GitLab          | Git fetch Hors-Prod <- GitLab      | `/home/deploy/.ssh/id_hp_gitlab`     |

---

## 6. Playbooks Ansible

Voir les fichiers dans `ansible/playbooks/` et `ansible/roles/`.

### Logique des rôles

```
roles/git_sync/   ->  fetch + checkout propre (sans merge)
roles/deploy/     ->  install deps + migrations + reload service
```

---

## 7. Configuration d'Ansible Semaphore

Voir le guide détaillé : [semaphore/project_config.md](semaphore/project_config.md)

### Les 4 Task Templates à créer

| Template               | Playbook                   | Inventory    | Environnement |
|------------------------|----------------------------|--------------|---------------|
| Synchro Hors-Prod      | `01_sync_hors_prod.yml`    | `hosts.yml`  | `env_hp`      |
| Déployer Hors-Prod     | `02_deploy_hors_prod.yml`  | `hosts.yml`  | `env_hp`      |
| Synchro Prod           | `03_sync_prod.yml`         | `hosts.yml`  | `env_prod`    |
| Déployer Prod          | `04_deploy_prod.yml`       | `hosts.yml`  | `env_prod`    |

---

## 8. Stratégie de Rollback

Voir le guide détaillé : [docs/rollback-strategy.md](docs/rollback-strategy.md)

### Résumé

Le rollback repose sur des **tags Git versionnés** et un **5e Task Template** dans Semaphore :

- `Rollback Prod` -> paramètre `target_version` (ex: `v1.4.2`) -> checkout du tag + redéploiement
- Aucune perte de code, aucun besoin d'accès manuel au serveur

---

## 9. Procédure opérationnelle standard

```
1. Dev push sur GitLab (branche main ou tag release)
        |
2. Responsable déploiement ouvre Semaphore
        |
3. Clic "Synchro Hors-Prod"   -> Attente fin (vert OK)
4. Clic "Déployer Hors-Prod"  -> Validation fonctionnelle sur Hors-Prod
        |
5. Validation OK par l'équipe QA / PO
        |
6. Clic "Synchro Prod"        -> Attente fin (vert OK)
7. Clic "Déployer Prod"        -> Validation production
        |
8. Si problème -> Clic "Rollback Prod" (saisir version précédente)
```
