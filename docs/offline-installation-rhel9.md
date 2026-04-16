# Guide d'installation hors-ligne — Red Hat Enterprise Linux 9

Ce guide décrit comment installer tous les composants du CI/CD sur des serveurs
RHEL9 sans accès à Internet. Toutes les étapes de téléchargement sont effectuées
sur une machine connectée, puis les packages sont transférés manuellement.

---

## 1. Vue d'ensemble

| Composant         | Rôle                          | Serveurs cibles                    |
|-------------------|-------------------------------|------------------------------------|
| Git               | Gestion des dépôts            | Tous les serveurs                  |
| Ansible           | Moteur d'automatisation       | git-ansible-hp, git-ansible-prod   |
| Ansible Semaphore | Interface web CI/CD           | semaphore-hp, semaphore-prod       |
| Nginx             | Reverse proxy HTTPS           | semaphore-hp, semaphore-prod       |
| OpenSSH           | Transferts sécurisés          | Tous les serveurs (déjà présent)   |
| Python 3          | Dépendance Ansible            | git-ansible-hp, git-ansible-prod   |

---

## 2. Prérequis

- **Machine connectée** : accès Internet, même architecture (x86_64), idéalement RHEL9 ou
  compatible (Rocky Linux 9, AlmaLinux 9) pour garantir la compatibilité des RPMs
- **Support de transfert** : clé USB, partage réseau isolé, ou serveur de fichiers intermédiaire
- **Accès root** sur tous les serveurs cibles
- **Abonnement RHEL9** sur les serveurs cibles (pour la résolution RPM) OU utilisation
  des repos offline Red Hat (ISO)

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

# Ansible Core (depuis EPEL ou repo Ansible)
# Option A : depuis EPEL (recommandé)
dnf download --resolve --destdir=/tmp/offline-packages/rpms ansible-core

# Nginx (reverse proxy HTTPS)
dnf download --resolve --destdir=/tmp/offline-packages/rpms nginx

# OpenSSL et outils réseau
dnf download --resolve --destdir=/tmp/offline-packages/rpms openssl openldap-clients

# Outils système supplémentaires
dnf download --resolve --destdir=/tmp/offline-packages/rpms \
  rsync tar gzip curl wget
```

> **Note** : `dnf download --resolve` télécharge le package ET toutes ses dépendances
> transitives. Testez toujours l'installation sur une machine RHEL9 propre avant le
> déploiement réel.

### 3.3 Ansible Semaphore (binaire)

```bash
# Identifier la dernière version stable sur https://github.com/semaphoreui/semaphore/releases
SEMAPHORE_VERSION="2.10.22"

wget -P /tmp/offline-packages/semaphore \
  "https://github.com/semaphoreui/semaphore/releases/download/v${SEMAPHORE_VERSION}/semaphore_${SEMAPHORE_VERSION}_linux_amd64.tar.gz"

# Vérifier le checksum (disponible sur la page releases)
wget -P /tmp/offline-packages/semaphore \
  "https://github.com/semaphoreui/semaphore/releases/download/v${SEMAPHORE_VERSION}/semaphore_${SEMAPHORE_VERSION}_checksums.txt"
sha256sum -c <(grep linux_amd64.tar.gz /tmp/offline-packages/semaphore/semaphore_${SEMAPHORE_VERSION}_checksums.txt)
```

### 3.4 Collections Ansible (si nécessaires)

```bash
# Collection community.general (mail, modules supplémentaires)
ansible-galaxy collection download community.general \
  -p /tmp/offline-packages/ansible-collections

# Collection community.postgresql (si pg_dump Ansible natif souhaité)
ansible-galaxy collection download community.postgresql \
  -p /tmp/offline-packages/ansible-collections
```

### 3.5 Packages Python supplémentaires (si Ansible via pip)

```bash
# Si ansible-core n'est pas disponible via RPM, ou pour avoir une version plus récente
pip download ansible-core --dest /tmp/offline-packages/pip
pip download jinja2 pyyaml cryptography --dest /tmp/offline-packages/pip
```

---

## 4. Transfert vers les serveurs cibles

```bash
# Exemple via SCP (adapter selon votre méthode de transfert)
# Depuis la machine connectée vers le serveur cible

