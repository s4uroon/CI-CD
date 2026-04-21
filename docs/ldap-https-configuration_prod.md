# Configuration LDAP et HTTPS — Semaphore PROD

> Document dédié à la zone Production. Pour la zone Hors-Prod, voir [ldap-https-configuration_hp.md](./ldap-https-configuration_hp.md).

Ce guide couvre la configuration de l'authentification LDAP et de l'accès HTTPS
pour l'instance Semaphore PROD (`semaphore-prod.company.com`).

---

## 1. Configuration LDAP

Ansible Semaphore supporte nativement l'authentification LDAP (Active Directory ou OpenLDAP).
Les comptes locaux restent disponibles en parallèle comme fallback.

### 1.1 Méthode A — Variables d'environnement (recommandée)

Créer le fichier `/etc/semaphore/semaphore.env` sur **Semaphore PROD** :

```bash
# ─── LDAP ─────────────────────────────────────────────────────────────────────
SEMAPHORE_LDAP_ACTIVATED=yes
SEMAPHORE_LDAP_HOST=ldap.company.com
SEMAPHORE_LDAP_PORT=389

# TLS vers le serveur LDAP (recommandé en production)
SEMAPHORE_LDAP_NEEDTLS=yes

# Compte de service pour les recherches LDAP (lecture seule)
SEMAPHORE_LDAP_DN_BIND="cn=semaphore-reader,dc=company,dc=com"
SEMAPHORE_LDAP_PASSWORD=<mot_de_passe_compte_service>

# Base de recherche
SEMAPHORE_LDAP_DN_SEARCH="dc=company,dc=com"

# Filtre de recherche :
#   Active Directory  : (sAMAccountName=%s)
#   OpenLDAP          : (uid=%s)
SEMAPHORE_LDAP_SEARCH_FILTER="(sAMAccountName=%s)"

# Correspondance des attributs LDAP → Semaphore
SEMAPHORE_LDAP_MAPPING_DN="dn"
SEMAPHORE_LDAP_MAPPING_USERNAME="sAMAccountName"
SEMAPHORE_LDAP_MAPPING_FULLNAME="cn"
SEMAPHORE_LDAP_MAPPING_EMAIL="mail"

# URL publique de cette instance PROD
SEMAPHORE_WEB_HOST=https://semaphore-prod.company.com
```

Protéger le fichier (contient un mot de passe) :
```bash
chmod 640 /etc/semaphore/semaphore.env
chown root:semaphore /etc/semaphore/semaphore.env
```

Référencer ce fichier dans l'unité systemd :
```ini
[Service]
EnvironmentFile=/etc/semaphore/semaphore.env
```

---

### 1.2 Méthode B — Fichier config.yml

Si Semaphore est configuré via `config.yml` :

```yaml
ldap:
  activated: true
  host: ldap.company.com
  port: 389
  needtls: true
  dn_bind: "cn=semaphore-reader,dc=company,dc=com"
  password: "<mot_de_passe>"
  dn_search: "dc=company,dc=com"
  search_filter: "(sAMAccountName=%s)"
  mapping:
    dn: dn
    username: sAMAccountName
    fullname: cn
    email: mail
```

---

### 1.3 Configuration OpenLDAP (alternative à Active Directory)

```bash
SEMAPHORE_LDAP_SEARCH_FILTER="(uid=%s)"
SEMAPHORE_LDAP_MAPPING_USERNAME="uid"
SEMAPHORE_LDAP_MAPPING_FULLNAME="displayName"
SEMAPHORE_LDAP_MAPPING_EMAIL="mail"
```

---

### 1.4 Vérification LDAP

Tester la connexion depuis **Semaphore PROD** avant d'activer :

```bash
# Test de connexion LDAP (paquet openldap-clients)
ldapsearch -H ldap://ldap.company.com:389 \
  -D "cn=semaphore-reader,dc=company,dc=com" \
  -w "<mot_de_passe>" \
  -b "dc=company,dc=com" \
  "(sAMAccountName=<votre_login>)" cn mail

# Test avec TLS (LDAPS)
ldapsearch -H ldaps://ldap.company.com:636 \
  -D "cn=semaphore-reader,dc=company,dc=com" \
  -w "<mot_de_passe>" \
  -b "dc=company,dc=com" \
  "(sAMAccountName=<votre_login>)" cn mail
```

---

## 2. Comptes locaux Semaphore PROD (fallback LDAP)

Même avec LDAP activé, des comptes locaux doivent exister. Minimum requis : **10 comptes**.
L'accès à Semaphore PROD doit être plus restreint que HP (principe du moindre privilège).

### 2.1 Création via CLI Semaphore

```bash
# Créer un compte admin local
semaphore user add \
  --admin \
  --login "admin-prod" \
  --name "Administrateur PROD" \
  --email "admin-prod@company.com" \
  --password "MotDePasseForte123!"
```

### 2.2 Rôles disponibles dans Semaphore

| Rôle          | Droits                                              |
|---------------|-----------------------------------------------------|
| `admin`       | Accès complet (config, templates, users)            |
| `manager`     | Créer/modifier templates et inventaires             |
| `task_runner` | Lancer des tâches uniquement                        |
| `viewer`      | Consulter les logs et l'historique uniquement       |

### 2.3 Comptes locaux recommandés pour Semaphore PROD

