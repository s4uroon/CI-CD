# Configuration d'Ansible Semaphore

## Prérequis

Ansible Semaphore doit être installé et accessible via son interface web
(par défaut : `http://semaphore.prod.company.com:3000`).

Installation recommandée via Docker Compose :
```yaml
# docker-compose.yml sur l'Orchestrateur
version: "3.8"
services:
  semaphore:
    image: semaphoreui/semaphore:latest
    ports:
      - "3000:3000"
    environment:
      SEMAPHORE_DB_DIALECT: bolt
      SEMAPHORE_ADMIN: admin
      SEMAPHORE_ADMIN_PASSWORD: "changeme"  # A changer !
      SEMAPHORE_ADMIN_EMAIL: admin@company.com
    volumes:
      - semaphore_data:/var/lib/semaphore
      - /home/semaphore/.ssh:/root/.ssh:ro   # Montage des clés SSH
volumes:
  semaphore_data:
```

---

## Étape 1 — Créer un Projet

Dans Semaphore → **Projects** → **New Project** :

```
Nom : Pipeline Déploiement Sécurisé
```

---

## Étape 2 — Configurer le Repository (Source du code Ansible)

Menu **Repositories** → **Add Repository** :

```yaml
Nom           : CI-CD Ansible
URL           : git@gitlab:mygroup/ci-cd.git
Branch        : main
SSH Key       : [clé D — id_hp_gitlab sur l'Orchestrateur, ou clé dédiée lecture GitLab]
```

> Ce dépôt contient les playbooks et rôles Ansible de ce projet.

---

## Étape 3 — Configurer les Clés SSH

Menu **Key Store** → **Add Key** pour chaque clé :

### Clé A — Orchestrateur vers Hors-Prod

```yaml
Nom  : ssh-key-orch-hors-prod
Type : SSH Key
Clé privée : [coller le contenu de ~/.ssh/id_orch_hp]
```

### Clé B — Orchestrateur vers Prod

```yaml
Nom  : ssh-key-orch-prod
Type : SSH Key
Clé privée : [coller le contenu de ~/.ssh/id_orch_prod]
```

### Clé GitLab — Lecture du dépôt CI-CD

```yaml
Nom  : ssh-key-gitlab-reader
Type : SSH Key
Clé privée : [clé de lecture du dépôt CI-CD sur GitLab]
```

---

## Étape 4 — Configurer les Inventaires

Menu **Inventory** → **Add Inventory** :

### Inventaire principal

```yaml
Nom    : hosts-production
Type   : File
Fichier: ansible/inventory/hosts.yml
Creds  : (laisser vide, les clés SSH sont dans ansible.cfg / group_vars)
```

---

## Étape 5 — Configurer les Environnements

Menu **Environment** → **Add Environment** :

### Environnement Hors-Prod

```yaml
Nom  : env_hors_prod
JSON :
{
  "ANSIBLE_CONFIG": "ansible/ansible.cfg",
  "GIT_BRANCH": "main"
}
```

### Environnement Production

```yaml
Nom  : env_prod
JSON :
{
  "ANSIBLE_CONFIG": "ansible/ansible.cfg",
  "GIT_BRANCH": "main"
}
```

> Les secrets (mots de passe BDD) doivent être passés ici en JSON
> ou mieux : via Ansible Vault avec la clé de déchiffrement en variable
> d'environnement `ANSIBLE_VAULT_PASSWORD_FILE`.

---

## Étape 6 — Créer les 5 Task Templates

Menu **Task Templates** → **Add Template** :

---

### Template 1 : Synchro Hors-Prod

