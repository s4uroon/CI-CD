# Notifications et alertes CI/CD

Ce guide décrit les options de notification disponibles pour les déploiements
HP et Prod, et comment les configurer.

---

## 1. Notifications natives Ansible Semaphore

Semaphore >= 2.9 intègre un système d'alertes natif, déclenché automatiquement
sur succès ou échec de n'importe quel template, sans modifier les playbooks.

### 1.1 Configuration email (SMTP)

Définir dans `/etc/semaphore/semaphore.env` :

```bash
# ─── Email ─────────────────────────────────────────────────────────────────────
SEMAPHORE_EMAIL_SENDER=semaphore@company.com
SEMAPHORE_EMAIL_HOST=smtp.company.com
SEMAPHORE_EMAIL_PORT=587
SEMAPHORE_EMAIL_USERNAME=semaphore@company.com
SEMAPHORE_EMAIL_PASSWORD=<mot_de_passe_smtp>

# Si le serveur SMTP ne requiert pas d'authentification (réseau interne)
# SEMAPHORE_EMAIL_USERNAME=
# SEMAPHORE_EMAIL_PASSWORD=
```

Après modification, redémarrer Semaphore :
```bash
systemctl restart semaphore
```

### 1.2 Configuration Slack

```bash
SEMAPHORE_SLACK_ALERT=yes
SEMAPHORE_SLACK_URL=https://hooks.slack.com/services/T.../B.../...
```

### 1.3 Configuration Telegram

```bash
SEMAPHORE_TELEGRAM_ALERT=yes
SEMAPHORE_TELEGRAM_CHAT=-1001234567890   # ID du canal/groupe
SEMAPHORE_TELEGRAM_TOKEN=...             # Token du bot Telegram
```

### 1.4 Configuration Microsoft Teams (webhook entrant)

Semaphore ne supporte pas Teams nativement, mais la section 3 décrit
comment envoyer vers Teams via un webhook dans les playbooks Ansible.

### 1.5 Activer les alertes par template

Dans Semaphore UI → **Task Templates** → éditer un template :
- Activer l'option **"Alert"** (ou **"Alerts"**)
- Sélectionner les destinataires / canaux configurés

Les alertes natives se déclenchent sur :
- **Échec** du template (toujours)
- **Succès** du template (configurable)

---

## 2. Notifications dans les playbooks Ansible

Pour des notifications plus personnalisées (détail du commit, nom du serveur,
environnement), ajouter des tâches de notification directement dans les playbooks.

### 2.1 Où insérer les notifications

Les notifications doivent être placées dans le **Play 3** (résumé global sur
`git_ansible_*`), pas dans le Play 2 (déploiement serveur par serveur).
Cela garantit **une seule notification par exécution**, quel que soit le nombre
de serveurs.

Exemple de structure dans `02_deploy_hp.yml` :

```yaml
# ─── Play 3 : Résumé global ────────────────────────────────────────────────────
- name: "[HP] Résumé global du déploiement"
  hosts: git_ansible_hp
  gather_facts: false

  tasks:
    - name: "Résumé final"
      ansible.builtin.debug:
        msg: [...]

    # ← Ajouter les notifications ici (post_tasks ou tasks supplémentaires)
    - name: "Notification email — déploiement HP réussi"
      community.general.mail:
        ...
      when: notify_email | default(false)
```

### 2.2 Notification email via module Ansible

