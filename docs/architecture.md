# Architecture — Pipeline de Déploiement Sécurisé

## Schéma global (vue réseau et flux de données)

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'primaryColor': '#2d3748', 'edgeLabelBackground':'#fff'}}}%%
flowchart TD
    DEV(["👨‍💻 Développeur"])
    GITLAB[("🗄️ GitLab Central\ngitlab.company.com\n[Stockage uniquement\nPas de pipeline actif]")]

    subgraph ZONE_DMZ["Zone DMZ / Hors-Production"]
        HP_GIT[("📁 Git Local #1\n/opt/apps/myapp\nremote: gitlab")]
        HP_APP["⚙️ Application\nHors-Prod\n(PHP/Python)"]
        HP_SRV["🖥️ Serveur Hors-Prod\nhors-prod.company.com"]
    end

    subgraph ZONE_PROD["Zone Production"]
        SEMAPHORE["🎛️ Ansible Semaphore\nsemaphore.prod.company.com\n[Orchestrateur — Single Pane of Glass]"]
        PROD_GIT[("📁 Git Local #2\n/opt/apps/myapp\nremote: staging → Hors-Prod")]
        PROD_APP["⚙️ Application\nProduction\n(PHP/Python)"]
        PROD_SRV["🖥️ Serveur Prod\nprod.company.com"]
    end

    DEV -->|"git push"| GITLAB

    SEMAPHORE -->|"SSH (Clé A)\n[Bouton 1] Synchro HP"| HP_SRV
    HP_SRV -->|"git fetch + checkout\n(Clé D)"| GITLAB
    HP_SRV --> HP_GIT
    HP_GIT --> HP_APP

    SEMAPHORE -->|"SSH (Clé A)\n[Bouton 2] Déployer HP"| HP_SRV
    HP_SRV -->|"scripts deploy\nmigrations, reload"| HP_APP

    SEMAPHORE -->|"SSH (Clé B)\n[Bouton 3] Synchro Prod"| PROD_SRV
    PROD_SRV -->|"git fetch + checkout via SSH\n(Clé C)"| HP_SRV
    HP_GIT -->|"expose dépôt\ngit over SSH"| PROD_SRV
    PROD_SRV --> PROD_GIT
    PROD_GIT --> PROD_APP

    SEMAPHORE -->|"SSH (Clé B)\n[Bouton 4] Déployer Prod"| PROD_SRV
    PROD_SRV -->|"scripts deploy\nmigrations, reload"| PROD_APP

    style ZONE_DMZ fill:#e6f3ff,stroke:#3182ce,color:#1a365d
    style ZONE_PROD fill:#fff5e6,stroke:#dd6b20,color:#7b341e
    style GITLAB fill:#fc6d26,color:#fff
    style SEMAPHORE fill:#2d3748,color:#fff
    style DEV fill:#48bb78,color:#fff
```

---

## Schéma des clés SSH

```mermaid
graph LR
    ORCH["🔑 Orchestrateur\n(Semaphore)"]
    HP["🖥️ Hors-Prod"]
    PROD["🖥️ Prod"]
    GITLAB["🗄️ GitLab"]

    ORCH -->|"Clé A\nid_orch_hp\n(Ansible → HP)"| HP
    ORCH -->|"Clé B\nid_orch_prod\n(Ansible → Prod)"| PROD
    PROD -->|"Clé C\nid_prod_hp_git\n(Git fetch Prod ← HP)"| HP
    HP   -->|"Clé D\nid_hp_gitlab\n(Git fetch HP ← GitLab)"| GITLAB

    style ORCH fill:#2d3748,color:#fff
    style HP fill:#3182ce,color:#fff
    style PROD fill:#dd6b20,color:#fff
    style GITLAB fill:#fc6d26,color:#fff
```

---

## Schéma de séquence — Déploiement complet

```mermaid
sequenceDiagram
    actor OPS as Opérateur
    participant SEM as Semaphore<br/>(Orchestrateur)
    participant HP as Serveur<br/>Hors-Prod
    participant GL as GitLab<br/>Central
    participant PROD as Serveur<br/>Prod

    Note over OPS,PROD: PHASE 1 — Synchronisation Hors-Prod

    OPS->>SEM: Clic "Synchro Hors-Prod"
    SEM->>HP: SSH + ansible playbook 01
    HP->>GL: git fetch --all (Clé D)
    GL-->>HP: objets Git
    HP->>HP: git checkout gitlab/main
    HP-->>SEM: OK (hash commit affiché)
    SEM-->>OPS: Tâche terminée ✓

    Note over OPS,PROD: PHASE 2 — Déploiement Hors-Prod

    OPS->>SEM: Clic "Déployer Hors-Prod"
    SEM->>HP: SSH + ansible playbook 02
    HP->>HP: install deps (pip/composer)
    HP->>HP: migrations BDD
    HP->>HP: reload service (systemd/php-fpm)
    HP-->>SEM: OK
    SEM-->>OPS: Tâche terminée ✓

    Note over OPS,PROD: VALIDATION QA sur Hors-Prod

    OPS->>OPS: Tests fonctionnels manuels

    Note over OPS,PROD: PHASE 3 — Synchronisation Prod

    OPS->>SEM: Clic "Synchro Prod"
    SEM->>PROD: SSH + ansible playbook 03
    PROD->>HP: git fetch staging (Clé C, git over SSH)
    HP-->>PROD: objets Git
    PROD->>PROD: git checkout staging/main (même hash)
    PROD-->>SEM: OK (hash commit affiché)
    SEM-->>OPS: Tâche terminée ✓

    Note over OPS,PROD: PHASE 4 — Déploiement Prod

    OPS->>SEM: Clic "Déployer Prod"
    SEM->>PROD: SSH + ansible playbook 04
    PROD->>PROD: install deps
    PROD->>PROD: migrations BDD
    PROD->>PROD: reload service
    PROD-->>SEM: OK
    SEM-->>OPS: Tâche terminée ✓
