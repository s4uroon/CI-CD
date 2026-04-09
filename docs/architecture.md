# Architecture — Pipeline de Déploiement Sécurisé

## Infrastructure : 6 serveurs distincts

| Serveur            | Hostname                  | Zone          | Rôle                                      |
|--------------------|---------------------------|---------------|-------------------------------------------|
| GitLab Central     | gitlab.company.com        | Externe/DMZ   | Stockage source de vérité (pas de CI/CD)  |
| Git Server HP      | git-hp.company.com        | Hors-Prod     | Dépôt bare, miroir de GitLab              |
| App Server HP      | app-hp.company.com        | Hors-Prod     | Application PHP/Python + working tree     |
| Git Server Prod    | git-prod.company.com      | Production    | Dépôt bare, miroir de Git HP             |
| App Server Prod    | app-prod.company.com      | Production    | Application PHP/Python + working tree     |
| Orchestrateur      | semaphore.prod.company.com| Production    | Ansible Semaphore, point d'entrée unique  |

---

## Schéma global — Vue réseau et flux de données

```mermaid
flowchart TD
    DEV(["👨‍💻 Développeur"])
    GITLAB[("🗄️ GitLab Central\ngitlab.company.com\n[Stockage uniquement]")]

    subgraph ZONE_HP["Zone Hors-Production (DMZ)"]
        GIT_HP[("📦 Git Server HP\ngit-hp.company.com\nDépôt bare\n/opt/git/myapp.git")]
        APP_HP["⚙️ App Server HP\napp-hp.company.com\n/opt/apps/myapp\n[working tree]"]
    end

    subgraph ZONE_PROD["Zone Production"]
        SEMAPHORE["🎛️ Orchestrateur\nsemaphore.prod.company.com\nAnsible Semaphore"]
        GIT_PROD[("📦 Git Server Prod\ngit-prod.company.com\nDépôt bare\n/opt/git/myapp.git")]
        APP_PROD["⚙️ App Server Prod\napp-prod.company.com\n/opt/apps/myapp\n[working tree]"]
    end

    DEV -->|"git push"| GITLAB

    SEMAPHORE -->|"SSH Clé A\n[Bouton 1] Synchro Git HP"| GIT_HP
    GIT_HP -->|"git fetch\n(Clé E)"| GITLAB

    SEMAPHORE -->|"SSH Clé B\n[Bouton 2] Déployer HP"| APP_HP
    APP_HP -->|"git fetch\n(Clé G)"| GIT_HP

    SEMAPHORE -->|"SSH Clé C\n[Bouton 3] Synchro Git Prod"| GIT_PROD
    GIT_PROD -->|"git fetch\n(Clé F)"| GIT_HP

    SEMAPHORE -->|"SSH Clé D\n[Bouton 4] Déployer Prod"| APP_PROD
    APP_PROD -->|"git fetch\n(Clé H)"| GIT_PROD

    style ZONE_HP fill:#e6f3ff,stroke:#3182ce,color:#1a365d
    style ZONE_PROD fill:#fff5e6,stroke:#dd6b20,color:#7b341e
    style GITLAB fill:#fc6d26,color:#fff
    style SEMAPHORE fill:#2d3748,color:#fff
    style DEV fill:#48bb78,color:#fff
    style GIT_HP fill:#3182ce,color:#fff
    style GIT_PROD fill:#dd6b20,color:#fff
```

---

## Schéma des 8 clés SSH

