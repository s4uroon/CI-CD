# Gestion des Clés SSH

## Vue d'ensemble des 8 paires de clés

L'architecture à 2 zones nécessite 8 paires de clés SSH distinctes,
réparties en deux catégories :

**Clés Ansible (A, C, E, G)** — les Semaphore pilotent leurs serveurs cibles.
**Clés Git (B, D, F, H)** — les serveurs tirent le code depuis leur source.

| Clé | Source             | Destination        | Usage                                       | Fichier clé privée (sur la source)               |
|-----|--------------------|--------------------|---------------------------------------------|--------------------------------------------------|
| A   | Semaphore HP       | Git+Ansible HP     | Ansible pilote HP (synchro + validation)    | `~/.ssh/id_semaphore_hp`                         |
| B   | Git+Ansible HP     | GitLab             | git fetch bare HP ← GitLab                 | `/opt/keys/id_githp_gitlab`                      |
| C   | Semaphore HP       | App Server(s) HP   | Ansible pilote le déploiement HP            | `~/.ssh/id_semaphore_hp_apps`                    |
| D   | App Server(s) HP   | Git+Ansible HP     | git fetch code HP (read-only)               | `/opt/keys/id_apphp_to_githp`                    |
| E   | Semaphore Prod     | Git+Ansible Prod   | Ansible pilote Prod (synchro + déploiement) | `~/.ssh/id_semaphore_prod`                       |
| F   | Git+Ansible Prod   | Git+Ansible HP     | SSH check flag + git fetch bare Prod ← HP   | `/opt/keys/id_gitprod_to_githp`                  |
| G   | Semaphore Prod     | App Server(s) Prod | Ansible pilote le déploiement Prod          | `~/.ssh/id_semaphore_prod_apps`                  |
| H   | App Server(s) Prod | Git+Ansible Prod   | git fetch code Prod (read-only)             | `/opt/keys/id_appprod_to_gitprod`                |

> **Clés D et H** : strictement read-only (`command="git-upload-pack ..."` dans `authorized_keys`).
> **Clé F** : accès SSH complet vers Git+Ansible HP (vérification flag + git fetch).

---

## Génération des clés

### Sur Semaphore HP

```bash
su - semaphore   # utilisateur qui lance Ansible Semaphore

# Clé A : Semaphore HP → Git+Ansible HP (Ansible)
ssh-keygen -t ed25519 -C "semaphore-hp-to-git-ansible-hp" \
  -f ~/.ssh/id_semaphore_hp -N ""

# Clé C : Semaphore HP → App Server(s) HP (Ansible)
ssh-keygen -t ed25519 -C "semaphore-hp-to-app-hp" \
  -f ~/.ssh/id_semaphore_hp_apps -N ""
```

### Sur Git+Ansible HP

```bash
su - deploy

# Clé B : Git+Ansible HP → GitLab (git fetch bare)
ssh-keygen -t ed25519 -C "git-ansible-hp-to-gitlab" \
  -f /opt/keys/id_githp_gitlab -N ""
chmod 600 /opt/keys/id_githp_gitlab
```

### Sur App Server(s) HP

```bash
su - deploy

# Clé D : App HP → Git+Ansible HP (git fetch working tree, read-only)
ssh-keygen -t ed25519 -C "app-hp-to-git-ansible-hp" \
  -f /opt/keys/id_apphp_to_githp -N ""
chmod 600 /opt/keys/id_apphp_to_githp
```

### Sur Semaphore Prod

```bash
su - semaphore

# Clé E : Semaphore Prod → Git+Ansible Prod (Ansible)
ssh-keygen -t ed25519 -C "semaphore-prod-to-git-ansible-prod" \
  -f ~/.ssh/id_semaphore_prod -N ""

# Clé G : Semaphore Prod → App Server(s) Prod (Ansible)
ssh-keygen -t ed25519 -C "semaphore-prod-to-app-prod" \
  -f ~/.ssh/id_semaphore_prod_apps -N ""
```

### Sur Git+Ansible Prod

