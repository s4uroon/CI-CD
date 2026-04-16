# Configuration Ansible Semaphore — Zone Production

Ce guide décrit comment configurer le projet Semaphore dans la zone Prod.
Semaphore Prod est installé sur `semaphore-prod.company.com`.

> **Important** : Semaphore Prod est une instance **complètement indépendante**
> de Semaphore HP. Elle possède son propre Key Store, ses propres repositories,
> et son propre inventory. Les deux instances ne communiquent pas directement.

---

## 1. Prérequis

- Semaphore >= 2.9 installé sur `semaphore-prod`
- Ansible installé sur `semaphore-prod` (Semaphore est le control node)
- Les clés SSH E et G générées et distribuées (voir [ssh-key-management.md](../docs/ssh-key-management.md))
- Le dépôt CI-CD accessible depuis Semaphore Prod
- **Prérequis opérationnel** : Le Bouton 3 de Semaphore HP doit avoir été exécuté
  avant de lancer le Bouton 1 Prod

---

## 2. Key Store — Clés SSH

Créer les entrées dans **Key Store** :

### Clé E — Semaphore Prod → Git+Ansible Prod

| Champ        | Valeur                                          |
|--------------|-------------------------------------------------|
| Name         | `ssh-key-E-semaphore-to-git-prod`               |
| Type         | `SSH Key`                                       |
| Private Key  | Contenu de `~/.ssh/id_semaphore_prod`           |
| Passphrase   | (vide si générée sans passphrase)               |

### Clé G — Semaphore Prod → App Server(s) Prod

| Champ        | Valeur                                          |
|--------------|-------------------------------------------------|
| Name         | `ssh-key-G-semaphore-to-app-prod`               |
| Type         | `SSH Key`                                       |
| Private Key  | Contenu de `~/.ssh/id_semaphore_prod_apps`      |
| Passphrase   | (vide si générée sans passphrase)               |

---

## 3. Repositories — Sources de code

### Dépôt CI-CD (playbooks Ansible)

| Champ       | Valeur                                                           |
|-------------|------------------------------------------------------------------|
| Name        | `ci-cd-playbooks-prod`                                           |
| URL         | `git@gitlab.company.com:ops/CI-CD.git`                           |
| Branch      | `main`                                                            |
| Access Key  | Clé SSH permettant à Semaphore Prod de lire le dépôt CI-CD       |

> Si Semaphore Prod n'a pas accès direct à GitLab (zone réseau isolée),
> une option alternative est de servir le dépôt CI-CD depuis Git+Ansible Prod.
> Dans ce cas, adapter l'URL vers le bare repo interne.

---

## 4. Inventory

### Inventory Prod

| Champ       | Valeur                                      |
|-------------|---------------------------------------------|
| Name        | `inventory-prod`                            |
| Type        | `File`                                      |
| Repository  | `ci-cd-playbooks-prod`                      |
| Path        | `ansible/inventory/prod/hosts.yml`          |

---

## 5. Environment

### Env Prod par défaut

| Champ | Valeur |
|-------|--------|
| Name  | `env-prod-default` |
| JSON  | `{}` |

---

## 6. Task Templates — Les 3 boutons Prod

### Bouton 1 — Synchro Git Prod

| Champ            | Valeur                                           |
|------------------|--------------------------------------------------|
| Name             | `[PROD] 1 — Synchro Git Prod`                   |
| Playbook         | `ansible/playbooks/prod/01_sync_git_prod.yml`    |
| Inventory        | `inventory-prod`                                 |
| Repository       | `ci-cd-playbooks-prod`                           |
| Environment      | `env-prod-default`                               |
| SSH Key          | `ssh-key-E-semaphore-to-git-prod`                |
| Description      | Vérifie le flag HP, puis fetche depuis Git+Ansible HP |

**Survey Variables** :

| Variable    | Titre affiché         | Description                  | Requis |
|-------------|----------------------|------------------------------|--------|
| `repo_name` | Nom du dépôt         | Ex: myapp, api-backend       | Oui    |

**Comportement si flag absent** :

```
BLOQUE : Aucun flag de validation HP trouvé pour <repo_name>.

Actions requises sur Semaphore HP :
  1. Bouton 1 — Synchro Git HP
  2. Bouton 2 — Deployer HP
  3. Bouton 3 — Valider HP

Ensuite relancez ce bouton.
```

Le playbook s'arrête immédiatement et ne fait aucun fetch si le flag est absent.

---

### Bouton 2 — Déployer Prod

