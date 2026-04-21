# Installation hors-ligne RHEL9 — Zone Production (PROD)

> Document dédié à la zone Production. Pour la zone Hors-Prod, voir [offline-installation-rhel9_hp.md](./offline-installation-rhel9_hp.md).

Ce guide décrit l'installation de tous les composants CI/CD sur les serveurs
RHEL9 de la zone Production sans accès Internet.

---

## 1. Serveurs cibles de la zone PROD

| Composant         | Rôle                          | Serveurs PROD cibles                      |
|-------------------|-------------------------------|-------------------------------------------|
| Git               | Gestion des dépôts            | git-ansible-prod, app[n]-prod             |
| Ansible           | Moteur d'automatisation       | git-ansible-prod                          |
| Ansible Semaphore | Interface web CI/CD           | semaphore-prod                            |
| Nginx             | Reverse proxy HTTPS           | semaphore-prod                            |
| OpenSSH           | Transferts sécurisés          | Tous (déjà présent)                       |
| Python 3          | Dépendance Ansible            | git-ansible-prod                          |
| PostgreSQL client | Sauvegardes BDD avant deploy  | app[n]-prod (pg_dump)                     |

---

## 2. Prérequis

- **Machine connectée** : accès Internet, même architecture (x86_64), idéalement RHEL9 ou
  compatible (Rocky Linux 9, AlmaLinux 9) pour garantir la compatibilité des RPMs
- **Support de transfert** : clé USB, partage réseau isolé, ou serveur de fichiers intermédiaire
- **Accès root** sur tous les serveurs cibles PROD
- **Abonnement RHEL9** sur les serveurs cibles
- **Validation HP préalable** : la zone HP doit être opérationnelle avant d'installer la PROD

---

## 3. Téléchargement sur la machine connectée

### 3.1 Créer les répertoires de téléchargement

```bash
mkdir -p /tmp/offline-packages/{rpms,pip,semaphore,ansible-collections}
```

### 3.2 Packages RPM système

```bash
# Git
dnf download --resolve --destdir=/tmp/offline-packages/rpms git

# Python3 et pip (requis pour Ansible)
dnf download --resolve --destdir=/tmp/offline-packages/rpms \
  python3 python3-pip python3-devel python3-setuptools

# Ansible Core
dnf download --resolve --destdir=/tmp/offline-packages/rpms ansible-core

# Nginx (reverse proxy HTTPS pour Semaphore PROD)
dnf download --resolve --destdir=/tmp/offline-packages/rpms nginx

# OpenSSL, outils réseau et client PostgreSQL
dnf download --resolve --destdir=/tmp/offline-packages/rpms \
  openssl openldap-clients postgresql

# Outils système supplémentaires
dnf download --resolve --destdir=/tmp/offline-packages/rpms \
  rsync tar gzip curl wget
```

> **Note** : `dnf download --resolve` télécharge le package ET toutes ses dépendances
> transitives. Testez toujours l'installation sur une machine RHEL9 propre avant le
> déploiement réel.

### 3.3 Ansible Semaphore (binaire — pour semaphore-prod)

```bash
SEMAPHORE_VERSION="2.10.22"

wget -P /tmp/offline-packages/semaphore \
  "https://github.com/semaphoreui/semaphore/releases/download/v${SEMAPHORE_VERSION}/semaphore_${SEMAPHORE_VERSION}_linux_amd64.tar.gz"

# Vérifier le checksum
wget -P /tmp/offline-packages/semaphore \
  "https://github.com/semaphoreui/semaphore/releases/download/v${SEMAPHORE_VERSION}/semaphore_${SEMAPHORE_VERSION}_checksums.txt"
sha256sum -c <(grep linux_amd64.tar.gz /tmp/offline-packages/semaphore/semaphore_${SEMAPHORE_VERSION}_checksums.txt)
```

> Utiliser la **même version** que sur Semaphore HP pour garantir la cohérence des templates.

### 3.4 Collections Ansible (pour git-ansible-prod)

```bash
# Collection community.general (mail, modules supplémentaires)
ansible-galaxy collection download community.general \
  -p /tmp/offline-packages/ansible-collections

# Collection community.postgresql (pour les sauvegardes BDD avant déploiement PROD)
ansible-galaxy collection download community.postgresql \
  -p /tmp/offline-packages/ansible-collections
```

