# Configuration d'Ansible Semaphore

## Prérequis

Ansible Semaphore installé sur l'Orchestrateur (`semaphore.prod.company.com`).

Installation recommandée via Docker Compose :
```yaml
version: "3.8"
services:
  semaphore:
    image: semaphoreui/semaphore:latest
    ports:
      - "3000:3000"
    environment:
      SEMAPHORE_DB_DIALECT: bolt
      SEMAPHORE_ADMIN: admin
      SEMAPHORE_ADMIN_PASSWORD: "changeme"
      SEMAPHORE_ADMIN_EMAIL: admin@company.com
    volumes:
      - semaphore_data:/var/lib/semaphore
      - /home/semaphore/.ssh:/root/.ssh:ro
volumes:
  semaphore_data:
```

---

## Étape 1 — Créer un Projet

**Projects → New Project** :
```
Nom : Pipeline Déploiement Sécurisé
```

---

## Étape 2 — Configurer le Repository

**Repositories → Add Repository** :
```yaml
Nom    : CI-CD Ansible
URL    : git@gitlab:mygroup/ci-cd.git
Branch : main
SSH Key: [clé de lecture du dépôt CI-CD sur GitLab]
```

---

## Étape 3 — Configurer le Key Store (8 clés)

**Key Store → Add Key** pour chacune des clés Ansible (A, B, C, D) :

| Nom dans Semaphore        | Clé privée à coller          | Usage                  |
|---------------------------|------------------------------|------------------------|
| `ssh-key-orch-git-hp`     | `~/.ssh/id_orch_git_hp`      | Ansible → Git HP       |
| `ssh-key-orch-app-hp`     | `~/.ssh/id_orch_app_hp`      | Ansible → App HP       |
| `ssh-key-orch-git-prod`   | `~/.ssh/id_orch_git_prod`    | Ansible → Git Prod     |
| `ssh-key-orch-app-prod`   | `~/.ssh/id_orch_app_prod`    | Ansible → App Prod     |

> Les clés E, F, G, H (git fetch entre serveurs) sont configurées directement
> dans `~/.ssh/` des serveurs cibles et ne passent pas par Semaphore.

---

## Étape 4 — Configurer les Inventaires

**Inventory → Add Inventory** :
```yaml
Nom    : hosts-pipeline
Type   : File
Fichier: ansible/inventory/hosts.yml
```

---

## Étape 5 — Configurer les Environnements

**Environment → Add Environment** :

### Environnement HP (pour Boutons 1 et 2)
```yaml
Nom  : env_hors_prod
JSON :
{
  "ANSIBLE_CONFIG": "ansible/ansible.cfg"
}
```

### Environnement Prod (pour Boutons 3, 4 et 5)
```yaml
Nom  : env_prod
JSON :
{
  "ANSIBLE_CONFIG": "ansible/ansible.cfg"
}
```

> Les secrets (mots de passe BDD) se définissent ici en JSON,
> ou via Ansible Vault avec `ANSIBLE_VAULT_PASSWORD_FILE` en variable d'env.

---

## Étape 6 — Créer les 5 Task Templates

---

### Template 1 : Synchro Git HP

```yaml
Nom         : 1. Synchro Git HP
Playbook    : ansible/playbooks/01_sync_git_hp.yml
Inventory   : hosts-pipeline
Repository  : CI-CD Ansible
Environment : env_hors_prod
SSH Key     : ssh-key-orch-git-hp
Description : Met à jour le dépôt bare sur Git HP depuis GitLab Central
```

Variables optionnelles (Survey) :
```yaml
- name    : git_branch
  title   : Branche à synchroniser
  default : main
  type    : String
  required: Non
```

---

### Template 2 : Déployer HP

```yaml
Nom         : 2. Déployer HP
Playbook    : ansible/playbooks/02_deploy_hp.yml
Inventory   : hosts-pipeline
Repository  : CI-CD Ansible
Environment : env_hors_prod
SSH Key     : ssh-key-orch-app-hp
Description : App HP tire le code depuis Git HP et déploie l'application
```

---

### Template 3 : Synchro Git Prod

```yaml
Nom         : 3. Synchro Git Prod
Playbook    : ansible/playbooks/03_sync_git_prod.yml
Inventory   : hosts-pipeline
Repository  : CI-CD Ansible
Environment : env_prod
SSH Key     : ssh-key-orch-git-prod
Description : Git Prod tire le code depuis Git HP (la Prod ne connaît pas GitLab)
```

---

### Template 4 : Déployer Prod

```yaml
Nom         : 4. Déployer Prod
Playbook    : ansible/playbooks/04_deploy_prod.yml
Inventory   : hosts-pipeline
Repository  : CI-CD Ansible
Environment : env_prod
SSH Key     : ssh-key-orch-app-prod
Description : App Prod tire le code depuis Git Prod et déploie l'application
```

---

### Template 5 : Rollback Prod

```yaml
Nom         : 5. Rollback Prod
Playbook    : ansible/playbooks/05_rollback_prod.yml
Inventory   : hosts-pipeline
Repository  : CI-CD Ansible
Environment : env_prod
SSH Key     : ssh-key-orch-app-prod
Description : Revient à une version précédente sur App Prod (checkout tag Git)
```

Variables obligatoires (Survey) :
```yaml
- name     : target_version
  title    : Version cible du rollback (ex: v1.4.2)
  description: Tag Git de la version vers laquelle revenir
  type     : String
  required : Oui
```

---

## Vue finale de l'interface Semaphore

```
┌──────────────────────────────────────────────────────────────────┐
│  Ansible Semaphore — Task Templates                              │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  [Run]  1. Synchro Git HP      Git HP <-- GitLab (bare fetch)   │
│  [Run]  2. Déployer HP         App HP <-- Git HP + deploy        │
│                                                                  │
│  [Run]  3. Synchro Git Prod    Git Prod <-- Git HP (bare fetch)  │
│  [Run]  4. Déployer Prod       App Prod <-- Git Prod + deploy    │
│                                                                  │
│  [Run]  5. Rollback Prod       [! saisir target_version]        │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## Gestion des accès utilisateurs

| Rôle Semaphore | Templates accessibles          | Profil                  |
|----------------|--------------------------------|-------------------------|
| `Viewer`       | Lecture logs uniquement        | Développeurs            |
| `Task Runner`  | Templates 1, 2, 3, 4           | Chefs de projet, QA     |
| `Manager`      | Tous (1 à 5, dont Rollback)    | Tech Lead, DevOps       |
| `Admin`        | Configuration complète         | Équipe Infrastructure   |

> Restreindre le Template 5 (Rollback Prod) aux rôles `Manager` et `Admin`.

---

## Notifications (optionnel)

```yaml
Notifications:
  - Event  : Task Failed
    Channel: Email ou Slack webhook
    Message : "ECHEC - {{ template_name }}"

  - Event  : Task Success
    Filter  : Templates 4 et 5 (Prod uniquement)
    Channel : Email
    Message : "OK - Déploiement/Rollback Production terminé"
```

---

## Audit et traçabilité

Semaphore conserve l'historique complet des tâches :
- **History** : chaque exécution avec horodatage, utilisateur, statut, durée
- **Logs** : sortie Ansible complète consultable par tâche
- Le `log_path` dans `ansible.cfg` centralise les logs sur l'Orchestrateur
