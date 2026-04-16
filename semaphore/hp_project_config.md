# Configuration Ansible Semaphore — Zone HP

Ce guide décrit comment configurer le projet Semaphore dans la zone HP.
Semaphore HP est installé sur `semaphore-hp.company.com`.

---

## 1. Prérequis

- Semaphore >= 2.9 installé sur `semaphore-hp`
- Ansible installé sur `semaphore-hp` (Semaphore est le control node)
- Les clés SSH A et C générées et distribuées (voir [ssh-key-management.md](../docs/ssh-key-management.md))
- Le dépôt CI-CD cloné ou accessible via GitLab

---

## 2. Key Store — Clés SSH

Créer les entrées dans **Key Store** (menu latéral → Key Store → +) :

### Clé A — Semaphore HP → Git+Ansible HP

| Champ        | Valeur                                        |
|--------------|-----------------------------------------------|
| Name         | `ssh-key-A-semaphore-to-git-hp`               |
| Type         | `SSH Key`                                     |
| Private Key  | Contenu de `~/.ssh/id_semaphore_hp`           |
| Passphrase   | (vide si générée sans passphrase)             |

### Clé C — Semaphore HP → App Server(s) HP

| Champ        | Valeur                                        |
|--------------|-----------------------------------------------|
| Name         | `ssh-key-C-semaphore-to-app-hp`               |
| Type         | `SSH Key`                                     |
| Private Key  | Contenu de `~/.ssh/id_semaphore_hp_apps`      |
| Passphrase   | (vide si générée sans passphrase)             |

---

## 3. Repositories — Sources de code

Créer l'entrée dans **Repositories** (menu → Repositories → +) :

### Dépôt CI-CD (playbooks Ansible)

| Champ       | Valeur                                                          |
|-------------|------------------------------------------------------------------|
| Name        | `ci-cd-playbooks-hp`                                            |
| URL         | `git@gitlab.company.com:ops/CI-CD.git`                          |
| Branch      | `main`                                                           |
| Access Key  | Clé SSH permettant à Semaphore HP de lire le dépôt CI-CD        |

> Si Semaphore HP est sur le même réseau que GitLab, créer une clé dédiée
> (Deploy Key sur le projet CI-CD dans GitLab) et l'enregistrer en Key Store.

---

## 4. Inventory

Créer l'entrée dans **Inventory** :

### Inventory HP

| Champ       | Valeur                                     |
|-------------|---------------------------------------------|
| Name        | `inventory-hp`                              |
| Type        | `File`                                      |
| Repository  | `ci-cd-playbooks-hp`                        |
| Path        | `ansible/inventory/hp/hosts.yml`            |
| SSH Key     | (non utilisé au niveau inventory)           |

---

## 5. Environment

Créer l'entrée dans **Environment** :

### Env HP par défaut

| Champ | Valeur |
|-------|--------|
| Name  | `env-hp-default` |
| JSON  | `{}` |

---

## 6. Task Templates — Les 4 boutons HP

### Bouton 1 — Synchro Git HP

| Champ            | Valeur                                      |
|------------------|---------------------------------------------|
| Name             | `[HP] 1 — Synchro Git HP`                  |
| Playbook         | `ansible/playbooks/hp/01_sync_git_hp.yml`   |
| Inventory        | `inventory-hp`                              |
| Repository       | `ci-cd-playbooks-hp`                        |
| Environment      | `env-hp-default`                            |
| SSH Key          | `ssh-key-A-semaphore-to-git-hp`             |
| Description      | Fetche le dépôt depuis GitLab vers le miroir bare HP |

**Survey Variables** :

| Variable    | Titre affiché         | Description                  | Requis |
|-------------|----------------------|------------------------------|--------|
| `repo_name` | Nom du dépôt         | Ex: myapp, api-backend       | Oui    |

---

### Bouton 2 — Déployer HP

| Champ            | Valeur                                      |
|------------------|---------------------------------------------|
| Name             | `[HP] 2 — Déployer HP`                     |
| Playbook         | `ansible/playbooks/hp/02_deploy_hp.yml`     |
| Inventory        | `inventory-hp`                              |
| Repository       | `ci-cd-playbooks-hp`                        |
| Environment      | `env-hp-default`                            |
| SSH Key          | `ssh-key-A-semaphore-to-git-hp`             |
| Description      | Déploie l'application sur les serveurs HP   |

> **Note** : Ce template utilise 2 clés SSH différentes selon la cible :
> - Clé A pour se connecter à Git+Ansible HP (Play 1)
> - Clé C pour se connecter aux App HP (Play 2)
>
> Ansible sélectionne la bonne clé via `ansible_ssh_private_key_file` dans
> l'inventory (`group_vars/git_ansible_hp.yml` et `group_vars/app_hp.yml`).
> La clé déclarée dans le template est utilisée par défaut pour les hôtes
> sans clé explicite dans l'inventory.

**Survey Variables** :

| Variable    | Titre affiché         | Description                  | Requis |
|-------------|----------------------|------------------------------|--------|
| `repo_name` | Nom du dépôt         | Ex: myapp, api-backend       | Oui    |

---

### Bouton 3 — Valider HP