```bash
su - deploy

# Clé F : Git+Ansible Prod → Git+Ansible HP (SSH check flag + git fetch)
ssh-keygen -t ed25519 -C "git-ansible-prod-to-git-ansible-hp" \
  -f /opt/keys/id_gitprod_to_githp -N ""
chmod 600 /opt/keys/id_gitprod_to_githp
```

### Sur App Server(s) Prod

```bash
su - deploy

# Clé H : App Prod → Git+Ansible Prod (git fetch working tree, read-only)
ssh-keygen -t ed25519 -C "app-prod-to-git-ansible-prod" \
  -f /opt/keys/id_appprod_to_gitprod -N ""
chmod 600 /opt/keys/id_appprod_to_gitprod
```

---

## Distribution des clés publiques

### Sur Git+Ansible HP — autorise : A, D, F

```bash
# Sur git-ansible-hp, en tant que deploy
# ~/.ssh/authorized_keys

# Clé A : Semaphore HP a un accès SSH complet (pour Ansible)
echo "# Clé A — Ansible Semaphore HP"
cat /tmp/id_semaphore_hp.pub >> ~/.ssh/authorized_keys

# Clé D : App HP peut uniquement lire les dépôts bare (git-upload-pack)
# Wrapper recommandé pour multi-dépôts (voir section ci-dessous)
echo 'command="git-upload-pack /opt/git/myapp.git",no-port-forwarding,no-X11-forwarding,no-agent-forwarding,no-pty <contenu id_apphp_to_githp.pub>' \
  >> ~/.ssh/authorized_keys

# Clé F : Git+Ansible Prod a un accès SSH complet (check flag + git fetch)
echo "# Clé F — Git+Ansible Prod"
cat /tmp/id_gitprod_to_githp.pub >> ~/.ssh/authorized_keys
```

> **Clé D** : restreinte avec `command="git-upload-pack ..."` — même compromise,
> impossible d'ouvrir un shell ou d'écrire quoi que ce soit.

**Script wrapper multi-dépôts pour Clé D** :

```bash
# /home/deploy/bin/git-read-wrapper.sh
#!/bin/bash
ALLOWED=(
  "/opt/git/myapp.git"
  "/opt/git/api-backend.git"
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
chmod +x /home/deploy/bin/git-read-wrapper.sh
# Dans authorized_keys pour la Clé D :
# command="/home/deploy/bin/git-read-wrapper.sh",no-port-forwarding,...
```

### Sur App Server(s) HP — autorise : C

```bash
# Sur chaque app[n]-hp, en tant que deploy
echo "# Clé C — Ansible Semaphore HP" >> ~/.ssh/authorized_keys
cat /tmp/id_semaphore_hp_apps.pub >> ~/.ssh/authorized_keys
```

### Sur GitLab — autorise : B

Option 1 — **Deploy Key sur le projet** (recommandée, plus restrictive) :
- **Projet → Settings → Repository → Deploy Keys**
- Ajouter le contenu de `id_githp_gitlab.pub`
- Accès **lecture seule** uniquement

Option 2 — Clé SSH sur un compte `ci-reader` :
- **Profile → SSH Keys → Add new key**
- Coller le contenu de `id_githp_gitlab.pub`
- Titre : `Git+Ansible HP deploy key`

### Sur Git+Ansible Prod — autorise : E, H

```bash
# Sur git-ansible-prod, en tant que deploy

# Clé E : Semaphore Prod a un accès SSH complet (pour Ansible)
echo "# Clé E — Ansible Semaphore Prod" >> ~/.ssh/authorized_keys
cat /tmp/id_semaphore_prod.pub >> ~/.ssh/authorized_keys

# Clé H : App Prod peut uniquement lire les dépôts bare (git-upload-pack)
echo 'command="git-upload-pack /opt/git/myapp.git",no-port-forwarding,no-X11-forwarding,no-agent-forwarding,no-pty <contenu id_appprod_to_gitprod.pub>' \
  >> ~/.ssh/authorized_keys
```

