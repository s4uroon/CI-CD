# Gestion des Clés SSH

## Vue d'ensemble des 4 paires de clés

| Paire | Identifiant       | Source          | Destination   | Usage                                     |
|-------|-------------------|-----------------|---------------|-------------------------------------------|
| A     | `id_orch_hp`      | Orchestrateur   | Hors-Prod     | Ansible execute playbooks 1 & 2 sur HP    |
| B     | `id_orch_prod`    | Orchestrateur   | Prod          | Ansible execute playbooks 3, 4 & 5 sur Prod |
| C     | `id_prod_hp_git`  | Prod            | Hors-Prod     | Git fetch : Prod tire le code depuis HP   |
| D     | `id_hp_gitlab`    | Hors-Prod       | GitLab        | Git fetch : HP tire le code depuis GitLab |

---

## Génération des clés

### Sur l'Orchestrateur (Semaphore)

```bash
# Connecté en tant que l'utilisateur Semaphore (ex: semaphore)
su - semaphore

# Clé A : Orchestrateur -> Hors-Prod (Ansible)
ssh-keygen -t ed25519 -C "orch-to-hors-prod-ansible" \
  -f ~/.ssh/id_orch_hp -N ""

# Clé B : Orchestrateur -> Prod (Ansible)
ssh-keygen -t ed25519 -C "orch-to-prod-ansible" \
  -f ~/.ssh/id_orch_prod -N ""

# Vérification
ls -la ~/.ssh/
# id_orch_hp      id_orch_hp.pub
# id_orch_prod    id_orch_prod.pub
```

### Sur Hors-Prod

```bash
# Connecté en tant que l'utilisateur de déploiement (ex: deploy)
su - deploy

# Clé D : Hors-Prod -> GitLab (git fetch)
ssh-keygen -t ed25519 -C "hors-prod-to-gitlab-git" \
  -f ~/.ssh/id_hp_gitlab -N ""
```

### Sur Prod

```bash
# Connecté en tant que l'utilisateur de déploiement (ex: deploy)
su - deploy

# Clé C : Prod -> Hors-Prod (git fetch)
ssh-keygen -t ed25519 -C "prod-to-hors-prod-git" \
  -f ~/.ssh/id_prod_hp_git -N ""
```

---

## Distribution des clés publiques

### Clé A (Orchestrateur → Hors-Prod)

Sur **Hors-Prod**, ajouter la clé publique de l'Orchestrateur :

```bash
# Sur Hors-Prod, en tant que deploy
mkdir -p ~/.ssh && chmod 700 ~/.ssh
cat >> ~/.ssh/authorized_keys << 'EOF'
# Clé A — Ansible Orchestrateur
<contenu de id_orch_hp.pub depuis l'Orchestrateur>
EOF
chmod 600 ~/.ssh/authorized_keys
```