```

---

## Schéma de rollback

```mermaid
sequenceDiagram
    actor OPS as Opérateur
    participant SEM as Semaphore
    participant PROD as Serveur Prod

    Note over OPS,PROD: Problème détecté en production

    OPS->>SEM: Clic "Rollback Prod"
    SEM->>OPS: Saisie requise : target_version ?
    OPS->>SEM: v1.4.2
    SEM->>PROD: SSH + playbook 05_rollback_prod.yml
    PROD->>PROD: git checkout tags/v1.4.2
    PROD->>PROD: scripts déploiement (version précédente)
    PROD->>PROD: reload service
    PROD-->>SEM: OK
    SEM-->>OPS: Rollback terminé ✓
```

---

## Zones réseau et flux autorisés

```
┌─────────────────────────────────────────────────────────────────────┐
│                        ZONE PRODUCTION                              │
│                                                                     │
│   ┌──────────────────────┐      ┌──────────────────────────────┐   │
│   │  Ansible Semaphore   │      │     Serveur Prod             │   │
│   │  (Orchestrateur)     │─────►│  - App PHP/Python            │   │
│   │                      │ SSH  │  - Git local #2              │   │
│   │  Port 3000 (web UI)  │  B   │  - remote: staging → HP      │   │
│   └──────────────────────┘      └──────────────────────────────┘   │
│             │                                    │ SSH (Clé C)      │
│             │ SSH (Clé A)                        │ git fetch        │
│             │                                    ▼                  │
│             │               ┌─────────────────────────────────────┐ │
└─────────────│───────────────│         ZONE HORS-PROD              │ │
              │               │                                     │ │
              └──────────────►│  Serveur Hors-Prod                  │ │
                              │  - App PHP/Python                   │ │
                              │  - Git local #1                     │ │
                              │  - remote: gitlab → GitLab          │ │
                              │             │ SSH (Clé D)           │ │
                              │             │ git fetch             │ │
                              │             ▼                       │ │
                              │       ┌──────────┐                  │ │
                              │       │  GitLab  │                  │ │
                              │       │ Central  │                  │ │
                              │       └──────────┘                  │ │
                              └─────────────────────────────────────┘ │
                                                                       │
└──────────────────────────────────────────────────────────────────────┘

Flux réseau autorisés :
  Orchestrateur → Hors-Prod  : TCP/22 (SSH)     [Clé A]
  Orchestrateur → Prod        : TCP/22 (SSH)     [Clé B]
  Prod → Hors-Prod            : TCP/22 (SSH)     [Clé C, git only]
  Hors-Prod → GitLab          : TCP/22 (SSH)     [Clé D, git only]

Flux interdits (principe du moindre privilège) :
  GitLab → tout serveur interne   : BLOQUE
  Internet → serveurs internes     : BLOQUE
  Prod → Internet                  : BLOQUE (sauf si dépôts packages internes)
```

---

## Décisions d'architecture

| Décision | Choix | Justification |
|----------|-------|---------------|
| Pas de `git pull` | `git fetch` + `git checkout` | Pas de merge automatique, contrôle total sur la version |
| Dépôts non-bare | Working tree actif | Le code est exécuté directement depuis le dépôt |
| Remote nommé `gitlab` sur HP | Pas `origin` | Clarté, évite toute ambiguïté si on ajoute des remotes |
| Remote nommé `staging` sur Prod | Pointe vers HP via SSH | Isolation réseau, la Prod ne connaît pas GitLab |
| Tags Git pour la Prod | `v1.x.y` sémantique | Rollback déterministe, traçabilité |
| Semaphore en zone Prod | Accès SSH vers HP et Prod | Un seul point d'entrée opérationnel |
| 4 boutons séparés | Synchro et Deploy distincts | Permet validation intermédiaire sans déployer automatiquement |