scp -r /tmp/offline-packages/ deploy@git-ansible-hp.company.com:/tmp/
scp -r /tmp/offline-packages/ deploy@semaphore-hp.company.com:/tmp/
scp -r /tmp/offline-packages/ deploy@git-ansible-prod.company.com:/tmp/
scp -r /tmp/offline-packages/ deploy@semaphore-prod.company.com:/tmp/

# Alternative via clé USB (monter la clé et copier)
# cp -r /tmp/offline-packages/ /media/usb/
```

---

## 5. Installation sur les serveurs cibles (en tant que root)

### 5.1 Packages RPM

```bash
# Désactiver tous les dépôts réseau et installer uniquement depuis les fichiers locaux
cd /tmp/offline-packages/rpms
dnf install --disablerepo="*" *.rpm

# Vérifier les installations
git --version
python3 --version
nginx --version
```

### 5.2 Ansible (si installé via pip)

```bash
pip3 install --no-index --find-links=/tmp/offline-packages/pip ansible-core
ansible --version
```

### 5.3 Collections Ansible

```bash
ansible-galaxy collection install \
  /tmp/offline-packages/ansible-collections/community-general-*.tar.gz \
  --offline
```

### 5.4 Ansible Semaphore

```bash
# Extraire le binaire
tar -xzf /tmp/offline-packages/semaphore/semaphore_*_linux_amd64.tar.gz \
  -C /usr/local/bin/ semaphore

chmod +x /usr/local/bin/semaphore
semaphore version
```

---

## 6. Configuration initiale de Semaphore

### 6.1 Créer l'utilisateur système

```bash
useradd --system --no-create-home --shell /bin/false semaphore
mkdir -p /etc/semaphore /var/lib/semaphore
chown semaphore:semaphore /etc/semaphore /var/lib/semaphore
```

### 6.2 Initialiser Semaphore (premier démarrage interactif)

```bash
semaphore setup
```

Répondre aux questions :
- **DB Driver** : `bolt` (BoltDB, base embarquée — recommandé pour simplicité)
- **DB path** : `/var/lib/semaphore/semaphore.bolt`
- **Playbook path** : `/home/semaphore` (dossier de travail Ansible)
- **Public URL** : `https://semaphore-hp.company.com` (ou l'URL HTTPS de votre instance)
- **Admin login / password** : créer le compte administrateur initial

> **Alternative BDD** : Pour une installation enterprise avec haute disponibilité,
> utiliser PostgreSQL à la place de BoltDB.
> Dans ce cas, `dnf install` le RPM postgresql offline et configurer en conséquence.

### 6.3 Fichier d'environnement

Créer `/etc/semaphore/semaphore.env` (voir `docs/ldap-https-configuration.md` pour le contenu complet) :

```bash
touch /etc/semaphore/semaphore.env
chmod 640 /etc/semaphore/semaphore.env
chown root:semaphore /etc/semaphore/semaphore.env

# Ajouter au minimum :
echo "SEMAPHORE_WEB_HOST=https://semaphore-hp.company.com" >> /etc/semaphore/semaphore.env
```

### 6.4 Unité systemd

Créer `/etc/systemd/system/semaphore.service` :

```ini
[Unit]
Description=Ansible Semaphore
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

## 7. Configuration Nginx (HTTPS)

Voir le guide complet dans `docs/ldap-https-configuration.md` (section 3).

```bash
# Résumé rapide
cp /etc/nginx/conf.d/semaphore.conf  # (créer selon le guide LDAP/HTTPS)
nginx -t
systemctl enable nginx
systemctl start nginx
```

---

## 8. Considérations spécifiques RHEL9

### 8.1 SELinux (enforcing par défaut)

```bash
# Pour Nginx → Semaphore (proxy réseau)
setsebool -P httpd_can_network_connect 1