Ou via Ansible (playbook d'initialisation) :

```yaml
- name: Autoriser la clé Ansible de l'Orchestrateur sur Hors-Prod
  ansible.posix.authorized_key:
    user: deploy
    state: present
    key: "{{ lookup('file', '~/.ssh/id_orch_hp.pub') }}"
    comment: "Ansible Orchestrateur - Clé A"
```

### Clé B (Orchestrateur → Prod)

Identique à la Clé A, mais sur le serveur **Prod** :

```bash
# Sur Prod, en tant que deploy
cat >> ~/.ssh/authorized_keys << 'EOF'
# Clé B — Ansible Orchestrateur
<contenu de id_orch_prod.pub depuis l'Orchestrateur>
EOF
```

### Clé C (Prod → Hors-Prod, git uniquement)

Sur **Hors-Prod**, ajouter la clé publique de Prod avec des **restrictions** :

```bash
# Sur Hors-Prod, en tant que deploy
cat >> ~/.ssh/authorized_keys << 'EOF'
# Clé C — Git fetch depuis Prod (lecture seule, git-upload-pack uniquement)
command="git-upload-pack '/opt/apps/myapp'",no-port-forwarding,no-X11-forwarding,no-agent-forwarding,no-pty <contenu de id_prod_hp_git.pub depuis Prod>
EOF
```

> **Important** : La restriction `command="git-upload-pack ..."` limite la clé C
> au seul usage git (lecture seule). Même si la clé est compromise, elle ne peut
> pas ouvrir un shell interactif.

Pour autoriser plusieurs dépôts, utiliser un wrapper script :

```bash
# /home/deploy/bin/git-shell-wrapper.sh
#!/bin/bash
# Autorise uniquement git-upload-pack sur les dépôts listés
ALLOWED_REPOS=(
  "/opt/apps/myapp"
  "/opt/apps/myapp2"
)

if [[ "$SSH_ORIGINAL_COMMAND" =~ ^git-upload-pack\ \'(.+)\'$ ]]; then
  REPO="${BASH_REMATCH[1]}"
  for allowed in "${ALLOWED_REPOS[@]}"; do
    if [[ "$REPO" == "$allowed" ]]; then
      exec git-upload-pack "$REPO"
    fi
  done
fi

echo "Accès refusé : dépôt non autorisé ou commande invalide" >&2
exit 1
```

```bash
# Dans authorized_keys sur Hors-Prod :
command="/home/deploy/bin/git-shell-wrapper.sh",no-port-forwarding,no-X11-forwarding,no-agent-forwarding,no-pty <clé pub Prod>
```

### Clé D (Hors-Prod → GitLab)

Sur **GitLab**, ajouter la clé publique de l'utilisateur de lecture :

1. Se connecter à GitLab avec le compte `ci-reader`
2. Aller dans **Profile → SSH Keys**
3. Coller le contenu de `id_hp_gitlab.pub`
4. Nommer : `Hors-Prod deploy key`
5. Valider

Ou via GitLab Deploy Keys (par projet, en lecture seule) — recommandé pour
limiter l'accès au minimum nécessaire.

---

## Configuration SSH côté clients

### Fichier `~/.ssh/config` sur l'Orchestrateur

```ssh-config
# Hors-Prod — Ansible (Clé A)
Host hors-prod
    HostName hors-prod.company.com
    User deploy
    IdentityFile ~/.ssh/id_orch_hp
    IdentitiesOnly yes
    StrictHostKeyChecking accept-new
    ServerAliveInterval 60
    ServerAliveCountMax 3

# Prod — Ansible (Clé B)
Host prod
    HostName prod.company.com
    User deploy
    IdentityFile ~/.ssh/id_orch_prod
    IdentitiesOnly yes
    StrictHostKeyChecking accept-new
    ServerAliveInterval 60
    ServerAliveCountMax 3
```

### Fichier `~/.ssh/config` sur Prod (pour git fetch)

```ssh-config
# Hors-Prod — Git fetch uniquement (Clé C)
Host hors-prod-git
    HostName hors-prod.company.com
    User deploy
    IdentityFile ~/.ssh/id_prod_hp_git
    IdentitiesOnly yes
    StrictHostKeyChecking accept-new
    # Pas de ForwardAgent, pas de X11
    ForwardAgent no
    ForwardX11 no
```

Le remote Git sur Prod pointe alors vers :
```
git remote set-url staging git+ssh://hors-prod-git/opt/apps/myapp
# ou simplement :
git remote set-url staging deploy@hors-prod-git:/opt/apps/myapp
```

### Fichier `~/.ssh/config` sur Hors-Prod (pour git fetch GitLab)

```ssh-config
# GitLab — Git fetch uniquement (Clé D)
Host gitlab
    HostName gitlab.company.com
    User git
    IdentityFile ~/.ssh/id_hp_gitlab
    IdentitiesOnly yes
    StrictHostKeyChecking accept-new
    ForwardAgent no
    ForwardX11 no
```

---

## Ajout des known_hosts (anti-MITM)

Pré-enregistrer les fingerprints pour éviter les prompts interactifs :

### Sur l'Orchestrateur

```bash
# Enregistrer les fingerprints de Hors-Prod et Prod
ssh-keyscan -H hors-prod.company.com >> ~/.ssh/known_hosts
ssh-keyscan -H prod.company.com       >> ~/.ssh/known_hosts
```

### Sur Hors-Prod

```bash
# Enregistrer le fingerprint de GitLab
ssh-keyscan -H gitlab.company.com >> ~/.ssh/known_hosts
```

### Sur Prod

```bash
# Enregistrer le fingerprint de Hors-Prod (pour git)
ssh-keyscan -H hors-prod.company.com >> ~/.ssh/known_hosts
```

> Vérifier les fingerprints manuellement lors de la première installation
> en les comparant avec ceux fournis par l'administrateur système.

---

## Tests de connectivité

### Depuis l'Orchestrateur

```bash
# Test Clé A (Ansible vers Hors-Prod)
ssh -i ~/.ssh/id_orch_hp deploy@hors-prod "echo 'Clé A OK'"

# Test Clé B (Ansible vers Prod)
ssh -i ~/.ssh/id_orch_prod deploy@prod "echo 'Clé B OK'"
```

### Depuis Hors-Prod

```bash
# Test Clé D (git fetch depuis GitLab)
ssh -T git@gitlab.company.com
# Réponse attendue : "Welcome to GitLab, @ci-reader!"
```

### Depuis Prod

```bash
# Test Clé C (git fetch depuis Hors-Prod)
ssh -i ~/.ssh/id_prod_hp_git deploy@hors-prod "echo 'Clé C OK'"

# Test git clone depuis Hors-Prod
GIT_SSH_COMMAND="ssh -i ~/.ssh/id_prod_hp_git" \
  git ls-remote deploy@hors-prod.company.com:/opt/apps/myapp
```

---

## Rotation des clés

La rotation doit être effectuée sans interruption de service :

1. Générer la nouvelle paire de clés
2. Ajouter la **nouvelle clé publique** dans `authorized_keys` côté destination
3. Tester la nouvelle clé
4. Mettre à jour la configuration Semaphore (nouvelle clé privée)
5. Supprimer l'**ancienne clé publique** de `authorized_keys`
6. Supprimer l'ancienne clé privée

```bash
# Etape 2 : ajouter sans supprimer l'ancienne
echo "<nouvelle_cle_pub>" >> ~/.ssh/authorized_keys

# Etape 5 : supprimer l'ancienne (identifier par commentaire ou debut de cle)
sed -i '/orch-to-hors-prod-ansible-old/d' ~/.ssh/authorized_keys
```

---

## Matrice de permissions réseau (pare-feu)

| Source            | Destination       | Port | Protocole | Règle        |
|-------------------|-------------------|------|-----------|--------------|
| Orchestrateur     | Hors-Prod         | 22   | TCP       | AUTORISER    |
| Orchestrateur     | Prod              | 22   | TCP       | AUTORISER    |
| Prod              | Hors-Prod         | 22   | TCP       | AUTORISER    |
| Hors-Prod         | GitLab            | 22   | TCP       | AUTORISER    |
| GitLab            | Hors-Prod/Prod    | *    | *         | BLOQUER      |
| Internet          | tous serveurs     | *    | *         | BLOQUER      |
| Prod              | Internet          | *    | *         | BLOQUER      |