### 3.5 Packages Python supplémentaires

```bash
pip download ansible-core --dest /tmp/offline-packages/pip
pip download jinja2 pyyaml cryptography --dest /tmp/offline-packages/pip
```

---

## 4. Transfert vers les serveurs PROD

```bash
# Vers Semaphore PROD
scp -r /tmp/offline-packages/ deploy@semaphore-prod.company.com:/tmp/

# Vers Git+Ansible PROD
scp -r /tmp/offline-packages/ deploy@git-ansible-prod.company.com:/tmp/

# Vers App Server(s) PROD (répéter pour chaque app[n]-prod)
scp -r /tmp/offline-packages/ deploy@app1-prod.company.com:/tmp/

# Alternative via clé USB (monter la clé et copier)
# cp -r /tmp/offline-packages/ /media/usb/
```

---

## 5. Installation sur les serveurs PROD (en tant que root)

### 5.1 Sur tous les serveurs PROD — Packages RPM communs

```bash
cd /tmp/offline-packages/rpms
dnf install --disablerepo="*" *.rpm

# Vérifier
git --version
python3 --version
```

### 5.2 Sur Git+Ansible PROD — Ansible

```bash
# Via RPM
dnf install --disablerepo="*" ansible-core-*.rpm

# Ou via pip
pip3 install --no-index --find-links=/tmp/offline-packages/pip ansible-core
ansible --version

# Collections Ansible (y compris postgresql pour les BDD backups)
ansible-galaxy collection install \
  /tmp/offline-packages/ansible-collections/community-general-*.tar.gz \
  --offline
ansible-galaxy collection install \
  /tmp/offline-packages/ansible-collections/community-postgresql-*.tar.gz \
  --offline
```

### 5.3 Sur Semaphore PROD — Installation Semaphore

```bash
# Extraire le binaire
tar -xzf /tmp/offline-packages/semaphore/semaphore_*_linux_amd64.tar.gz \
  -C /usr/local/bin/ semaphore

chmod +x /usr/local/bin/semaphore
semaphore version

# Nginx
nginx --version
```

### 5.4 Sur App Server(s) PROD — Client PostgreSQL

```bash
# Vérifier que pg_dump est disponible (requis pour sauvegardes avant déploiement)
pg_dump --version
# Si absent :
dnf install --disablerepo="*" postgresql-*.rpm
```

---

## 6. Configuration initiale de Semaphore PROD

### 6.1 Créer l'utilisateur système

```bash
useradd --system --no-create-home --shell /bin/false semaphore
mkdir -p /etc/semaphore /var/lib/semaphore
chown semaphore:semaphore /etc/semaphore /var/lib/semaphore
```

### 6.2 Initialiser Semaphore PROD (premier démarrage interactif)

```bash
semaphore setup
```

Répondre aux questions :
- **DB Driver** : `bolt` (BoltDB embarquée) ou `postgres` (PostgreSQL pour HA)
- **DB path** : `/var/lib/semaphore/semaphore.bolt` (si BoltDB)
- **Playbook path** : `/home/semaphore`
- **Public URL** : `https://semaphore-prod.company.com`
- **Admin login / password** : créer le compte administrateur initial PROD

> **Recommandation PROD** : Pour une haute disponibilité, utiliser PostgreSQL
> à la place de BoltDB. Dans ce cas, installer le RPM postgresql offline et
> configurer en conséquence.

### 6.3 Fichier d'environnement PROD

```bash
touch /etc/semaphore/semaphore.env
chmod 640 /etc/semaphore/semaphore.env
chown root:semaphore /etc/semaphore/semaphore.env

echo "SEMAPHORE_WEB_HOST=https://semaphore-prod.company.com" >> /etc/semaphore/semaphore.env
```

Voir [ldap-https-configuration_prod.md](./ldap-https-configuration_prod.md) pour le contenu complet.

### 6.4 Unité systemd (Semaphore PROD)

Créer `/etc/systemd/system/semaphore.service` :