# Restaurer les contextes SELinux du binaire Semaphore
restorecon -v /usr/local/bin/semaphore

# Vérifier les refus SELinux si problème
ausearch -m avc -ts recent | audit2allow -a
```

### 8.2 Firewalld

```bash
# Ouvrir SSH (si pas déjà ouvert)
firewall-cmd --permanent --add-service=ssh

# Ouvrir HTTPS pour l'interface Semaphore
firewall-cmd --permanent --add-service=https

# Appliquer
firewall-cmd --reload
firewall-cmd --list-all
```

### 8.3 Vérification des dépendances RPM

Les RPMs RHEL9 peuvent avoir des dépendances sur des bibliothèques système spécifiques.
Pour s'assurer que toutes les dépendances sont incluses dans le téléchargement :

```bash
# Sur la machine connectée, vérifier que le download inclut TOUTES les deps
dnf download --resolve --destdir=/tmp/test-deps ansible-core
rpm -qpR /tmp/test-deps/*.rpm | sort -u > /tmp/deps-requises.txt
# Vérifier manuellement que tout est présent
```

### 8.4 Gestion des mises à jour offline

Pour mettre à jour Semaphore :
1. Télécharger le nouveau binaire sur une machine connectée
2. Transférer vers le serveur
3. `systemctl stop semaphore`
4. Remplacer le binaire : `cp semaphore_new /usr/local/bin/semaphore`
5. `systemctl start semaphore`

---

## 9. Vérification finale

```bash
# Vérifier toutes les installations
git --version          # → git version 2.x.x
ansible --version      # → ansible [core 2.x.x]
semaphore version      # → 2.x.x
nginx -v               # → nginx version: nginx/1.x.x

# Vérifier les services
systemctl status semaphore
systemctl status nginx

# Tester l'accès HTTPS
curl -Ik https://semaphore-hp.company.com
# Attendu : HTTP/2 200 avec header Strict-Transport-Security
```

---

## 10. Langages supplémentaires (Java, Node.js, Go)

Pour les projets multi-langages, des runtimes supplémentaires doivent être installés
sur les **serveurs applicatifs** (App HP, App Prod) — pas sur les serveurs Semaphore.

### Java

```bash
# Télécharger (machine connectée)
dnf download --resolve --destdir=/tmp/offline-packages/rpms java-17-openjdk java-17-openjdk-devel

# Pour Maven
dnf download --resolve --destdir=/tmp/offline-packages/rpms maven

# Installer (serveurs App)
dnf install --disablerepo="*" /tmp/offline-packages/rpms/*.rpm
java -version
mvn --version
```

### Node.js

```bash
# Télécharger (machine connectée)
dnf download --resolve --destdir=/tmp/offline-packages/rpms nodejs npm

# Installer
dnf install --disablerepo="*" /tmp/offline-packages/rpms/*.rpm
node --version
npm --version
```

> **Mode offline npm** : Pour `npm ci --prefer-offline`, le cache npm doit être
> pré-chargé sur le serveur. Exécuter `npm install` sur une machine connectée pour
> peupler `~/.npm`, puis transférer ce répertoire vers le serveur cible.
> Alternative recommandée : déployer **Verdaccio** (registre npm privé, open source)
> sur un serveur interne pour servir les packages Node.js.

### Go

```bash
# Télécharger le tarball Go (machine connectée)
GO_VERSION="1.22.0"
wget https://go.dev/dl/go${GO_VERSION}.linux-amd64.tar.gz

# Installer (serveurs App)
tar -C /usr/local -xzf go${GO_VERSION}.linux-amd64.tar.gz
echo 'export PATH=$PATH:/usr/local/go/bin' >> /etc/profile.d/go.sh
source /etc/profile.d/go.sh
go version

# CRITIQUE pour mode offline : pré-charger le vendor/
# Sur machine connectée (dans le répertoire du projet Go) :
go mod vendor
# Committer le répertoire vendor/ dans le dépôt avant de déployer
```
