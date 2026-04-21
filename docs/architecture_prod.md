# Architecture — Zone Production (PROD)

> Document dédié à la zone Production. Pour la zone Hors-Prod, voir [architecture_hp.md](./architecture_hp.md).
> Pour le mécanisme de validation entre zones, voir [validation-flag.md](./validation-flag.md).

---

## Serveurs de la zone PROD

| Serveur              | Hostname                      | Zone          | Rôle                                              |
|----------------------|-------------------------------|---------------|---------------------------------------------------|
| GitLab Central       | gitlab.company.com            | Externe/DMZ   | Stockage source de vérité (pas de CI/CD)          |
| Git+Ansible HP       | git-ansible-hp.company.com    | Hors-Prod     | Source pour la PROD (dépôt validé)                |
| Semaphore Prod       | semaphore-prod.company.com    | Production    | Semaphore UI + Ansible control node Prod          |
| Git+Ansible Prod     | git-ansible-prod.company.com  | Production    | Dépôt bare miroir + Ansible installé              |
| App Server(s) Prod   | app[n]-prod.company.com       | Production    | N serveurs applicatifs Prod (PHP/Python/Node.js/Java/Go) |

> **Git+Ansible Prod** : ce serveur héberge à la fois le dépôt bare Git
> et une installation Ansible. La PROD ne tire que depuis Git+Ansible HP
> (jamais directement depuis GitLab), et uniquement après validation du flag HP.

---

## Flux réseau — Zone PROD

```mermaid
flowchart TD
    GIT_HP[("📦 Git+Ansible HP\ngit-ansible-hp\n[Source validée HP]")]

    subgraph ZONE_PROD["Zone Production"]
        SEM_PROD["🎛️ Semaphore Prod\nsemaphore-prod\n[Control node Prod]"]
        GIT_PROD[("📦 Git+Ansible Prod\ngit-ansible-prod\n/opt/git/<repo>.git")]
        APP_PROD["⚙️ App Server(s) Prod\napp[n]-prod\n/opt/apps/<repo>"]
    end

    SEM_PROD -->|"SSH Clé E — Ansible"| GIT_PROD
    GIT_PROD -->|"SSH Clé F — check flag + git fetch"| GIT_HP
    SEM_PROD -->|"SSH Clé G — Ansible"| APP_PROD
    APP_PROD -->|"git fetch Clé H"| GIT_PROD

    style ZONE_PROD fill:#fff5e6,stroke:#dd6b20,color:#7b341e
    style GIT_HP fill:#3182ce,color:#fff
    style SEM_PROD fill:#c05621,color:#fff
    style GIT_PROD fill:#dd6b20,color:#fff
```

---

## Clés SSH de la zone PROD (E, F, G, H)

```mermaid
graph LR
    SEM_PROD["🎛️ Semaphore Prod"]
    GIT_PROD["📦 Git+Ansible Prod"]
    APP_PROD["⚙️ App Prod"]
    GIT_HP["📦 Git+Ansible HP"]

    SEM_PROD -->|"Clé E — SSH Ansible"| GIT_PROD
    GIT_PROD -->|"Clé F — SSH check flag + git fetch"| GIT_HP
    SEM_PROD -->|"Clé G — SSH Ansible"| APP_PROD
    APP_PROD -->|"Clé H — git fetch (read-only)"| GIT_PROD

    style SEM_PROD fill:#c05621,color:#fff
    style GIT_PROD fill:#dd6b20,color:#fff
    style APP_PROD fill:#f6ad55,color:#1a1a1a
    style GIT_HP fill:#3182ce,color:#fff
```

| Clé | Source             | Destination        | Type accès                    | Fichier clé privée (sur la source)      |
|-----|--------------------|--------------------|-------------------------------|-----------------------------------------|
| E   | Semaphore Prod     | Git+Ansible Prod   | SSH Ansible (complet)         | `~/.ssh/id_semaphore_prod`              |
| F   | Git+Ansible Prod   | Git+Ansible HP     | SSH check flag + git fetch    | `/opt/keys/id_gitprod_to_githp`         |
| G   | Semaphore Prod     | App Server(s) Prod | SSH Ansible (complet)         | `~/.ssh/id_semaphore_prod_apps`         |
| H   | App Server(s) Prod | Git+Ansible Prod   | git fetch (read-only)         | `/opt/keys/id_appprod_to_gitprod`       |

> La clé H est strictement read-only (`command="git-upload-pack ..."` dans `authorized_keys`).
> La clé F a accès SSH complet vers Git+Ansible HP pour vérifier le flag et fetcher.
> Voir [ssh-key-management_prod.md](./ssh-key-management_prod.md) pour la configuration complète.

---

## Les 3 boutons Semaphore PROD

| Bouton | Playbook                        | Cible Ansible                 | Action                                              |
|--------|---------------------------------|-------------------------------|-----------------------------------------------------|
| 1      | `prod/01_sync_git_prod.yml`     | `git_ansible_prod`            | Vérifie flag HP, fetche Git HP → miroir bare Prod   |
| 2      | `prod/02_deploy_prod.yml`       | `git_ansible_prod` + App Prod | Sauvegarde BDD + déploiement sur App Prod           |
| 3      | `prod/03_rollback_prod.yml`     | App Prod                      | Sauvegarde BDD + retour à une version précédente    |