```ini
[Unit]
Description=Ansible Semaphore PROD
Documentation=https://docs.semaphoreui.com
After=network.target

[Service]
Type=simple
User=semaphore
Group=semaphore
WorkingDirectory=/var/lib/semaphore
ExecStart=/usr/local/bin/semaphore server --config /etc/semaphore/config.json
Restart=on-failure
RestartSec=10
EnvironmentFile=/etc/semaphore/semaphore.env

# Sécurité
NoNewPrivileges=true
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload
systemctl enable semaphore
systemctl start semaphore
systemctl status semaphore
```

---

## 7. Configuration Nginx PROD

Voir le guide complet dans [ldap-https-configuration_prod.md](./ldap-https-configuration_prod.md) (section 3).

```bash
# Résumé rapide
nginx -t
systemctl enable nginx
systemctl start nginx
```

---

## 8. Considérations spécifiques RHEL9 (zone PROD)

### 8.1 SELinux (enforcing par défaut)

```bash
# Pour Nginx → Semaphore PROD (proxy réseau)
setsebool -P httpd_can_network_connect 1

# Restaurer les contextes SELinux du binaire Semaphore
restorecon -v /usr/local/bin/semaphore

# Vérifier les refus SELinux si problème
ausearch -m avc -ts recent | audit2allow -a
```

### 8.2 Firewalld (zone PROD)

```bash
# Ouvrir SSH (si pas déjà ouvert)
firewall-cmd --permanent --add-service=ssh

# Ouvrir HTTPS pour l'interface Semaphore PROD
firewall-cmd --permanent --add-service=https

# Appliquer
firewall-cmd --reload
firewall-cmd --list-all
```

### 8.3 Vérification des dépendances RPM

```bash
dnf download --resolve --destdir=/tmp/test-deps ansible-core
rpm -qpR /tmp/test-deps/*.rpm | sort -u > /tmp/deps-requises.txt
```

### 8.4 Mises à jour offline de Semaphore PROD

Planifier les mises à jour en dehors des heures de pointe :
1. Télécharger le nouveau binaire sur une machine connectée
2. Transférer vers `semaphore-prod`
3. Annoncer la maintenance
4. `systemctl stop semaphore`
5. Remplacer le binaire : `cp semaphore_new /usr/local/bin/semaphore`
6. `systemctl start semaphore`
7. Vérifier le bon fonctionnement avant de lever la maintenance

---

## 9. Langages supplémentaires sur App Server(s) PROD

À installer sur les serveurs `app[n]-prod` selon la stack applicative.
Les versions doivent correspondre exactement à celles installées en HP.

### Java

```bash
dnf download --resolve --destdir=/tmp/offline-packages/rpms java-17-openjdk java-17-openjdk-devel
dnf install --disablerepo="*" /tmp/offline-packages/rpms/*.rpm
java -version
```

### Node.js

```bash
dnf download --resolve --destdir=/tmp/offline-packages/rpms nodejs npm
dnf install --disablerepo="*" /tmp/offline-packages/rpms/*.rpm
node --version
npm --version
```

> **Mode offline npm** : Transférer le cache npm (`~/.npm`) depuis une machine connectée.
> Alternative recommandée : déployer **Verdaccio** (registre npm privé) sur un serveur
> interne PROD pour servir les packages Node.js.

### Go

```bash
GO_VERSION="1.22.0"
# Transférer l'archive depuis une machine connectée
tar -C /usr/local -xzf go${GO_VERSION}.linux-amd64.tar.gz
echo 'export PATH=$PATH:/usr/local/go/bin' >> /etc/profile.d/go.sh
source /etc/profile.d/go.sh
go version
```

---

## 10. Vérification finale (zone PROD)

```bash
# Sur semaphore-prod
git --version          # → git version 2.x.x
semaphore version      # → 2.x.x (même version que HP)
nginx -v               # → nginx version: nginx/1.x.x
systemctl status semaphore
systemctl status nginx
curl -Ik https://semaphore-prod.company.com   # → HTTP/2 200

# Sur git-ansible-prod
git --version
ansible --version      # → ansible [core 2.x.x]

# Sur app[n]-prod
git --version
pg_dump --version      # CRITIQUE : requis pour sauvegardes avant déploiement
# Vérifier le runtime applicatif selon la stack

# Test de connectivité vers Git+Ansible HP (clé F)
ssh -i /opt/keys/id_gitprod_to_githp deploy@git-ansible-hp.company.com \
  "git -C /opt/git/myapp.git tag -l 'validated/hp/myapp'"
```