```yaml
Nom          : 1. Synchro Hors-Prod
Playbook     : ansible/playbooks/01_sync_hors_prod.yml
Inventory    : hosts-production
Repository   : CI-CD Ansible
Environment  : env_hors_prod
SSH Key      : ssh-key-orch-hors-prod
Description  : Synchronise le code de GitLab vers le serveur Hors-Prod

# Options avancées :
Suppress Success Alerts : Non
Allow Override of Args  : Oui  (pour passer une branche spécifique)
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

### Template 2 : Déployer Hors-Prod

```yaml
Nom          : 2. Déployer Hors-Prod
Playbook     : ansible/playbooks/02_deploy_hors_prod.yml
Inventory    : hosts-production
Repository   : CI-CD Ansible
Environment  : env_hors_prod
SSH Key      : ssh-key-orch-hors-prod
Description  : Exécute les scripts de déploiement sur Hors-Prod
```

---

### Template 3 : Synchro Prod

```yaml
Nom          : 3. Synchro Prod
Playbook     : ansible/playbooks/03_sync_prod.yml
Inventory    : hosts-production
Repository   : CI-CD Ansible
Environment  : env_prod
SSH Key      : ssh-key-orch-prod
Description  : Synchronise le code de Hors-Prod vers le serveur Prod
```

---

### Template 4 : Déployer Prod

```yaml
Nom          : 4. Déployer Prod
Playbook     : ansible/playbooks/04_deploy_prod.yml
Inventory    : hosts-production
Repository   : CI-CD Ansible
Environment  : env_prod
SSH Key      : ssh-key-orch-prod
Description  : Exécute les scripts de déploiement en Production
```

---

### Template 5 : Rollback Prod

```yaml
Nom          : 5. Rollback Prod
Playbook     : ansible/playbooks/05_rollback_prod.yml
Inventory    : hosts-production
Repository   : CI-CD Ansible
Environment  : env_prod
SSH Key      : ssh-key-orch-prod
Description  : Revient à une version précédente en Production
```

Variables obligatoires (Survey) :
```yaml
- name     : target_version
  title    : Version cible du rollback (ex: v1.4.2)
  description: Tag Git de la version vers laquelle revenir (voir git log --tags)
  type     : String
  required : Oui
```

---

## Vue finale de l'interface Semaphore

```
┌─────────────────────────────────────────────────────────────────┐
│  Ansible Semaphore — Task Templates                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  [▶ Run]  1. Synchro Hors-Prod      [Hors-Prod ← GitLab]      │
│  [▶ Run]  2. Déployer Hors-Prod     [Scripts deploy HP]        │
│  [▶ Run]  3. Synchro Prod           [Prod ← Hors-Prod]        │
│  [▶ Run]  4. Déployer Prod          [Scripts deploy Prod]      │
│  [▶ Run]  5. Rollback Prod          [⚠ Saisir version]        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Gestion des accès utilisateurs dans Semaphore

### Rôles recommandés

| Rôle Semaphore | Accès autorisé                          | Usage                    |
|----------------|-----------------------------------------|--------------------------|
| `Viewer`       | Lecture seule (logs, historique)        | Développeurs             |
| `Task Runner`  | Lancer les templates 1, 2, 3, 4         | Chefs de projet, QA      |
| `Manager`      | Lancer tous les templates (incl. 5)     | Tech Lead, DevOps        |
| `Admin`        | Configuration complète                  | Équipe Infrastructure    |

> Restreindre l'accès au template 5 (Rollback Prod) aux rôles `Manager` et `Admin`
> uniquement. Semaphore gère les permissions par template.

---

## Notifications (optionnel)

Semaphore supporte les alertes par email et webhook :

```yaml
# Dans les paramètres du projet :
Notifications:
  - Event  : Task Failed
    Channel: Email / Slack webhook
    Message: "ECHEC - {{ template_name }} sur {{ inventory }}"

  - Event  : Task Success
    Channel: Email
    Filter  : Templates contenant "Prod"
    Message: "OK - Déploiement Production réussi"
```

---

## Audit et traçabilité

Semaphore conserve un historique complet des tâches :

- **Menu History** : toutes les exécutions avec horodatage, utilisateur, statut
- **Logs complets** : sortie Ansible consultable par tâche
- Export possible vers un SIEM via webhook sur chaque événement

Pour une traçabilité renforcée, configurer le `log_path` dans `ansible.cfg`
et centraliser les logs via rsyslog ou Loki.