> Le **Bouton 1 Prod est bloqué** si le Bouton 3 HP n'a pas été exécuté.
> Voir [validation-flag.md](./validation-flag.md) pour le détail.

---

## Schéma de séquence — Déploiement PROD complet

```mermaid
sequenceDiagram
    actor OPS_PROD as Opérateur Prod
    participant SPROD as Semaphore Prod
    participant GPROD as Git+Ansible Prod
    participant GHP as Git+Ansible HP
    participant APROD as App Prod

    Note over OPS_PROD,APROD: ─── ZONE PROD (après validation HP) ───

    OPS_PROD->>SPROD: Bouton 1 — Synchro Git Prod
    SPROD->>GPROD: SSH Clé E + playbook 01
    GPROD->>GHP: SSH Clé F — vérif. flag validated/hp/<repo>
    GHP-->>GPROD: flag présent ✓
    GPROD->>GHP: git fetch (Clé F)
    GHP-->>GPROD: objets Git
    GPROD-->>SPROD: miroir Prod mis à jour

    OPS_PROD->>SPROD: Bouton 2 — Déployer Prod
    SPROD->>APROD: SSH Clé G + playbook 02
    APROD->>APROD: sauvegarde BDD (pg_dump)
    APROD->>GPROD: git fetch (Clé H)
    GPROD-->>APROD: objets Git
    APROD->>APROD: checkout + deps + migrations + reload
    APROD->>APROD: health check HTTP
    APROD-->>SPROD: déploiement OK
```

---

## Configuration par dépôt (zone PROD)

Les champs PROD pertinents du fichier `ansible/repos/<repo_name>.yml` :

```yaml
# ansible/repos/myapp.yml (extrait zone PROD)
repo_name: myapp
git_branch: main

app_root: "/opt/apps/myapp"
app_service_name: myapp
app_user: deploy
app_group: deploy

# Serveurs cibles Prod (liste configurable par dépôt)
app_servers_prod:
  - app1-prod.company.com
  - app2-prod.company.com

run_migrations: true
db_host_prod: prod-db.company.com
db_name_prod: myapp_prod
db_user_prod: app_user
```

---

## Nature des dépôts Git en zone PROD

```
Git+Ansible Prod  (/opt/git/<repo>.git)   → BARE REPOSITORY
  Dépôt bare = uniquement les objets Git, pas de working tree.
  Miroir de Git+Ansible HP, après vérification du flag.
  La PROD ne tire JAMAIS directement depuis GitLab.

App Server(s) Prod  (/opt/apps/<repo>)    → WORKING TREE
  Working tree = dépôt + fichiers sources visibles.
  C'est ici que le code de production s'exécute.
  Sauvegarde BDD (pg_dump) effectuée AVANT chaque déploiement.
```

---

## Zones réseau et flux autorisés (PROD)

```
┌─────────────────────────────────────────────────────────────────────────┐
│  ZONE PROD                                                              │
│                                                                         │
│  ┌──────────────────┐   Clé E   ┌──────────────────────────────────┐   │
│  │  Semaphore Prod  │──────────►│  Git+Ansible Prod                │   │
│  │                  │   Clé G   │  (bare /opt/git/*.git)            │   │
│  │                  │──────────►│  App Server(s) Prod              │   │
│  └──────────────────┘          └──────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘

Connexions git initiées PAR les serveurs PROD (pull model) :

  Git+Ansible Prod   --Clé F--> Git+Ansible HP    (SSH check flag + git fetch)
  App Server(s) Prod --Clé H--> Git+Ansible Prod  (git fetch)

Règles firewall minimales pour la zone PROD :
  Semaphore Prod     --> Git+Ansible Prod  : TCP/22   AUTORISER
  Semaphore Prod     --> App Prod          : TCP/22   AUTORISER
  Git+Ansible Prod   --> Git+Ansible HP    : TCP/22   AUTORISER
  App Prod           --> Git+Ansible Prod  : TCP/22   AUTORISER
  Semaphore Prod     --> Zone HP           : *        BLOQUER (sauf Git HP TCP/22)
  Tout autre flux                          : *        BLOQUER
```

---

## Décisions d'architecture (zone PROD)

| Décision | Choix | Justification |
|----------|-------|---------------|
| Semaphore Prod indépendant | Instance dédiée Prod | Isolation complète ; un incident HP n'affecte pas la Prod |
| Git+Ansible Prod combiné | Un seul serveur | Simplifie la gestion des clés ; le serveur git est aussi le control node |
| Dépôts bare sur Git+Ansible Prod | `/opt/git/<repo>.git` | Pas de working tree = pas de risque d'exécution accidentelle |
| Pull model exclusif | Toujours `git fetch`, jamais de push vers les cibles | La PROD ne peut recevoir que ce que HP a validé |
| Clé H read-only | `command="git-upload-pack ..."` | Même compromise, impossible d'ouvrir un shell ou d'écrire |
| Sauvegarde BDD avant PROD | `pg_dump -Fc` avant chaque déploiement | Rollback possible si migration échoue |
| Rolling deploy PROD | `serial: 1` | Déploiement serveur par serveur, disponibilité partielle maintenue |
| Fetch + Checkout (pas git pull) | `git fetch` puis `git checkout --force` | Aucun merge automatique, contrôle total sur la version déployée |