| Champ            | Valeur                                          |
|------------------|--------------------------------------------------|
| Name             | `[HP] 3 — Valider HP (autoriser déploiement Prod)` |
| Playbook         | `ansible/playbooks/hp/03_validate_hp.yml`       |
| Inventory        | `inventory-hp`                                  |
| Repository       | `ci-cd-playbooks-hp`                            |
| Environment      | `env-hp-default`                                |
| SSH Key          | `ssh-key-A-semaphore-to-git-hp`                 |
| Description      | Pose le tag de validation HP — débloque Semaphore Prod |

**Survey Variables** :

| Variable    | Titre affiché         | Description                  | Requis |
|-------------|----------------------|------------------------------|--------|
| `repo_name` | Nom du dépôt         | Ex: myapp, api-backend       | Oui    |

> **Attention** : Ce bouton pose le tag `validated/hp/<repo_name>` sur le
> commit actuellement dans le bare HP. Il doit être exécuté **après** avoir
> effectué les tests sur HP, pas avant.

---

### Bouton 4 — Rollback HP

| Champ            | Valeur                                          |
|------------------|--------------------------------------------------|
| Name             | `[HP] 4 — Rollback HP`                          |
| Playbook         | `ansible/playbooks/hp/04_rollback_hp.yml`        |
| Inventory        | `inventory-hp`                                   |
| Repository       | `ci-cd-playbooks-hp`                             |
| Environment      | `env-hp-default`                                 |
| SSH Key          | `ssh-key-A-semaphore-to-git-hp`                  |
| Description      | Retourne l'application HP à une version précédente |

**Survey Variables** :

| Variable       | Titre affiché              | Description                                                        | Requis |
|----------------|----------------------------|--------------------------------------------------------------------|--------|
| `repo_name`    | Nom du dépôt               | Ex: myapp, api-backend                                             | Oui    |
| `rollback_tag` | Tag ou commit cible        | Ex: v1.3.0, backup/prod-pre-deploy-2026-04-09-14-30-00, ou hash court | Oui    |

> **Note** : Le tag doit exister dans le miroir bare HP.
> Pour lister les tags disponibles, exécutez sur `git-ansible-hp` :
> ```bash
> git -C /opt/git/<repo_name>.git tag -l
> git -C /opt/git/<repo_name>.git tag -l 'backup/*'
> ```
>
> **Attention** : Après un rollback HP, si vous souhaitez déployer la version
> rollbackée en Production, vous devez re-exécuter le **Bouton 3 — Valider HP**
> pour recréer le flag de validation sur la version cible.

---

## 7. Ordre d'exécution recommandé

```
1. [Bouton 1] Synchro Git HP     → survey: repo_name = myapp
   → Résultat : miroir HP à jour, commit affiché

2. [Bouton 2] Déployer HP        → survey: repo_name = myapp
   → Résultat : application déployée sur App HP, service actif

   ... Tests QA sur HP ...

3. [Bouton 3] Valider HP         → survey: repo_name = myapp
   → Résultat : tag validated/hp/myapp posé
   → Semaphore Prod peut maintenant démarrer
```

**En cas de problème après déploiement HP :**
```
4. [Bouton 4] Rollback HP        → survey: repo_name = myapp, rollback_tag = v1.2.0
   → Résultat : application revenue à la version v1.2.0 sur App HP
   → Si déploiement Prod souhaité : re-exécuter Bouton 3 sur la version rollbackée
```

---

## 8. Configuration SSH dans Ansible (group_vars)

Les fichiers `ansible/inventory/hp/group_vars/` définissent :

- `git_ansible_hp.yml` : `ansible_ssh_private_key_file` = chemin vers **Clé A**
- `app_hp.yml` : `ansible_ssh_private_key_file` = chemin vers **Clé C**

Ces chemins doivent correspondre aux clés stockées sur le serveur Semaphore HP
et référencées dans le Key Store Semaphore.

Dans Semaphore, il est possible d'utiliser le **Vault** pour stocker les clés
privées de manière chiffrée, ou de les déployer directement sur le système de
fichiers de Semaphore HP (chemin référencé dans `group_vars`).

---

## 9. Notifications (optionnel)

Configurer des alertes dans **Notifications** (menu → Notifications) :
- Slack / Teams : notifier l'équipe lors d'un déploiement HP ou d'une validation
- Email : alerter si un playbook échoue

---

## 10. Variables de group_vars à adapter

Éditer les fichiers suivants pour correspondre à votre infrastructure :

### `ansible/inventory/hp/hosts.yml`
```yaml
git_ansible_hp:
  hosts:
    git-ansible-hp.company.com:      # ← adapter le hostname
      ansible_host: 10.0.1.10        # ← adapter l'IP
```

### `ansible/inventory/hp/group_vars/git_ansible_hp.yml`
```yaml
gitlab_ssh_host: gitlab.company.com  # ← hostname GitLab
git_bare_base: /opt/git              # ← répertoire des bare repos
git_ssh_key_path: /opt/keys/id_githp_gitlab  # ← Clé B (sur git-ansible-hp)
```

### `ansible/inventory/hp/group_vars/app_hp.yml`
```yaml
git_server_host: git-ansible-hp.company.com  # ← hostname Git+Ansible HP
git_ssh_key_path: /opt/keys/id_apphp_to_githp # ← Clé D (sur les app HP)
app_user: deploy                              # ← utilisateur applicatif
app_group: deploy
```
