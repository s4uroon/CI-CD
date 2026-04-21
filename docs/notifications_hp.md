# Notifications et alertes — Zone Hors-Prod (HP)

> Document dédié à la zone HP. Pour la zone Production, voir [notifications_prod.md](./notifications_prod.md).

Ce guide décrit les options de notification pour les déploiements HP.

---

## 1. Notifications natives Ansible Semaphore HP

Semaphore >= 2.9 intègre un système d'alertes natif, déclenché automatiquement
sur succès ou échec de n'importe quel template, sans modifier les playbooks.

### 1.1 Configuration email (SMTP) sur Semaphore HP

Ajouter dans `/etc/semaphore/semaphore.env` sur **Semaphore HP** :

```bash
# ─── Email ─────────────────────────────────────────────────────────────────────
SEMAPHORE_EMAIL_SENDER=semaphore-hp@company.com
SEMAPHORE_EMAIL_HOST=smtp.company.com
SEMAPHORE_EMAIL_PORT=587
SEMAPHORE_EMAIL_USERNAME=semaphore-hp@company.com
SEMAPHORE_EMAIL_PASSWORD=<mot_de_passe_smtp>

# Si le serveur SMTP ne requiert pas d'authentification (réseau interne)
# SEMAPHORE_EMAIL_USERNAME=
# SEMAPHORE_EMAIL_PASSWORD=
```

Après modification, redémarrer Semaphore HP :
```bash
systemctl restart semaphore
```

### 1.2 Configuration Slack (HP)

```bash
SEMAPHORE_SLACK_ALERT=yes
SEMAPHORE_SLACK_URL=https://hooks.slack.com/services/T.../B.../...
```

### 1.3 Configuration Telegram (HP)

```bash
SEMAPHORE_TELEGRAM_ALERT=yes
SEMAPHORE_TELEGRAM_CHAT=-1001234567890   # ID du canal/groupe HP
SEMAPHORE_TELEGRAM_TOKEN=...             # Token du bot Telegram
```

### 1.4 Activer les alertes par template HP

Dans Semaphore HP UI → **Task Templates** → éditer un template :
- Activer l'option **"Alert"**
- Sélectionner les destinataires / canaux configurés

Les alertes natives se déclenchent sur :
- **Échec** du template (toujours)
- **Succès** du template (configurable)

---

## 2. Notifications dans les playbooks HP

Pour des notifications plus personnalisées (détail du commit, nom du serveur).

### 2.1 Où insérer les notifications dans les playbooks HP

Placer les notifications dans le **Play 3** (résumé global sur `git_ansible_hp`)
dans `hp/02_deploy_hp.yml`. Cela garantit **une seule notification par exécution** :

```yaml
# ─── Play 3 : Résumé global ────────────────────────────────────────────────────
- name: "[HP] Résumé global du déploiement"
  hosts: git_ansible_hp
  gather_facts: false

  tasks:
    - name: "Résumé final"
      ansible.builtin.debug:
        msg: [...]

    # ← Ajouter les notifications ici
    - name: "Notification email — déploiement HP réussi"
      community.general.mail:
        ...
      when: notify_email | default(false)
```

### 2.2 Notification email via module Ansible (HP)

```yaml
- name: "Notification email — déploiement HP réussi"
  community.general.mail:
    host: "{{ smtp_host }}"
    port: "{{ smtp_port | default(587) }}"
    username: "{{ smtp_user | default(omit) }}"
    password: "{{ smtp_password | default(omit) }}"
    from: "semaphore-hp@company.com"
    to: "{{ notify_email_recipients | default('equipe-ops@company.com') }}"
    subject: "[HP] Déploiement {{ repo_name }} — SUCCÈS"
    body: |
      Déploiement HP réussi

      Environnement : HP
      Dépôt         : {{ repo_name }}
      Date          : {{ ansible_date_time.iso8601 }}
  delegate_to: localhost
  when: notify_email | default(false)
  failed_when: false
```

Pour les **échecs HP**, utiliser un bloc `rescue` dans le Play 2 :

