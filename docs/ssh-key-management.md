# Gestion des Clés SSH

## Vue d'ensemble des 8 paires de clés

L'architecture à 6 serveurs nécessite 8 paires de clés SSH distinctes,
réparties en deux catégories :

**Clés Ansible (A, B, C, D)** — l'Orchestrateur pilote les serveurs cibles.
**Clés Git (E, F, G, H)** — les serveurs tirent le code depuis leur source.

| Clé | Source         | Destination    | Usage                                    | Fichier clé privée (sur la source)      |
|-----|----------------|----------------|------------------------------------------|-----------------------------------------|
| A   | Orchestrateur  | Git HP         | Ansible pilote la synchro bare           | `~/.ssh/id_orch_git_hp`                |
| B   | Orchestrateur  | App HP         | Ansible pilote le déploiement HP         | `~/.ssh/id_orch_app_hp`                |
| C   | Orchestrateur  | Git Prod       | Ansible pilote la synchro bare Prod      | `~/.ssh/id_orch_git_prod`              |
| D   | Orchestrateur  | App Prod       | Ansible pilote le déploiement Prod       | `~/.ssh/id_orch_app_prod`              |
| E   | Git HP         | GitLab         | git fetch bare HP <- GitLab              | `/home/deploy/.ssh/id_githp_gitlab`    |
| F   | Git Prod       | Git HP         | git fetch bare Prod <- Git HP            | `/home/deploy/.ssh/id_gitprod_githp`   |
| G   | App HP         | Git HP         | git fetch working tree App HP <- Git HP  | `/home/deploy/.ssh/id_apphp_githp`     |
| H   | App Prod       | Git Prod       | git fetch working tree App Prod <- Git Prod | `/home/deploy/.ssh/id_appprod_gitprod` |

---

## Génération des clés

### Sur l'Orchestrateur (Semaphore)

```bash
su - semaphore   # ou l'utilisateur qui lance Ansible

# Clé A : Orchestrateur -> Git HP (Ansible)
ssh-keygen -t ed25519 -C "orch-to-git-hp" -f ~/.ssh/id_orch_git_hp -N ""

# Clé B : Orchestrateur -> App HP (Ansible)
ssh-keygen -t ed25519 -C "orch-to-app-hp" -f ~/.ssh/id_orch_app_hp -N ""

# Clé C : Orchestrateur -> Git Prod (Ansible)
ssh-keygen -t ed25519 -C "orch-to-git-prod" -f ~/.ssh/id_orch_git_prod -N ""

# Clé D : Orchestrateur -> App Prod (Ansible)
ssh-keygen -t ed25519 -C "orch-to-app-prod" -f ~/.ssh/id_orch_app_prod -N ""
```

### Sur Git HP

```bash
su - deploy

# Clé E : Git HP -> GitLab (git fetch bare)
ssh-keygen -t ed25519 -C "git-hp-to-gitlab" -f ~/.ssh/id_githp_gitlab -N ""
```

### Sur Git Prod

```bash
su - deploy

# Clé F : Git Prod -> Git HP (git fetch bare)
ssh-keygen -t ed25519 -C "git-prod-to-git-hp" -f ~/.ssh/id_gitprod_githp -N ""
```

### Sur App HP

```bash
su - deploy

# Clé G : App HP -> Git HP (git fetch working tree)
ssh-keygen -t ed25519 -C "app-hp-to-git-hp" -f ~/.ssh/id_apphp_githp -N ""
```

### Sur App Prod

```bash
su - deploy

# Clé H : App Prod -> Git Prod (git fetch working tree)
ssh-keygen -t ed25519 -C "app-prod-to-git-prod" -f ~/.ssh/id_appprod_gitprod -N ""
```

---

## Distribution des clés publiques

### Sur Git HP — autorise : Clé A (Ansible), Clé F (git Prod), Clé G (git App HP)

```bash
# Sur Git HP, en tant que deploy
cat ~/.ssh/authorized_keys   # vérifier le fichier existant

# Clé A : accès Ansible complet depuis l'Orchestrateur
echo "# Clé A — Ansible Orchestrateur" >> ~/.ssh/authorized_keys
cat /tmp/id_orch_git_hp.pub  >> ~/.ssh/authorized_keys

# Clé F : Git Prod peut lire le dépôt bare (git-upload-pack uniquement)
echo 'command="git-upload-pack /opt/git/myapp.git",no-port-forwarding,no-X11-forwarding,no-agent-forwarding,no-pty <contenu de id_gitprod_githp.pub>' >> ~/.ssh/authorized_keys

# Clé G : App HP peut lire le dépôt bare (git-upload-pack uniquement)
echo 'command="git-upload-pack /opt/git/myapp.git",no-port-forwarding,no-X11-forwarding,no-agent-forwarding,no-pty <contenu de id_apphp_githp.pub>' >> ~/.ssh/authorized_keys
```