| Champ            | Valeur                                           |
|------------------|--------------------------------------------------|
| Name             | `[PROD] 2 — Déployer Prod`                      |
| Playbook         | `ansible/playbooks/prod/02_deploy_prod.yml`      |
| Inventory        | `inventory-prod`                                 |
| Repository       | `ci-cd-playbooks-prod`                           |
| Environment      | `env-prod-default`                               |
| SSH Key          | `ssh-key-E-semaphore-to-git-prod`                |
| Description      | Sauvegarde BDD + déploiement rolling sur App Prod |

> **Note** : Comme pour HP, ce template utilise 2 clés selon la cible.
> La Clé G (App Prod) est référencée via `ansible_ssh_private_key_file`
> dans `ansible/inventory/prod/group_vars/app_prod.yml`.

**Survey Variables** :

| Variable    | Titre affiché         | Description                  | Requis |
|-------------|----------------------|------------------------------|--------|
| `repo_name` | Nom du dépôt         | Ex: myapp, api-backend       | Oui    |

**Étapes du déploiement Prod (6 étapes)** :

```
Etape 0/6 : Sauvegarde BDD avant déploiement
  → pg_dump vers /opt/backups/<repo>_prod_<date>.dump
  → git tag backup/prod-pre-deploy-<date>-<heure>

Etape 1/6 : Synchronisation du code depuis Git+Ansible Prod
  → git fetch + git checkout --force origin/<branch>

Etape 2/6 : Installation des dépendances
  → composer install (PHP) ou pip install (Python)

Etape 3/6 : Migrations base de données
  → php artisan migrate --force (PHP)
  → python manage.py migrate (Python)

Etape 4/6 : Vidage et régénération du cache
  → php artisan config:cache + route:cache + view:cache (PHP)

Etape 5/6 : Redémarrage du service
  → systemctl restart <app_service>
  → Vérification : service actif après 5s

Etape 6/6 : Vérification santé
  → HTTP GET http://localhost/health → 200 OK
```

Le déploiement est **rolling** (`serial: 1`) : un serveur à la fois.
En cas d'échec sur un serveur, les serveurs suivants ne sont pas traités.

---

### Bouton 3 — Rollback Prod

| Champ            | Valeur                                              |
|------------------|-----------------------------------------------------|
| Name             | `[PROD] 3 — Rollback Prod`                          |
| Playbook         | `ansible/playbooks/prod/03_rollback_prod.yml`        |
| Inventory        | `inventory-prod`                                    |
| Repository       | `ci-cd-playbooks-prod`                              |
| Environment      | `env-prod-default`                                  |
| SSH Key          | `ssh-key-E-semaphore-to-git-prod`                   |
| Description      | Sauvegarde BDD + retour à une version précédente en Prod |

**Survey Variables** :

| Variable       | Titre affiché              | Description                                                              | Requis |
|----------------|----------------------------|--------------------------------------------------------------------------|--------|
| `repo_name`    | Nom du dépôt               | Ex: myapp, api-backend                                                   | Oui    |
| `rollback_tag` | Tag ou commit cible        | Ex: v1.3.0, backup/prod-pre-deploy-2026-04-09-14-30-00, ou hash court   | Oui    |

**Étapes du rollback Prod (5 étapes)** :

```
Etape 0/5 : Sauvegarde BDD + tag git de sécurité pré-rollback
  → pg_dump vers /opt/backups/<repo>_prod_pre-rollback_<date>.dump
  → git tag rollback/prod-pre-rollback-<date>-<heure>
  (permet d'annuler le rollback lui-même si nécessaire)

Etape 1/5 : Fetch depuis Git+Ansible Prod (tags inclus)
  → git fetch --prune --tags --force

Etape 2/5 : Checkout vers la version cible
  → git checkout --force tags/<rollback_tag>

Etape 3/5 : Réinstallation des dépendances
  → composer install (PHP) ou pip install (Python) ou npm ci (Node.js) etc.

Etape 4/5 : Rollback migrations (best-effort)
  → bash migrations/rollback.sh <rollback_tag> (si le script existe)
  → Si absent : avertissement uniquement — le dump BDD de l'Etape 0 est le filet

Etape 5/5 : Redémarrage service + health check
  → systemctl restart <app_service>
  → HTTP GET http://localhost/health → 200 OK
```

> **Important** : Le tag doit exister dans le bare Prod.
> Si le tag vient de HP (ex: `validated/hp/myapp`), lancez d'abord le **Bouton 1 Prod**
> pour le transférer depuis HP vers Prod.
>
> Pour lister les tags disponibles en Prod :
> ```bash
> git -C /opt/git/<repo_name>.git tag -l
> git -C /opt/git/<repo_name>.git tag -l 'backup/prod-pre-*'
> ```

---

## 7. Ordre d'exécution recommandé

