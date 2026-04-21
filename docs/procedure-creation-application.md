# Procédure de création d'une nouvelle application : [NOM_APP]

> Remplacer `[NOM_APP]` par le nom réel de l'application (ex: `myapp`, `api-backend`).
> Suivre les étapes chronologiquement. Cocher chaque action au fur et à mesure.

---

## Pré-requis

- [ ] Accès administrateur à GitLab Central (`gitlab.company.com`)
- [ ] Accès SSH aux serveurs Git+Ansible HP (`git-ansible-hp.company.com`)
- [ ] Accès SSH aux serveurs Git+Ansible PROD (`git-ansible-prod.company.com`)
- [ ] Accès à Semaphore HP (`https://semaphore-hp.company.com`)
- [ ] Accès à Semaphore PROD (`https://semaphore-prod.company.com`)
- [ ] Droits sudo sur les App Servers HP (`app[n]-hp.company.com`)
- [ ] Droits sudo sur les App Servers PROD (`app[n]-prod.company.com`)
- [ ] Définir la stack applicative de `[NOM_APP]` (PHP / Python / Node.js / Java / Go)

---

## 1. GitLab — Création du dépôt principal

- [ ] Se connecter à GitLab Central (`https://gitlab.company.com`)
- [ ] Créer un nouveau projet : `[NOM_APP]`
- [ ] Définir la visibilité (interne ou privée selon politique de l'organisation)
- [ ] Initialiser avec un `README.md` minimal
- [ ] Ajouter les membres de l'équipe avec les droits appropriés (Maintainer / Developer)
- [ ] Créer la branche `main` (ou `master`) comme branche protégée
- [ ] Configurer les Deploy Keys nécessaires :
  - [ ] Ajouter la clé publique **B** (depuis Git+Ansible HP) en tant que Deploy Key (lecture seule)
  - [ ] Ajouter la clé publique **F** (depuis Git+Ansible PROD) en tant que Deploy Key (lecture seule) — *optionnel si PROD tire depuis HP*
- [ ] Récupérer et noter l'URL SSH du dépôt : `git@gitlab.company.com:<groupe>/[NOM_APP].git`
- [ ] Créer le fichier de configuration dans le dépôt CI-CD : `ansible/repos/[NOM_APP].yml`

  ```yaml
  # ansible/repos/[NOM_APP].yml
  repo_name: [NOM_APP]
  git_remote_url: "git@gitlab.company.com:<groupe>/[NOM_APP].git"
  git_branch: main
  app_type: php          # "php", "python", "nodejs", "java", "go" ou "static"

  app_root: "/opt/apps/[NOM_APP]"
  app_service_name: [NOM_APP]
  app_user: deploy
  app_group: deploy

  app_servers_hp:
    - app1-hp.company.com

  app_servers_prod:
    - app1-prod.company.com

  run_migrations: false
  # db_host_prod: prod-db.company.com
  # db_name_prod: [NOM_APP]_prod
  # db_user_prod: app_user
  ```

---

## 2. Environnement HP (Hors-Prod)

### 2.1 Création du dépôt Git HP

- [ ] Se connecter en SSH au serveur **Git+Ansible HP**
- [ ] Initialiser le dépôt bare HP :
  ```bash
  git init --bare /opt/git/[NOM_APP].git
  chown -R deploy:deploy /opt/git/[NOM_APP].git
  ```
- [ ] Configurer le remote GitLab dans le bare repo :
  ```bash
  git -C /opt/git/[NOM_APP].git remote add origin git@gitlab.company.com:<groupe>/[NOM_APP].git
  ```
- [ ] Vérifier la connexion GitLab (clé B) :
  ```bash
  GIT_SSH_COMMAND="ssh -i /opt/keys/id_githp_gitlab" \
    git -C /opt/git/[NOM_APP].git ls-remote origin
  ```
- [ ] Ajouter `[NOM_APP]` dans le script wrapper de la clé D (`/home/deploy/bin/git-read-wrapper.sh`) :
  ```bash
  ALLOWED=(
    "/opt/git/myapp.git"
    "/opt/git/[NOM_APP].git"   # ← ajouter cette ligne
  )
  ```

### 2.2 Préparation des App Servers HP

- [ ] Se connecter à chaque **App Server HP** (`app[n]-hp.company.com`)
- [ ] Créer le répertoire de déploiement :
  ```bash
  mkdir -p /opt/apps/[NOM_APP]
  chown deploy:deploy /opt/apps/[NOM_APP]
  ```
- [ ] Vérifier les pré-requis système (runtime selon la stack de `[NOM_APP]`) :
  - [ ] PHP : `php --version` + Composer installé
  - [ ] Python : `python3 --version` + pip/venv installés
  - [ ] Node.js : `node --version` + npm installé
  - [ ] Java : `java -version` + Maven/Gradle installé
  - [ ] Go : `go version`
- [ ] Initialiser le working tree HP depuis le bare repo :
  ```bash
  # Sur Git+Ansible HP
  git -C /opt/git/[NOM_APP].git worktree add /opt/apps/[NOM_APP] main
  # OU depuis App HP (après un premier git fetch)
  git clone --no-checkout /opt/git/[NOM_APP].git /opt/apps/[NOM_APP]
  ```
- [ ] Tester la connectivité SSH Ansible depuis Semaphore HP (clé C) :
  ```bash
  ssh -i ~/.ssh/id_semaphore_hp_apps deploy@app1-hp.company.com "echo 'Clé C OK'"
  ```
- [ ] Tester le git fetch depuis l'App HP (clé D) :
  ```bash
  GIT_SSH_COMMAND="ssh -i /opt/keys/id_apphp_to_githp" \
    git ls-remote deploy@git-ansible-hp:/opt/git/[NOM_APP].git
  ```

### 2.3 Configuration de Semaphore HP

- [ ] Se connecter à l'interface web **Semaphore HP** (`https://semaphore-hp.company.com`)
- [ ] Vérifier que les clés SSH A et C sont bien configurées dans le Key Store Semaphore HP
- [ ] Vérifier que le Repository CI-CD est bien configuré dans Semaphore HP
- [ ] Ajouter `[NOM_APP]` dans les templates existants si Survey Variable `repo_name` est utilisé :
  - [ ] Template "Synchro Git HP" — vérifier que `repo_name` accepte `[NOM_APP]`
  - [ ] Template "Déployer HP" — vérifier que `repo_name` accepte `[NOM_APP]`
  - [ ] Template "Valider HP" — vérifier que `repo_name` accepte `[NOM_APP]`
  - [ ] Template "Rollback HP" — vérifier que `repo_name` accepte `[NOM_APP]`
- [ ] Tester le **Bouton 1 HP** (Synchro Git HP) avec `repo_name = [NOM_APP]` :
  - [ ] Vérification que le miroir bare HP est mis à jour ✓
- [ ] Tester le **Bouton 2 HP** (Déployer HP) avec `repo_name = [NOM_APP]` :
  - [ ] Vérification que l'application se déploie sur App HP ✓
- [ ] Effectuer les tests QA sur HP
- [ ] Tester le **Bouton 3 HP** (Valider HP) avec `repo_name = [NOM_APP]` :
  - [ ] Vérification que le tag `validated/hp/[NOM_APP]` est créé sur bare HP ✓
  ```bash
  git -C /opt/git/[NOM_APP].git tag -l 'validated/hp/[NOM_APP]'
  ```

---

## 3. Environnement PROD (Production)

### 3.1 Création du dépôt Git PROD

- [ ] Se connecter en SSH au serveur **Git+Ansible PROD**
- [ ] Initialiser le dépôt bare PROD :
  ```bash
  git init --bare /opt/git/[NOM_APP].git
  chown -R deploy:deploy /opt/git/[NOM_APP].git
  ```
- [ ] Configurer le remote pointant vers Git+Ansible HP (pull-only depuis HP) :
  ```bash
  git -C /opt/git/[NOM_APP].git remote add hp git@git-ansible-hp.company.com:/opt/git/[NOM_APP].git
  ```
- [ ] Tester la connexion HP (clé F) — vérifier le flag et le fetch :
  ```bash
  ssh -i /opt/keys/id_gitprod_to_githp deploy@git-ansible-hp \
    "git -C /opt/git/[NOM_APP].git tag -l 'validated/hp/[NOM_APP]'"
  ```
- [ ] Ajouter `[NOM_APP]` dans le script wrapper de la clé H (`/home/deploy/bin/git-read-wrapper.sh`) :
  ```bash
  ALLOWED=(
    "/opt/git/myapp.git"
    "/opt/git/[NOM_APP].git"   # ← ajouter cette ligne
  )
  ```

### 3.2 Préparation des App Servers PROD

- [ ] Se connecter à chaque **App Server PROD** (`app[n]-prod.company.com`)
- [ ] Créer le répertoire de déploiement :
  ```bash
  mkdir -p /opt/apps/[NOM_APP]
  chown deploy:deploy /opt/apps/[NOM_APP]
  ```
- [ ] Vérifier les pré-requis système (même stack que HP) :
  - [ ] Runtime applicatif installé et à la même version que HP
  - [ ] `pg_dump` disponible si `run_migrations: true` (sauvegarde BDD avant déploiement)
- [ ] Créer le répertoire de sauvegardes BDD :
  ```bash
  mkdir -p /opt/backups
  chown deploy:deploy /opt/backups
  ```
- [ ] Initialiser le working tree PROD depuis le bare repo PROD :
  ```bash
  # Sur Git+Ansible PROD (après un premier sync)
  git -C /opt/git/[NOM_APP].git worktree add /opt/apps/[NOM_APP] main
  ```
- [ ] Tester la connectivité SSH Ansible depuis Semaphore PROD (clé G) :
  ```bash
  ssh -i ~/.ssh/id_semaphore_prod_apps deploy@app1-prod.company.com "echo 'Clé G OK'"
  ```
- [ ] Tester le git fetch depuis l'App PROD (clé H) :
  ```bash
  GIT_SSH_COMMAND="ssh -i /opt/keys/id_appprod_to_gitprod" \
    git ls-remote deploy@git-ansible-prod:/opt/git/[NOM_APP].git
  ```

### 3.3 Configuration de Semaphore PROD

- [ ] Se connecter à l'interface web **Semaphore PROD** (`https://semaphore-prod.company.com`)
- [ ] Vérifier que les clés SSH E et G sont bien configurées dans le Key Store Semaphore PROD
- [ ] Vérifier que le Repository CI-CD est bien configuré dans Semaphore PROD
- [ ] Vérifier que les templates existants acceptent `[NOM_APP]` en Survey Variable :
  - [ ] Template "Synchro Git Prod"
  - [ ] Template "Déployer Prod"
  - [ ] Template "Rollback PROD"
- [ ] Tester le **Bouton 1 PROD** (Synchro Git Prod) avec `repo_name = [NOM_APP]` :
  - [ ] Vérification que le flag `validated/hp/[NOM_APP]` est bien détecté ✓
  - [ ] Vérification que le miroir bare PROD est mis à jour ✓
- [ ] Tester le **Bouton 2 PROD** (Déployer Prod) avec `repo_name = [NOM_APP]` :
  - [ ] Vérification que l'application se déploie sur App PROD ✓
  - [ ] Vérification du health check HTTP ✓

---

## 4. Validation finale

- [ ] Vérifier que le déploiement HP est fonctionnel (application accessible sur HP)
- [ ] Vérifier que le déploiement PROD est fonctionnel (application accessible en PROD)
- [ ] Vérifier que le rollback HP fonctionne (Bouton 4 HP, optionnel)
- [ ] Vérifier que le rollback PROD fonctionne (Bouton 3 PROD, optionnel)
- [ ] Documenter les spécificités de `[NOM_APP]` dans le README de son dépôt GitLab :
  - [ ] Stack technique et version du runtime
  - [ ] Variables d'environnement spécifiques
  - [ ] Ports applicatifs
  - [ ] Prérequis BDD si applicable
- [ ] Informer l'équipe de la disponibilité de la nouvelle application en HP et PROD

---

## Récapitulatif des ressources créées

| Ressource                          | Valeur                                           |
|------------------------------------|--------------------------------------------------|
| Dépôt GitLab                       | `git@gitlab.company.com:<groupe>/[NOM_APP].git`  |
| Bare repo HP                       | `git-ansible-hp:/opt/git/[NOM_APP].git`          |
| Working tree HP                    | `app[n]-hp:/opt/apps/[NOM_APP]`                  |
| Flag validation HP                 | `validated/hp/[NOM_APP]`                         |
| Bare repo PROD                     | `git-ansible-prod:/opt/git/[NOM_APP].git`        |
| Working tree PROD                  | `app[n]-prod:/opt/apps/[NOM_APP]`                |
| Config Ansible                     | `ansible/repos/[NOM_APP].yml`                    |