```mermaid
graph LR
    ORCH["🔑 Orchestrateur"]
    GIT_HP["📦 Git HP"]
    APP_HP["⚙️ App HP"]
    GIT_PROD["📦 Git Prod"]
    APP_PROD["⚙️ App Prod"]
    GITLAB["🗄️ GitLab"]

    ORCH -->|"Clé A — Ansible pilote"| GIT_HP
    ORCH -->|"Clé B — Ansible pilote"| APP_HP
    ORCH -->|"Clé C — Ansible pilote"| GIT_PROD
    ORCH -->|"Clé D — Ansible pilote"| APP_PROD

    GIT_HP   -->|"Clé E — git fetch"| GITLAB
    GIT_PROD -->|"Clé F — git fetch"| GIT_HP
    APP_HP   -->|"Clé G — git fetch"| GIT_HP
    APP_PROD -->|"Clé H — git fetch"| GIT_PROD

    style ORCH fill:#2d3748,color:#fff
    style GIT_HP fill:#3182ce,color:#fff
    style APP_HP fill:#63b3ed,color:#fff
    style GIT_PROD fill:#dd6b20,color:#fff
    style APP_PROD fill:#f6ad55,color:#1a1a1a
    style GITLAB fill:#fc6d26,color:#fff
```

| Clé | Source       | Destination  | Type accès       | Fichier clé privée (sur la source)          |
|-----|--------------|--------------|------------------|---------------------------------------------|
| A   | Orchestrateur| Git HP       | SSH Ansible      | `~/.ssh/id_orch_git_hp`                    |
| B   | Orchestrateur| App HP       | SSH Ansible      | `~/.ssh/id_orch_app_hp`                    |
| C   | Orchestrateur| Git Prod     | SSH Ansible      | `~/.ssh/id_orch_git_prod`                  |
| D   | Orchestrateur| App Prod     | SSH Ansible      | `~/.ssh/id_orch_app_prod`                  |
| E   | Git HP       | GitLab       | git (read-only)  | `/home/deploy/.ssh/id_githp_gitlab`        |
| F   | Git Prod     | Git HP       | git (read-only)  | `/home/deploy/.ssh/id_gitprod_githp`       |
| G   | App HP       | Git HP       | git (read-only)  | `/home/deploy/.ssh/id_apphp_githp`         |
| H   | App Prod     | Git Prod     | git (read-only)  | `/home/deploy/.ssh/id_appprod_gitprod`     |

---

## Schéma de séquence — Déploiement complet

```mermaid
sequenceDiagram
    actor OPS as Opérateur
    participant SEM as Semaphore<br/>(Orchestrateur)
    participant GHP as Git Server HP<br/>(bare)
    participant AHP as App Server HP<br/>(working tree)
    participant GL as GitLab
    participant GPROD as Git Server Prod<br/>(bare)
    participant APROD as App Server Prod<br/>(working tree)

    Note over OPS,APROD: BOUTON 1 — Synchro Git HP

    OPS->>SEM: Clic "Synchro Git HP"
    SEM->>GHP: SSH (Clé A) + playbook 01
    GHP->>GL: git fetch --all --tags (Clé E)
    GL-->>GHP: objets Git
    GHP-->>SEM: Miroir bare mis à jour
    SEM-->>OPS: OK — commit affiché

    Note over OPS,APROD: BOUTON 2 — Déployer HP

    OPS->>SEM: Clic "Déployer HP"
    SEM->>AHP: SSH (Clé B) + playbook 02
    AHP->>GHP: git fetch (Clé G)
    GHP-->>AHP: objets Git
    AHP->>AHP: git checkout + deps + migrations
    AHP->>AHP: reload service
    AHP-->>SEM: OK
    SEM-->>OPS: OK — Validation QA

    Note over OPS,APROD: VALIDATION QA sur Hors-Prod

    OPS->>OPS: Tests fonctionnels

    Note over OPS,APROD: BOUTON 3 — Synchro Git Prod

    OPS->>SEM: Clic "Synchro Git Prod"
    SEM->>GPROD: SSH (Clé C) + playbook 03
    GPROD->>GHP: git fetch --all --tags (Clé F)
    GHP-->>GPROD: objets Git
    GPROD-->>SEM: Miroir bare Prod mis à jour
    SEM-->>OPS: OK — commit affiché

    Note over OPS,APROD: BOUTON 4 — Déployer Prod

    OPS->>SEM: Clic "Déployer Prod"
    SEM->>APROD: SSH (Clé D) + playbook 04
    APROD->>GPROD: git fetch (Clé H)
    GPROD-->>APROD: objets Git
    APROD->>APROD: git checkout + sauvegarde BDD
    APROD->>APROD: deps + migrations + reload
    APROD-->>SEM: OK
    SEM-->>OPS: OK
```

