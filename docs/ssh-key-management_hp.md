# Gestion des Clés SSH — Zone Hors-Prod (HP)

> Document dédié à la zone HP. Pour la zone Production, voir [ssh-key-management_prod.md](./ssh-key-management_prod.md).

---

## Vue d'ensemble des clés SSH de la zone HP

La zone HP utilise **4 paires de clés SSH** (A, B, C, D) :

**Clés Ansible HP (A, C)** — Semaphore HP pilote ses serveurs cibles.
**Clés Git HP (B, D)** — Les serveurs tirent le code depuis leur source.

| Clé | Source             | Destination        | Usage                                       | Fichier clé privée (sur la source)    |
|-----|--------------------|--------------------|---------------------------------------------|---------------------------------------|
| A   | Semaphore HP       | Git+Ansible HP     | Ansible pilote HP (synchro + validation)    | `~/.ssh/id_semaphore_hp`              |
| B   | Git+Ansible HP     | GitLab             | git fetch bare HP ← GitLab                 | `/opt/keys/id_githp_gitlab`           |
| C   | Semaphore HP       | App Server(s) HP   | Ansible pilote le déploiement HP            | `~/.ssh/id_semaphore_hp_apps`         |
| D   | App Server(s) HP   | Git+Ansible HP     | git fetch code HP (read-only)               | `/opt/keys/id_apphp_to_githp`         |

> **Clé D** : strictement read-only (`command="git-upload-pack ..."` dans `authorized_keys`).

---

## Génération des clés HP

### Sur Semaphore HP

```bash
su - semaphore

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

---

## Distribution des clés publiques HP

### Sur Git+Ansible HP — autorise : A, D, F

```bash
# Sur git-ansible-hp, en tant que deploy

# Clé A : Semaphore HP a un accès SSH complet (pour Ansible)
echo "# Clé A — Ansible Semaphore HP" >> ~/.ssh/authorized_keys
cat /tmp/id_semaphore_hp.pub >> ~/.ssh/authorized_keys

# Clé D : App HP peut uniquement lire les dépôts bare (git-upload-pack)
# Wrapper recommandé pour multi-dépôts (voir section ci-dessous)
echo 'command="/home/deploy/bin/git-read-wrapper.sh",no-port-forwarding,no-X11-forwarding,no-agent-forwarding,no-pty <contenu id_apphp_to_githp.pub>' \
  >> ~/.ssh/authorized_keys

# Clé F (PROD) : Git+Ansible Prod a un accès SSH complet (check flag + git fetch)
# Cette clé est configurée depuis le côté PROD — voir ssh-key-management_prod.md
echo "# Clé F — Git+Ansible Prod (sera ajoutée lors de l'installation PROD)" >> ~/.ssh/authorized_keys
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
```

### Sur App Server(s) HP — autorise : C

```bash
# Sur chaque app[n]-hp, en tant que deploy
echo "# Clé C — Ansible Semaphore HP" >> ~/.ssh/authorized_keys
cat /tmp/id_semaphore_hp_apps.pub >> ~/.ssh/authorized_keys
```

### Sur GitLab — autorise : B

**Option 1 — Deploy Key sur le projet** (recommandée) :
- **Projet → Settings → Repository → Deploy Keys**
- Ajouter le contenu de `id_githp_gitlab.pub`
- Accès **lecture seule** uniquement

**Option 2 — Clé SSH sur un compte `ci-reader`** :
- **Profile → SSH Keys → Add new key**
- Coller le contenu de `id_githp_gitlab.pub`
- Titre : `Git+Ansible HP deploy key`

---

## Fichiers `~/.ssh/config` sur chaque serveur HP

### Sur Semaphore HP

```ssh-config
# Git+Ansible HP — Ansible (Clé A)
Host git-ansible-hp
    HostName git-ansible-hp.company.com
    User deploy
    IdentityFile ~/.ssh/id_semaphore_hp
    IdentitiesOnly yes
    StrictHostKeyChecking accept-new

# App HP — Ansible (Clé C)
# Les app servers HP sont configurés dans l'inventory ansible/inventory/hp/
```

### Sur Git+Ansible HP

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

### Sur App Server(s) HP

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

---

## Enregistrement des known_hosts (zone HP)

Pré-enregistrer les fingerprints pour éviter les prompts interactifs :

```bash
# Sur Semaphore HP
ssh-keyscan -H git-ansible-hp.company.com >> ~/.ssh/known_hosts

# Sur Git+Ansible HP
ssh-keyscan -H gitlab.company.com >> ~/.ssh/known_hosts

# Sur App Server(s) HP
ssh-keyscan -H git-ansible-hp.company.com >> ~/.ssh/known_hosts
```

---

## Tests de connectivité HP

```bash
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
```

---

## Matrice de permissions réseau (zone HP)

| Source             | Destination        | Port | Protocole | Règle     |
|--------------------|--------------------|----- |-----------|-----------|
| Semaphore HP       | Git+Ansible HP     | 22   | TCP       | AUTORISER |
| Semaphore HP       | App Server(s) HP   | 22   | TCP       | AUTORISER |
| Git+Ansible HP     | GitLab             | 22   | TCP       | AUTORISER |
| App Server(s) HP   | Git+Ansible HP     | 22   | TCP       | AUTORISER |
| Semaphore HP       | Zone Prod          | *    | *         | BLOQUER   |
| Internet           | tout interne HP    | *    | *         | BLOQUER   |

---

## Procédure de rotation des clés HP

```
1. Générer la nouvelle paire de clés :
     ssh-keygen -t ed25519 -C "<commentaire>" -f <nouveau_fichier> -N ""

2. Ajouter la nouvelle clé publique dans authorized_keys côté destination
   (les deux clés coexistent temporairement)

3. Tester la nouvelle clé :
     ssh -i <nouveau_fichier> <user>@<destination> "echo OK"

4. Mettre à jour la configuration Semaphore HP (Key Store) avec la nouvelle clé privée
   OU mettre à jour le chemin dans les group_vars/ Ansible

5. Supprimer l'ancienne clé publique de authorized_keys côté destination
   (identifier par le commentaire dans la clé)

6. Supprimer l'ancienne clé privée du serveur source
```

> **Astuce** : Le commentaire généré avec `-C` lors de `ssh-keygen` apparaît
> à la fin de chaque ligne dans `authorized_keys`. Il permet d'identifier
> précisément quelle clé supprimer sans risque de confusion.