Nécessite la collection `community.general` (voir `offline-installation-rhel9.md`
pour l'installation offline).

```yaml
- name: "Notification email — déploiement réussi"
  community.general.mail:
    host: "{{ smtp_host }}"
    port: "{{ smtp_port | default(587) }}"
    username: "{{ smtp_user | default(omit) }}"
    password: "{{ smtp_password | default(omit) }}"
    from: "semaphore@company.com"
    to: "{{ notify_email_recipients | default('equipe-ops@company.com') }}"
    subject: "[{{ deploy_env | upper | default('HP') }}] Déploiement {{ repo_name }} — SUCCÈS"
    body: |
      Déploiement réussi

      Environnement : {{ deploy_env | upper | default('HP') }}
      Dépôt         : {{ repo_name }}
      Date          : {{ ansible_date_time.iso8601 }}
  delegate_to: localhost
  when: notify_email | default(false)
  failed_when: false   # Ne jamais bloquer un déploiement pour une notification
```

Pour les **échecs**, utiliser un bloc `rescue` dans le Play 2 :

```yaml
rescue:
  - name: "Notification email — déploiement ECHOUE"
    community.general.mail:
      subject: "[{{ deploy_env | upper }}] Déploiement {{ repo_name }} — ECHEC"
      body: |
        ECHEC du déploiement sur {{ inventory_hostname }}
        Dépôt     : {{ repo_name }}
        Date      : {{ ansible_date_time.iso8601 | default('N/A') }}
        Consultez les logs Semaphore pour le détail.
      ...
    delegate_to: localhost
    failed_when: false
```

### 2.3 Webhook Teams / Slack (via uri module)

Ne nécessite aucune collection supplémentaire.

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
      summary: "Déploiement {{ repo_name }} réussi"
      sections:
        - activityTitle: "[HP] Déploiement {{ repo_name }} — SUCCÈS"
          facts:
            - name: "Dépôt"
              value: "{{ repo_name }}"
            - name: "Date"
              value: "{{ ansible_date_time.iso8601 }}"
    status_code: [200, 201, 202]
  when: teams_webhook_url is defined
  failed_when: false   # Ne jamais bloquer pour une notification

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

## 3. Variables de configuration par environnement

Ajouter dans `ansible/inventory/hp/group_vars/all.yml` :

```yaml
# ─── Notifications ─────────────────────────────────────────────────────────────

# Email (désactivé par défaut — activer en définissant notify_email: true)
notify_email: false
smtp_host: smtp.company.com
smtp_port: 587
# smtp_user: semaphore@company.com      # décommenter si authentification SMTP
# smtp_password: "..."                  # utiliser Vault Semaphore pour ce secret
notify_email_recipients: equipe-ops@company.com

# Webhook Teams (décommenter et renseigner l'URL si disponible)
# teams_webhook_url: "https://company.webhook.office.com/webhookb2/..."

# Webhook Slack (décommenter et renseigner l'URL si disponible)
# slack_webhook_url: "https://hooks.slack.com/services/..."
```

Ajouter les mêmes variables dans `ansible/inventory/prod/group_vars/all.yml`
en adaptant les destinataires (ex: `equipe-prod@company.com`).

---

## 4. Gestion des secrets de notification

Les URLs de webhooks et mots de passe SMTP sont des secrets.
Ne **jamais** les committer en clair dans les group_vars.

**Options recommandées** :

1. **Semaphore Key Store** (Variables) : stocker les secrets dans Semaphore
   et les passer en Extra Variables lors de l'exécution du template

2. **Ansible Vault** : chiffrer les fichiers group_vars contenant les secrets
   ```bash
   ansible-vault encrypt ansible/inventory/hp/group_vars/secrets.yml
   ```
   Et ajouter le mot de passe Vault dans le Key Store Semaphore.

3. **Variables d'environnement sur le control node** : définir dans
   `/etc/semaphore/semaphore.env` et référencer via `lookup('env', 'SMTP_PASSWORD')`

---

## 5. Résumé des options

| Option                      | Avantages                                | Inconvénients                          |
|-----------------------------|------------------------------------------|----------------------------------------|
| Alertes natives Semaphore   | Zéro modification playbook, simple       | Peu de personnalisation du message     |
| Module `community.general.mail` | Message très personnalisé            | Collection à installer offline         |
| Webhook via `uri` module    | Aucune dépendance, Teams/Slack natif     | Message moins riche que l'email        |
| SMTP Semaphore natif        | Intégré, aucun playbook à modifier       | Format de message fixe                 |

**Recommandation** : Activer les **alertes natives Semaphore** pour la surveillance
de base (échec/succès), et ajouter un **webhook Teams/Slack** dans les playbooks
pour les notifications opérationnelles détaillées.