> **Important** : Les clés F et G sont restreintes avec `command="git-upload-pack ..."`.
> Même si elles sont compromises, elles ne permettent pas d'ouvrir un shell ni d'écrire dans le dépôt.

**Script de wrapper multi-dépôts** (si vous avez plusieurs applications) :

```bash
# /home/deploy/bin/git-read-wrapper.sh
#!/bin/bash
ALLOWED=(
  "/opt/git/myapp.git"
  "/opt/git/myapp2.git"
)
if [[ "$SSH_ORIGINAL_COMMAND" =~ ^git-upload-pack\ \'(.+)\'$ ]]; then
  REPO="${BASH_REMATCH[1]}"
  for allowed in "${ALLOWED[@]}"; do
    [[ "$REPO" == "$allowed" ]] && exec git-upload-pack "$REPO"
  done
fi
echo "Accès refusé" >&2; exit 1
```

```bash
# Dans authorized_keys, remplacer git-upload-pack par le wrapper :
command="/home/deploy/bin/git-read-wrapper.sh",no-port-forwarding,...
```

### Sur App HP — autorise : Clé B (Ansible)

```bash
# Sur App HP, en tant que deploy
echo "# Clé B — Ansible Orchestrateur" >> ~/.ssh/authorized_keys
cat /tmp/id_orch_app_hp.pub >> ~/.ssh/authorized_keys
```

### Sur Git Prod — autorise : Clé C (Ansible), Clé H (git App Prod)

```bash
# Sur Git Prod, en tant que deploy

# Clé C : accès Ansible complet depuis l'Orchestrateur
echo "# Clé C — Ansible Orchestrateur" >> ~/.ssh/authorized_keys
cat /tmp/id_orch_git_prod.pub >> ~/.ssh/authorized_keys

# Clé H : App Prod peut lire le dépôt bare (git-upload-pack uniquement)
echo 'command="git-upload-pack /opt/git/myapp.git",no-port-forwarding,no-X11-forwarding,no-agent-forwarding,no-pty <contenu de id_appprod_gitprod.pub>' >> ~/.ssh/authorized_keys
```

### Sur App Prod — autorise : Clé D (Ansible)

```bash
# Sur App Prod, en tant que deploy
echo "# Clé D — Ansible Orchestrateur" >> ~/.ssh/authorized_keys
cat /tmp/id_orch_app_prod.pub >> ~/.ssh/authorized_keys
```

### Sur GitLab — autorise : Clé E (Git HP)

1. Connectez-vous sur GitLab avec le compte `ci-reader` (lecture seule)
2. **Profile → SSH Keys → Add new key**
3. Coller le contenu de `id_githp_gitlab.pub`
4. Titre : `Git HP deploy key`

Ou via une **Deploy Key** sur le projet (plus restrictif, recommandé) :
- **Projet → Settings → Repository → Deploy Keys**
- Ajouter la clé E avec accès **lecture seule**

---

## Fichiers `~/.ssh/config` sur chaque serveur

### Orchestrateur

```ssh-config
# Git HP — Ansible (Clé A)
Host git-hp
    HostName git-hp.company.com
    User deploy
    IdentityFile ~/.ssh/id_orch_git_hp
    IdentitiesOnly yes
    StrictHostKeyChecking accept-new

# App HP — Ansible (Clé B)
Host app-hp
    HostName app-hp.company.com
    User deploy
    IdentityFile ~/.ssh/id_orch_app_hp
    IdentitiesOnly yes
    StrictHostKeyChecking accept-new

# Git Prod — Ansible (Clé C)
Host git-prod
    HostName git-prod.company.com
    User deploy
    IdentityFile ~/.ssh/id_orch_git_prod
    IdentitiesOnly yes
    StrictHostKeyChecking accept-new

# App Prod — Ansible (Clé D)
Host app-prod
    HostName app-prod.company.com
    User deploy
    IdentityFile ~/.ssh/id_orch_app_prod
    IdentitiesOnly yes
    StrictHostKeyChecking accept-new
```

### Git HP

```ssh-config
# GitLab — git fetch bare (Clé E)
Host gitlab
    HostName gitlab.company.com
    User git
    IdentityFile ~/.ssh/id_githp_gitlab
    IdentitiesOnly yes
    StrictHostKeyChecking accept-new
    ForwardAgent no
```

### Git Prod

```ssh-config
# Git HP — git fetch bare (Clé F)
Host git-hp
    HostName git-hp.company.com
    User deploy
    IdentityFile ~/.ssh/id_gitprod_githp
    IdentitiesOnly yes
    StrictHostKeyChecking accept-new
    ForwardAgent no
```