```bash
# Administrateurs PROD (2)
semaphore user add --admin --login "admin-prod1" --name "Admin PROD 1" --email "admin-prod1@company.com" --password "..."
semaphore user add --admin --login "admin-prod2" --name "Admin PROD 2" --email "admin-prod2@company.com" --password "..."

# Opérateurs PROD (2 — task_runner uniquement, pas manager)
semaphore user add --login "ops-prod1" --name "Ops PROD 1" --email "ops-prod1@company.com" --password "..."
semaphore user add --login "ops-prod2" --name "Ops PROD 2" --email "ops-prod2@company.com" --password "..."

# Auditeurs (2 — lecture seule uniquement)
semaphore user add --login "audit-prod1" --name "Auditeur PROD 1" --email "audit-prod1@company.com" --password "..."
semaphore user add --login "audit-prod2" --name "Auditeur PROD 2" --email "audit-prod2@company.com" --password "..."

# Compte de supervision NOC (2 — viewer uniquement)
semaphore user add --login "noc1" --name "NOC 1" --email "noc1@company.com" --password "..."
semaphore user add --login "noc2" --name "NOC 2" --email "noc2@company.com" --password "..."

# Comptes de secours (2 — admin, à conserver sous enveloppe scellée)
semaphore user add --admin --login "break-glass-prod1" --name "Break Glass PROD 1" --email "breakglass-prod1@company.com" --password "..."
semaphore user add --admin --login "break-glass-prod2" --name "Break Glass PROD 2" --email "breakglass-prod2@company.com" --password "..."
```

### 2.4 Modification du mot de passe d'un compte local

```bash
semaphore user change-by-login \
  --login "ops-prod1" \
  --password "NouveauMotDePasse789!"
```

---

## 3. Configuration HTTPS — Nginx reverse proxy (PROD)

Semaphore écoute sur `127.0.0.1:3000` par défaut (HTTP). Nginx assure le reverse proxy HTTPS.

> **Important** : Les headers `Upgrade` et `Connection` sont **obligatoires** pour que
> le streaming des logs en temps réel (WebSocket) fonctionne dans le navigateur.

### 3.1 Fichier de configuration Nginx pour Semaphore PROD

Créer `/etc/nginx/conf.d/semaphore.conf` sur **Semaphore PROD** :

```nginx
# Redirection HTTP → HTTPS
server {
    listen 80;
    server_name semaphore-prod.company.com;
    return 301 https://$host$request_uri;
}

# Serveur HTTPS
server {
    listen 443 ssl;
    server_name semaphore-prod.company.com;

    # Certificats TLS
    ssl_certificate     /etc/pki/tls/certs/semaphore.crt;
    ssl_certificate_key /etc/pki/tls/private/semaphore.key;

    # Protocoles et ciphers sécurisés
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5:!RC4;
    ssl_prefer_server_ciphers on;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;

    # Headers de sécurité
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Frame-Options SAMEORIGIN always;
    add_header X-Content-Type-Options nosniff always;

    # Proxy vers Semaphore PROD
    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;

        # OBLIGATOIRE pour WebSocket (logs en temps réel)
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Timeout pour les longues tâches Ansible
        proxy_read_timeout 3600;
        proxy_send_timeout 3600;
    }
}
```

### 3.2 Certificats TLS pour Semaphore PROD

**Option A — Certificat signé par une CA interne (recommandé PROD)** :
```bash
cp semaphore.crt /etc/pki/tls/certs/semaphore.crt
cp semaphore.key /etc/pki/tls/private/semaphore.key
chmod 600 /etc/pki/tls/private/semaphore.key

# Ajouter la CA interne au trust store RHEL9
cp ca-interne.crt /etc/pki/ca-trust/source/anchors/
update-ca-trust
```

**Option B — Certificat auto-signé (à éviter en PROD, acceptable pour environnement isolé)** :
```bash
openssl req -x509 -newkey rsa:4096 -days 3650 -nodes \
  -keyout /etc/pki/tls/private/semaphore.key \
  -out /etc/pki/tls/certs/semaphore.crt \
  -subj "/C=FR/O=Company/CN=semaphore-prod.company.com"
chmod 600 /etc/pki/tls/private/semaphore.key
```

### 3.3 SELinux (RHEL9)

```bash
# Autoriser Nginx à se connecter en réseau (proxy vers Semaphore PROD)
setsebool -P httpd_can_network_connect 1

# Vérifier le contexte SELinux du binaire semaphore
restorecon -v /usr/local/bin/semaphore
```

### 3.4 Firewalld

```bash
# Ouvrir le port HTTPS pour les accès à Semaphore PROD
firewall-cmd --permanent --add-service=https
firewall-cmd --reload

# Vérifier
firewall-cmd --list-services
```

### 3.5 Activer et démarrer Nginx

```bash
systemctl enable nginx
systemctl start nginx
systemctl status nginx
```

---

## 4. Vérification complète — Semaphore PROD

```bash
# 1. Vérifier que Nginx démarre sans erreur
nginx -t

# 2. Tester la redirection HTTP → HTTPS
curl -I http://semaphore-prod.company.com
# Attendu : HTTP/1.1 301 Moved Permanently

# 3. Tester l'accès HTTPS
curl -I https://semaphore-prod.company.com
# Attendu : HTTP/1.1 200 OK avec Strict-Transport-Security

# 4. Tester le WebSocket (logs en temps réel)
# Ouvrir un navigateur → https://semaphore-prod.company.com
# Lancer un template → les logs doivent s'afficher en temps réel dans l'UI

# 5. Vérifier l'authentification LDAP
# Se connecter avec un compte LDAP sur https://semaphore-prod.company.com
# Si échec, vérifier les logs Semaphore :
journalctl -u semaphore -n 50
```
