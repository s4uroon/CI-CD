# Gestion des Clés SSH — Zone Production (PROD)

> Document dédié à la zone Production. Pour la zone Hors-Prod, voir [ssh-key-management_hp.md](./ssh-key-management_hp.md).

---

## Vue d'ensemble des clés SSH de la zone PROD

La zone PROD utilise **4 paires de clés SSH** (E, F, G, H) :

**Clés Ansible PROD (E, G)** — Semaphore PROD pilote ses serveurs cibles.
**Clés Git PROD (F, H)** — Les serveurs tirent le code depuis leur source.

| Clé | Source             | Destination        | Usage                                       | Fichier clé privée (sur la source)     |
|-----|--------------------|--------------------|---------------------------------------------|----------------------------------------|
| E   | Semaphore Prod     | Git+Ansible Prod   | Ansible pilote Prod (synchro + déploiement) | `~/.ssh/id_semaphore_prod`             |
| F   | Git+Ansible Prod   | Git+Ansible HP     | SSH check flag + git fetch bare PROD ← HP   | `/opt/keys/id_gitprod_to_githp`        |
| G   | Semaphore Prod     | App Server(s) Prod | Ansible pilote le déploiement Prod          | `~/.ssh/id_semaphore_prod_apps`        |
| H   | App Server(s) Prod | Git+Ansible Prod   | git fetch code Prod (read-only)             | `/opt/keys/id_appprod_to_gitprod`      |

> **Clé H** : strictement read-only (`command="git-upload-pack ..."` dans `authorized_keys`).
> **Clé F** : accès SSH complet vers Git+Ansible HP (vérification flag `validated/hp/<repo>` + git fetch).

---

## Génération des clés PROD

### Sur Semaphore PROD

```bash
su - semaphore

# Clé E : Semaphore Prod → Git+Ansible Prod (Ansible)
ssh-keygen -t ed25519 -C "semaphore-prod-to-git-ansible-prod" \
  -f ~/.ssh/id_semaphore_prod -N ""

# Clé G : Semaphore Prod → App Server(s) Prod (Ansible)
ssh-keygen -t ed25519 -C "semaphore-prod-to-app-prod" \
  -f ~/.ssh/id_semaphore_prod_apps -N ""
```

### Sur Git+Ansible PROD

```bash
su - deploy

# Clé F : Git+Ansible Prod → Git+Ansible HP (SSH check flag + git fetch)
ssh-keygen -t ed25519 -C "git-ansible-prod-to-git-ansible-hp" \
  -f /opt/keys/id_gitprod_to_githp -N ""
chmod 600 /opt/keys/id_gitprod_to_githp
```

### Sur App Server(s) PROD

```bash
su - deploy

# Clé H : App Prod → Git+Ansible Prod (git fetch working tree, read-only)
ssh-keygen -t ed25519 -C "app-prod-to-git-ansible-prod" \
  -f /opt/keys/id_appprod_to_gitprod -N ""
chmod 600 /opt/keys/id_appprod_to_gitprod
```

---

## Distribution des clés publiques PROD

### Sur Git+Ansible HP — autorise également : F

```bash
# Sur git-ansible-hp, en tant que deploy
# (à exécuter lors de l'installation de la zone PROD)

# Clé F : Git+Ansible Prod a un accès SSH complet (check flag + git fetch)
echo "# Clé F — Git+Ansible Prod" >> ~/.ssh/authorized_keys
cat /tmp/id_gitprod_to_githp.pub >> ~/.ssh/authorized_keys
```

> Cette clé est ajoutée sur un serveur HP — coordonner avec l'administrateur HP.

### Sur Git+Ansible PROD — autorise : E, H