```yaml
rescue:
  - name: "Notification email — déploiement HP ECHOUE"
    community.general.mail:
      subject: "[HP] Déploiement {{ repo_name }} — ECHEC"
      body: |
        ECHEC du déploiement HP sur {{ inventory_hostname }}
        Dépôt     : {{ repo_name }}
        Date      : {{ ansible_date_time.iso8601 | default('N/A') }}
        Consultez les logs Semaphore HP pour le détail.
      ...
    delegate_to: localhost
    failed_when: false
```

### 2.3 Webhook Teams / Slack (HP)

```yaml
- name: "Notification Teams — déploiement HP réussi"
  ansible.builtin.uri:
    url: "{{ teams_webhook_url }}"
    method: POST
    body_format: json
    body:
      "@type": "MessageCard"
      "@context": "http://schema.org/extensions"
      themeColor: "00FF00"
      summary: "Déploiement HP {{ repo_name }} réussi"
      sections:
        - activityTitle: "[HP] Déploiement {{ repo_name }} — SUCCÈS"
          facts:
            - name: "Dépôt"
              value: "{{ repo_name }}"
            - name: "Date"
              value: "{{ ansible_date_time.iso8601 }}"
    status_code: [200, 201, 202]
  when: teams_webhook_url is defined
  failed_when: false

- name: "Notification Slack — déploiement HP réussi"
  ansible.builtin.uri:
    url: "{{ slack_webhook_url }}"
    method: POST
    body_format: json
    body:
      text: ":white_check_mark: *[HP]* Déploiement `{{ repo_name }}` — SUCCÈS ({{ ansible_date_time.date }})"
    status_code: 200
  when: slack_webhook_url is defined
  failed_when: false
```

---

## 3. Variables de configuration HP

Ajouter dans `ansible/inventory/hp/group_vars/all.yml` :

```yaml
# ─── Notifications HP ──────────────────────────────────────────────────────────

# Email (désactivé par défaut — activer en définissant notify_email: true)
notify_email: false
smtp_host: smtp.company.com
smtp_port: 587
# smtp_user: semaphore-hp@company.com    # décommenter si authentification SMTP
# smtp_password: "..."                   # utiliser Vault Semaphore pour ce secret
notify_email_recipients: equipe-ops@company.com

# Webhook Teams HP (décommenter et renseigner l'URL si disponible)
# teams_webhook_url: "https://company.webhook.office.com/webhookb2/..."

# Webhook Slack HP (décommenter et renseigner l'URL si disponible)
# slack_webhook_url: "https://hooks.slack.com/services/..."
```

---

## 4. Gestion des secrets de notification

Les URLs de webhooks et mots de passe SMTP sont des secrets.
Ne **jamais** les committer en clair dans les group_vars.

**Options recommandées** :

1. **Semaphore Key Store** : stocker les secrets dans Semaphore HP et les passer
   en Extra Variables lors de l'exécution du template

2. **Ansible Vault** : chiffrer les fichiers group_vars contenant les secrets
   ```bash
   ansible-vault encrypt ansible/inventory/hp/group_vars/secrets.yml
   ```
   Et ajouter le mot de passe Vault dans le Key Store Semaphore HP.

3. **Variables d'environnement** : définir dans `/etc/semaphore/semaphore.env`
   et référencer via `lookup('env', 'SMTP_PASSWORD')`

---

## 5. Résumé des options

| Option                          | Avantages                                | Inconvénients                          |
|---------------------------------|------------------------------------------|----------------------------------------|
| Alertes natives Semaphore HP    | Zéro modification playbook, simple       | Peu de personnalisation du message     |
| Module `community.general.mail` | Message très personnalisé                | Collection à installer offline         |
| Webhook via `uri` module        | Aucune dépendance, Teams/Slack natif     | Message moins riche que l'email        |
| SMTP Semaphore HP natif         | Intégré, aucun playbook à modifier       | Format de message fixe                 |

**Recommandation** : Activer les **alertes natives Semaphore HP** pour la surveillance
de base, et ajouter un **webhook Teams/Slack** dans les playbooks pour les notifications
opérationnelles détaillées.