```
Prérequis : Semaphore HP a exécuté ses 3 boutons (Synchro + Deploy + Valider)

1. [Bouton 1 Prod] Synchro Git Prod  → survey: repo_name = myapp
   → Résultat : flag HP vérifié ✓, miroir Prod à jour, commit validé affiché

2. [Bouton 2 Prod] Déployer Prod     → survey: repo_name = myapp
   → Résultat : BDD sauvegardée, déploiement rolling sur tous les App Prod
```

**En cas de problème après déploiement Prod :**
```
3. [Bouton 3 Prod] Rollback Prod     → survey: repo_name = myapp, rollback_tag = v1.3.0
   → Résultat : BDD sauvegardée, application revenue à la version v1.3.0
   → Le dump BDD pré-rollback est disponible dans /opt/backups/ si nécessaire
```

---

## 8. Variables de group_vars à adapter

### `ansible/inventory/prod/hosts.yml`
```yaml
git_ansible_prod:
  hosts:
    git-ansible-prod.company.com:    # ← adapter le hostname
      ansible_host: 10.0.2.10        # ← adapter l'IP
app_prod:
  hosts:
    app1-prod.company.com:
      ansible_host: 10.0.2.21
```

### `ansible/inventory/prod/group_vars/git_ansible_prod.yml`
```yaml
git_hp_ssh_host: git-ansible-hp.company.com  # ← hostname Git+Ansible HP
git_bare_base: /opt/git                       # ← répertoire des bare repos
git_ssh_key_path: /opt/keys/id_gitprod_to_githp # ← Clé F (sur git-ansible-prod)
```

### `ansible/inventory/prod/group_vars/app_prod.yml`
```yaml
git_server_host: git-ansible-prod.company.com   # ← hostname Git+Ansible Prod
git_ssh_key_path: /opt/keys/id_appprod_to_gitprod # ← Clé H (sur les app Prod)
app_user: deploy
app_group: deploy
db_backup_dir: /opt/backups                      # ← répertoire des dumps BDD
```

---

## 9. Sécurité spécifique Prod

### Contrôle d'accès Semaphore

Configurer dans Semaphore Prod un **Team** dédié avec des rôles restreints :

- `Task Runner` : peut uniquement lancer les templates (pas modifier)
- `Manager` : peut modifier les templates et la configuration

Limiter les accès aux boutons Prod aux seules personnes autorisées.

### Confirmation avant déploiement

Pour ajouter une confirmation manuelle avant le Bouton 2 Prod, utiliser la
fonctionnalité **Confirm** dans les Task Templates Semaphore (si disponible
dans votre version), ou documenter le processus d'approbation dans votre
workflow (ticket ITSM, approval Slack, etc.).

### Audit trail

Semaphore enregistre chaque exécution avec :
- L'utilisateur qui a lancé la tâche
- L'heure exacte
- Les survey variables utilisées
- L'output complet du playbook

Ces logs sont accessibles dans **Task History** pour chaque template.

---

## 10. Gestion de plusieurs dépôts

Pour déployer plusieurs applications, créer autant de fichiers de config que
d'applications dans `ansible/repos/` :

```
ansible/repos/
├── myapp.yml
├── api-backend.yml
├── frontend.yml
└── worker-service.yml
```

Les 2 mêmes boutons Semaphore servent pour tous les dépôts — il suffit de
changer la Survey Variable `repo_name` à chaque exécution.

Chaque dépôt peut avoir sa propre liste de serveurs cibles :
```yaml
# ansible/repos/api-backend.yml
app_servers_prod:
  - api1-prod.company.com
  - api2-prod.company.com
  - api3-prod.company.com   # 3 serveurs pour l'API
```

---

## 11. Rollback rapide

En cas de problème après un déploiement Prod, utiliser le **Bouton 3 — Rollback Prod**.

**Étapes** :

1. Identifier le tag à cibler :
   ```bash
   # Tags de backup créés automatiquement avant chaque déploiement
   git -C /opt/git/myapp.git tag -l 'backup/prod-pre-deploy-*'
   # → backup/prod-pre-deploy-2026-04-09-14-30-00
   
   # Tags sémantiques de version
   git -C /opt/git/myapp.git tag -l 'v*'
   # → v1.3.0, v1.2.1, v1.2.0
   ```

2. Lancer le **Bouton 3 Prod** avec :
   - `repo_name` = `myapp`
   - `rollback_tag` = `backup/prod-pre-deploy-2026-04-09-14-30-00`

3. Le playbook effectue automatiquement :
   - Une sauvegarde BDD pré-rollback (filet de sécurité)
   - Le checkout vers la version demandée
   - La réinstallation des dépendances et le restart du service

4. Si la BDD doit également être restaurée (données altérées) :
   ```bash
   pg_restore -Fc -d myapp_prod /opt/backups/myapp_prod_pre-rollback_<date>.dump
   ```

Voir [docs/rollback-strategy.md](../docs/rollback-strategy.md) pour le guide complet.