---

## Nature des dépôts Git

```
Git Server HP    (/opt/git/myapp.git)   → BARE REPOSITORY
Git Server Prod  (/opt/git/myapp.git)   → BARE REPOSITORY

  Dépôt bare = uniquement les objets Git, pas de working tree.
  Sert de miroir/relais. Aucun code ne s'exécute ici.

  Contenu :
    HEAD  branches/  config  description
    hooks/  info/  objects/  packed-refs  refs/

App Server HP    (/opt/apps/myapp)      → WORKING TREE
App Server Prod  (/opt/apps/myapp)      → WORKING TREE

  Working tree = dépôt + fichiers sources visibles.
  C'est ici que le code PHP/Python s'exécute.

  Contenu :
    .git/   index.php   app/   config/   ...
```

---

## Zones réseau et flux autorisés

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  ZONE PRODUCTION                                                             │
│                                                                              │
│  ┌─────────────────────┐    Clé A    ┌───────────────────────┐              │
│  │  Orchestrateur      │────────────►│  Git Server HP        │              │
│  │  (Semaphore)        │    Clé B    ├───────────────────────┤              │
│  │                     │────────────►│  App Server HP        │  ZONE HP     │
│  │                     │    Clé C    ├───────────────────────┤              │
│  │                     │────────────►│  Git Server Prod      │              │
│  │                     │    Clé D    ├───────────────────────┤              │
│  │                     │────────────►│  App Server Prod      │              │
│  └─────────────────────┘            └───────────────────────┘              │
└──────────────────────────────────────────────────────────────────────────────┘

Connexions git initiées PAR les serveurs cibles :

  Git HP   --Clé E--> GitLab   (git fetch depuis HP vers GitLab)
  Git Prod --Clé F--> Git HP   (git fetch depuis Prod vers HP)
  App HP   --Clé G--> Git HP   (git fetch pour déploiement)
  App Prod --Clé H--> Git Prod (git fetch pour déploiement)

Règles firewall minimales requises :
  Orchestrateur  --> Git HP    : TCP/22
  Orchestrateur  --> App HP    : TCP/22
  Orchestrateur  --> Git Prod  : TCP/22
  Orchestrateur  --> App Prod  : TCP/22
  Git HP         --> GitLab    : TCP/22
  Git Prod       --> Git HP    : TCP/22
  App HP         --> Git HP    : TCP/22
  App Prod       --> Git Prod  : TCP/22

Tout autre flux : BLOQUE
```

---

## Décisions d'architecture

| Décision | Choix | Justification |
|----------|-------|---------------|
| Dépôts bare sur Git HP et Git Prod | `/opt/git/myapp.git` | Pas de working tree = pas de risque d'exécution accidentelle, usage pur miroir |
| Working tree sur App HP et App Prod | `/opt/apps/myapp` | Le code s'exécute directement depuis le dépôt cloné |
| La Prod ne connaît pas GitLab | Git Prod → Git HP uniquement | Isolation réseau stricte, la Prod ne peut pas aller chercher n'importe quoi |
| Fetch + Checkout (pas git pull) | `git fetch` puis `git checkout --force` | Aucun merge automatique, contrôle total sur la version déployée |
| Tags Git pour le rollback | `v1.x.y` sémantique | Rollback déterministe vers n'importe quelle version antérieure |
| 4 boutons séparés | Synchro et Deploy distincts | Validation intermédiaire entre chaque étape, pas de déploiement automatique |
| Clés SSH read-only (F, G, H) | `command=git-upload-pack` dans `authorized_keys` | Même si une clé est compromise, impossible d'ouvrir un shell ou d'écrire |