### Sur App Server(s) Prod — autorise : G

```bash
# Sur chaque app[n]-prod, en tant que deploy
echo "# Clé G — Ansible Semaphore Prod" >> ~/.ssh/authorized_keys
cat /tmp/id_semaphore_prod_apps.pub >> ~/.ssh/authorized_keys
```

---

## Fichiers `~/.ssh/config` sur chaque serveur

### Semaphore HP

```ssh-config
# Git+Ansible HP — Ansible (Clé A)
Host git-ansible-hp
    HostName git-ansible-hp.company.com
    User deploy
    IdentityFile ~/.ssh/id_semaphore_hp
    IdentitiesOnly yes
    StrictHostKeyChecking accept-new

# App HP — Ansible (Clé C)
# Note : les app servers HP sont configurés dans l'inventory ansible/inventory/hp/
```

### Git+Ansible HP

```ssh-config
# GitLab — git fetch bare (Clé B)
Host gitlab
    HostName gitlab.company.com
    User git
    IdentityFile /opt/keys/id_githp_gitlab
    IdentitiesOnly yes
    StrictHostKeyChecking accept-new
    ForwardAgent no
```

### App Server(s) HP

```ssh-config
# Git+Ansible HP — git fetch working tree (Clé D)
Host git-ansible-hp
    HostName git-ansible-hp.company.com
    User deploy
    IdentityFile /opt/keys/id_apphp_to_githp
    IdentitiesOnly yes
    StrictHostKeyChecking accept-new
    ForwardAgent no
```

### Semaphore Prod

```ssh-config
# Git+Ansible Prod — Ansible (Clé E)
Host git-ansible-prod
    HostName git-ansible-prod.company.com
    User deploy
    IdentityFile ~/.ssh/id_semaphore_prod
    IdentitiesOnly yes
    StrictHostKeyChecking accept-new

# App Prod — Ansible (Clé G)
# Note : les app servers Prod sont configurés dans ansible/inventory/prod/
```

### Git+Ansible Prod

```ssh-config
# Git+Ansible HP — SSH check flag + git fetch (Clé F)
Host git-ansible-hp
    HostName git-ansible-hp.company.com
    User deploy
    IdentityFile /opt/keys/id_gitprod_to_githp
    IdentitiesOnly yes
    StrictHostKeyChecking accept-new
    ForwardAgent no
```

### App Server(s) Prod

```ssh-config
# Git+Ansible Prod — git fetch working tree (Clé H)
Host git-ansible-prod
    HostName git-ansible-prod.company.com
    User deploy
    IdentityFile /opt/keys/id_appprod_to_gitprod
    IdentitiesOnly yes
    StrictHostKeyChecking accept-new
    ForwardAgent no
```

---

## Enregistrement des known_hosts

Pré-enregistrer les fingerprints pour éviter les prompts interactifs :

```bash
# Sur Semaphore HP
ssh-keyscan -H git-ansible-hp.company.com >> ~/.ssh/known_hosts
# (les app HP seront acceptés au premier contact avec StrictHostKeyChecking=accept-new)

# Sur Git+Ansible HP
ssh-keyscan -H gitlab.company.com >> ~/.ssh/known_hosts

# Sur App Server(s) HP
ssh-keyscan -H git-ansible-hp.company.com >> ~/.ssh/known_hosts

# Sur Semaphore Prod
ssh-keyscan -H git-ansible-prod.company.com >> ~/.ssh/known_hosts

# Sur Git+Ansible Prod
ssh-keyscan -H git-ansible-hp.company.com >> ~/.ssh/known_hosts

# Sur App Server(s) Prod
ssh-keyscan -H git-ansible-prod.company.com >> ~/.ssh/known_hosts
```

---

## Tests de connectivité