### App HP

```ssh-config
# Git HP — git fetch working tree (Clé G)
Host git-hp
    HostName git-hp.company.com
    User deploy
    IdentityFile ~/.ssh/id_apphp_githp
    IdentitiesOnly yes
    StrictHostKeyChecking accept-new
    ForwardAgent no
```

### App Prod

```ssh-config
# Git Prod — git fetch working tree (Clé H)
Host git-prod
    HostName git-prod.company.com
    User deploy
    IdentityFile ~/.ssh/id_appprod_gitprod
    IdentitiesOnly yes
    StrictHostKeyChecking accept-new
    ForwardAgent no
```

---

## Enregistrement des known_hosts

Pré-enregistrer les fingerprints pour éviter les prompts interactifs :

```bash
# Sur l'Orchestrateur
ssh-keyscan -H git-hp.company.com  >> ~/.ssh/known_hosts
ssh-keyscan -H app-hp.company.com  >> ~/.ssh/known_hosts
ssh-keyscan -H git-prod.company.com >> ~/.ssh/known_hosts
ssh-keyscan -H app-prod.company.com >> ~/.ssh/known_hosts

# Sur Git HP
ssh-keyscan -H gitlab.company.com  >> ~/.ssh/known_hosts

# Sur Git Prod
ssh-keyscan -H git-hp.company.com  >> ~/.ssh/known_hosts

# Sur App HP
ssh-keyscan -H git-hp.company.com  >> ~/.ssh/known_hosts

# Sur App Prod
ssh-keyscan -H git-prod.company.com >> ~/.ssh/known_hosts
```

---

## Tests de connectivité

```bash
# Depuis l'Orchestrateur
ssh -i ~/.ssh/id_orch_git_hp  deploy@git-hp   "echo 'Clé A OK'"
ssh -i ~/.ssh/id_orch_app_hp  deploy@app-hp   "echo 'Clé B OK'"
ssh -i ~/.ssh/id_orch_git_prod deploy@git-prod "echo 'Clé C OK'"
ssh -i ~/.ssh/id_orch_app_prod deploy@app-prod "echo 'Clé D OK'"

# Depuis Git HP (Clé E)
ssh -T git@gitlab.company.com
# Réponse : "Welcome to GitLab, @ci-reader!"

# Depuis Git Prod (Clé F — git fetch depuis Git HP)
GIT_SSH_COMMAND="ssh -i ~/.ssh/id_gitprod_githp" \
  git ls-remote deploy@git-hp:/opt/git/myapp.git

# Depuis App HP (Clé G — git fetch depuis Git HP)
GIT_SSH_COMMAND="ssh -i ~/.ssh/id_apphp_githp" \
  git ls-remote deploy@git-hp:/opt/git/myapp.git

# Depuis App Prod (Clé H — git fetch depuis Git Prod)
GIT_SSH_COMMAND="ssh -i ~/.ssh/id_appprod_gitprod" \
  git ls-remote deploy@git-prod:/opt/git/myapp.git
```

---

## Matrice de permissions réseau (pare-feu)

| Source         | Destination    | Port | Protocole | Règle     |
|----------------|----------------|------|-----------|-----------|
| Orchestrateur  | Git HP         | 22   | TCP       | AUTORISER |
| Orchestrateur  | App HP         | 22   | TCP       | AUTORISER |
| Orchestrateur  | Git Prod       | 22   | TCP       | AUTORISER |
| Orchestrateur  | App Prod       | 22   | TCP       | AUTORISER |
| Git HP         | GitLab         | 22   | TCP       | AUTORISER |
| Git Prod       | Git HP         | 22   | TCP       | AUTORISER |
| App HP         | Git HP         | 22   | TCP       | AUTORISER |
| App Prod       | Git Prod       | 22   | TCP       | AUTORISER |
| GitLab         | tout interne   | *    | *         | BLOQUER   |
| Internet       | tout interne   | *    | *         | BLOQUER   |

---

## Procédure de rotation des clés (sans interruption)

```
1. Générer la nouvelle paire de clés (nouvelle_cle / nouvelle_cle.pub)
2. Ajouter nouvelle_cle.pub dans authorized_keys côté destination
3. Tester la nouvelle clé (ssh -i nouvelle_cle ...)
4. Mettre à jour la configuration Semaphore (Key Store) avec la nouvelle clé privée
5. Supprimer l'ancienne clé publique de authorized_keys
6. Supprimer l'ancienne clé privée

Identification des clés : utiliser les commentaires (-C lors de ssh-keygen)
pour retrouver facilement quelle clé supprimer dans authorized_keys.
```