```bash
# Sur git-ansible-prod, en tant que deploy

# Clé E : Semaphore Prod a un accès SSH complet (pour Ansible)
echo "# Clé E — Ansible Semaphore Prod" >> ~/.ssh/authorized_keys
cat /tmp/id_semaphore_prod.pub >> ~/.ssh/authorized_keys

# Clé H : App Prod peut uniquement lire les dépôts bare (git-upload-pack)
echo 'command="/home/deploy/bin/git-read-wrapper.sh",no-port-forwarding,no-X11-forwarding,no-agent-forwarding,no-pty <contenu id_appprod_to_gitprod.pub>' \
  >> ~/.ssh/authorized_keys
```

**Script wrapper multi-dépôts pour Clé H** (sur Git+Ansible PROD) :

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

### Sur App Server(s) PROD — autorise : G

```bash
# Sur chaque app[n]-prod, en tant que deploy
echo "# Clé G — Ansible Semaphore Prod" >> ~/.ssh/authorized_keys
cat /tmp/id_semaphore_prod_apps.pub >> ~/.ssh/authorized_keys
```

---

## Fichiers `~/.ssh/config` sur chaque serveur PROD

### Sur Semaphore PROD

```ssh-config
# Git+Ansible Prod — Ansible (Clé E)
Host git-ansible-prod
    HostName git-ansible-prod.company.com
    User deploy
    IdentityFile ~/.ssh/id_semaphore_prod
    IdentitiesOnly yes
    StrictHostKeyChecking accept-new

# App Prod — Ansible (Clé G)
# Les app servers Prod sont configurés dans ansible/inventory/prod/
```

### Sur Git+Ansible PROD

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

### Sur App Server(s) PROD

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

## Enregistrement des known_hosts (zone PROD)

```bash
# Sur Semaphore PROD
ssh-keyscan -H git-ansible-prod.company.com >> ~/.ssh/known_hosts

# Sur Git+Ansible PROD
ssh-keyscan -H git-ansible-hp.company.com >> ~/.ssh/known_hosts

# Sur App Server(s) PROD
ssh-keyscan -H git-ansible-prod.company.com >> ~/.ssh/known_hosts
```

---

## Tests de connectivité PROD

```bash
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

## Matrice de permissions réseau (zone PROD)

| Source             | Destination        | Port | Protocole | Règle     |
|--------------------|--------------------|----- |-----------|-----------|
| Semaphore Prod     | Git+Ansible Prod   | 22   | TCP       | AUTORISER |
| Semaphore Prod     | App Server(s) Prod | 22   | TCP       | AUTORISER |
| Git+Ansible Prod   | Git+Ansible HP     | 22   | TCP       | AUTORISER |
| App Server(s) Prod | Git+Ansible Prod   | 22   | TCP       | AUTORISER |
| Semaphore Prod     | Zone HP (sauf Git HP TCP/22) | * | * | BLOQUER |
| GitLab             | tout interne Prod  | *    | *         | BLOQUER   |
| Internet           | tout interne Prod  | *    | *         | BLOQUER   |

---

## Procédure de rotation des clés PROD

```
1. Générer la nouvelle paire de clés :
     ssh-keygen -t ed25519 -C "<commentaire>" -f <nouveau_fichier> -N ""

2. Ajouter la nouvelle clé publique dans authorized_keys côté destination
   (les deux clés coexistent temporairement — pas d'interruption de service)

3. Tester la nouvelle clé :
     ssh -i <nouveau_fichier> <user>@<destination> "echo OK"

4. Mettre à jour la configuration Semaphore PROD (Key Store) avec la nouvelle clé privée
   OU mettre à jour le chemin dans les group_vars/ Ansible

5. Supprimer l'ancienne clé publique de authorized_keys côté destination
   (identifier par le commentaire dans la clé : "semaphore-prod-to-git-ansible-prod", etc.)

6. Supprimer l'ancienne clé privée du serveur source
```

> **Note spéciale pour la clé F** : La rotation de la clé F nécessite une coordination
> entre les équipes HP et PROD, car la clé publique F est sur un serveur HP
> (`authorized_keys` de Git+Ansible HP).