```bash
# === ZONE HP ===

# Clé A : Semaphore HP → Git+Ansible HP
ssh -i ~/.ssh/id_semaphore_hp deploy@git-ansible-hp "echo 'Clé A OK'"

# Clé B : Git+Ansible HP → GitLab
ssh -T git@gitlab.company.com
# Réponse attendue : "Welcome to GitLab, @ci-reader!"

# Clé C : Semaphore HP → App HP
ssh -i ~/.ssh/id_semaphore_hp_apps deploy@app1-hp "echo 'Clé C OK'"

# Clé D : App HP → Git+Ansible HP (read-only)
GIT_SSH_COMMAND="ssh -i /opt/keys/id_apphp_to_githp" \
  git ls-remote deploy@git-ansible-hp:/opt/git/myapp.git

# === ZONE PROD ===

# Clé E : Semaphore Prod → Git+Ansible Prod
ssh -i ~/.ssh/id_semaphore_prod deploy@git-ansible-prod "echo 'Clé E OK'"

# Clé F : Git+Ansible Prod → Git+Ansible HP (check flag)
ssh -i /opt/keys/id_gitprod_to_githp deploy@git-ansible-hp \
  "git -C /opt/git/myapp.git tag -l 'validated/hp/myapp'"

# Clé F : Git+Ansible Prod → Git+Ansible HP (git fetch)
GIT_SSH_COMMAND="ssh -i /opt/keys/id_gitprod_to_githp" \
  git ls-remote deploy@git-ansible-hp:/opt/git/myapp.git

# Clé G : Semaphore Prod → App Prod
ssh -i ~/.ssh/id_semaphore_prod_apps deploy@app1-prod "echo 'Clé G OK'"

# Clé H : App Prod → Git+Ansible Prod (read-only)
GIT_SSH_COMMAND="ssh -i /opt/keys/id_appprod_to_gitprod" \
  git ls-remote deploy@git-ansible-prod:/opt/git/myapp.git
```

---

## Matrice de permissions réseau (pare-feu)

| Source             | Destination        | Port | Protocole | Règle     |
|--------------------|--------------------| -----|-----------|-----------|
| Semaphore HP       | Git+Ansible HP     | 22   | TCP       | AUTORISER |
| Semaphore HP       | App Server(s) HP   | 22   | TCP       | AUTORISER |
| Git+Ansible HP     | GitLab             | 22   | TCP       | AUTORISER |
| App Server(s) HP   | Git+Ansible HP     | 22   | TCP       | AUTORISER |
| Semaphore Prod     | Git+Ansible Prod   | 22   | TCP       | AUTORISER |
| Semaphore Prod     | App Server(s) Prod | 22   | TCP       | AUTORISER |
| Git+Ansible Prod   | Git+Ansible HP     | 22   | TCP       | AUTORISER |
| App Server(s) Prod | Git+Ansible Prod   | 22   | TCP       | AUTORISER |
| GitLab             | tout interne       | *    | *         | BLOQUER   |
| Internet           | tout interne       | *    | *         | BLOQUER   |
| Semaphore HP       | Zone Prod          | *    | *         | BLOQUER   |
| Semaphore Prod     | Zone HP (sauf Git HP port 22) | * | * | BLOQUER |

---

## Procédure de rotation des clés (sans interruption)

```
1. Générer la nouvelle paire de clés :
     ssh-keygen -t ed25519 -C "<commentaire>" -f <nouveau_fichier> -N ""

2. Ajouter la nouvelle clé publique dans authorized_keys côté destination
   (les deux clés coexistent temporairement)

3. Tester la nouvelle clé :
     ssh -i <nouveau_fichier> <user>@<destination> "echo OK"

4. Mettre à jour la configuration Semaphore (Key Store) avec la nouvelle clé privée
   OU mettre à jour le chemin dans les group_vars/ Ansible

5. Supprimer l'ancienne clé publique de authorized_keys côté destination
   (identifier par le commentaire dans la clé : "git-ansible-hp-to-gitlab", etc.)

6. Supprimer l'ancienne clé privée du serveur source
```

> **Astuce** : Le commentaire généré avec `-C` lors de `ssh-keygen` apparaît
> à la fin de chaque ligne dans `authorized_keys`. Il permet d'identifier
> précisément quelle clé supprimer sans risque de confusion.
