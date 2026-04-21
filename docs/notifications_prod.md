# Notifications et alertes — Zone Production (PROD)

> Document dédié à la zone Production. Pour la zone Hors-Prod, voir [notifications_hp.md](./notifications_hp.md).

Ce guide décrit les options de notification pour les déploiements PROD.

---

## 1. Notifications natives Ansible Semaphore PROD

Semaphore >= 2.9 intègre un système d'alertes natif, déclenché automatiquement
sur succès ou échec de n'importe quel template, sans modifier les playbooks.

### 1.1 Configuration email (SMTP) sur Semaphore PROD

Ajouter dans `/etc/semaphore/semaphore.env` sur **Semaphore PROD** :

```bash
# ─── Email ─────────────────────────────────────────────────────────────────────
SEMAPHORE_EMAIL_SENDER=semaphore-prod@company.com
SEMAPHORE_EMAIL_HOST=smtp.company.com
SEMAPHORE_EMAIL_PORT=587
SEMAPHORE_EMAIL_USERNAME=semaphore-prod@company.com
SEMAPHORE_EMAIL_PASSWORD=<mot_de_passe_smtp>

# Si le serveur SMTP ne requiert pas d'authentification (réseau interne)
# SEMAPHORE_EMAIL_USERNAME=
# SEMAPHORE_EMAIL_PASSWORD=
```

Après modification, redémarrer Semaphore PROD :
```bash
systemctl restart semaphore
```

### 1.2 Configuration Slack (PROD)

```bash
SEMAPHORE_SLACK_ALERT=yes
SEMAPHORE_SLACK_URL=https://hooks.slack.com/services/T.../B.../...
```

### 1.3 Configuration Telegram (PROD)

```bash
SEMAPHORE_TELEGRAM_ALERT=yes
SEMAPHORE_TELEGRAM_CHAT=-1009876543210   # ID du canal/groupe PROD (différent de HP)
SEMAPHORE_TELEGRAM_TOKEN=...             # Token du bot Telegram
```

### 1.4 Activer les alertes par template PROD

Dans Semaphore PROD UI → **Task Templates** → éditer un template :
- Activer l'option **"Alert"**
- Sélectionner les destinataires / canaux configurés

Les alertes natives se déclenchent sur :
- **Échec** du template (toujours)
- **Succès** du template (configurable)

---

## 2. Notifications dans les playbooks PROD

### 2.1 Où insérer les notifications dans les playbooks PROD

Placer les notifications dans le **Play 3** (résumé global sur `git_ansible_prod`)
dans `prod/02_deploy_prod.yml`. Cela garantit **une seule notification par exécution** :

```yaml
# ─── Play 3 : Résumé global ────────────────────────────────────────────────────
- name: "[PROD] Résumé global du déploiement"
  hosts: git_ansible_prod
  gather_facts: false

  tasks:
    - name: "Résumé final"
      ansible.builtin.debug:
        msg: [...]

    # ← Ajouter les notifications ici
    - name: "Notification email — déploiement PROD réussi"
      community.general.mail:
        ...
      when: notify_email | default(false)
```

### 2.2 Notification email via module Ansible (PROD)

```yaml
- name: "Notification email — déploiement PROD réussi"
  community.general.mail:
    host: "{{ smtp_host }}"
    port: "{{ smtp_port | default(587) }}"
    username: "{{ smtp_user | default(omit) }}"
    password: "{{ smtp_password | default(omit) }}"
    from: "semaphore-prod@company.com"
    to: "{{ notify_email_recipients | default('equipe-prod@company.com') }}"
    subject: "[PROD] Déploiement {{ repo_name }} — SUCCÈS"
    body: |
      Déploiement PRODUCTION réussi

      Environnement : PROD
      Dépôt         : {{ repo_name }}
      Date          : {{ ansible_date_time.iso8601 }}
  delegate_to: localhost
  when: notify_email | default(false)
  failed_when: false
```

Pour les **échecs PROD**, la notification doit également être envoyée à l'équipe d'astreinte :

```yaml
rescue:
  - name: "Notification email — déploiement PROD ECHOUE"
    community.general.mail:
      subject: "[PROD] ⚠️ Déploiement {{ repo_name }} — ECHEC"
      to: "{{ notify_email_recipients_prod_critical | default('astreinte@company.com') }}"
      body: |
        ECHEC CRITIQUE du déploiement PRODUCTION sur {{ inventory_hostname }}
        Dépôt     : {{ repo_name }}
        Date      : {{ ansible_date_time.iso8601 | default('N/A') }}
        Consultez les logs Semaphore PROD pour le détail.
        Pensez à vérifier si une sauvegarde BDD a été réalisée.
      ...
    delegate_to: localhost
    failed_when: false
```

### 2.3 Webhook Teams / Slack (PROD)

```yaml
- name: "Notification Teams — déploiement PROD réussi"
  ansible.builtin.uri:
    url: "{{ teams_webhook_url }}"
    method: POST
    body_format: json
    body:
      "@type": "MessageCard"
      "@context": "http://schema.org/extensions"
      themeColor: "FF8C00"
      summary: "Déploiement PROD {{ repo_name }} réussi"
      sections:
        - activityTitle: "[PROD] Déploiement {{ repo_name }} — SUCCÈS"
          facts:
            - name: "Dépôt"
              value: "{{ repo_name }}"
            - name: "Date"
              value: "{{ ansible_date_time.iso8601 }}"
    status_code: [200, 201, 202]
  when: teams_webhook_url is defined
  failed_when: false

- name: "Notification Slack — déploiement PROD réussi"
  ansible.builtin.uri:
    url: "{{ slack_webhook_url }}"
    method: POST
    body_format: json
    body:
      text: ":rocket: *[PROD]* Déploiement `{{ repo_name }}` — SUCCÈS ({{ ansible_date_time.date }})"
    status_code: 200
  when: slack_webhook_url is defined
  failed_when: false
```

---

## 3. Variables de configuration PROD

Ajouter dans `ansible/inventory/prod/group_vars/all.yml` :

```yaml
# ─── Notifications PROD ────────────────────────────────────────────────────────

# Email (désactivé par défaut — activer en définissant notify_email: true)
notify_email: false
smtp_host: smtp.company.com
smtp_port: 587
# smtp_user: semaphore-prod@company.com  # décommenter si authentification SMTP
# smtp_password: "..."                   # utiliser Vault Semaphore pour ce secret
notify_email_recipients: equipe-prod@company.com
notify_email_recipients_prod_critical: astreinte@company.com

# Webhook Teams PROD (décommenter et renseigner l'URL si disponible)
# teams_webhook_url: "https://company.webhook.office.com/webhookb2/..."

# Webhook Slack PROD (décommenter et renseigner l'URL si disponible)
# slack_webhook_url: "https://hooks.slack.com/services/..."
```

> Les destinataires PROD (`equipe-prod@company.com`, `astreinte@company.com`)
> doivent être distincts de ceux de HP pour éviter les alertes croisées.

---

## 4. Gestion des secrets de notification

Les URLs de webhooks et mots de passe SMTP sont des secrets.
Ne **jamais** les committer en clair dans les group_vars.

**Options recommandées** :

1. **Semaphore Key Store** : stocker les secrets dans Semaphore PROD et les passer
   en Extra Variables lors de l'exécution du template

2. **Ansible Vault** : chiffrer les fichiers group_vars contenant les secrets
   ```bash
   ansible-vault encrypt ansible/inventory/prod/group_vars/secrets.yml
   ```
   Et ajouter le mot de passe Vault dans le Key Store Semaphore PROD.

3. **Variables d'environnement** : définir dans `/etc/semaphore/semaphore.env`
   et référencer via `lookup('env', 'SMTP_PASSWORD')`

---

## 5. Résumé des options

| Option                           | Avantages                                | Inconvénients                          |
|----------------------------------|------------------------------------------|----------------------------------------|
| Alertes natives Semaphore PROD   | Zéro modification playbook, simple       | Peu de personnalisation du message     |
| Module `community.general.mail`  | Message très personnalisé                | Collection à installer offline         |
| Webhook via `uri` module         | Aucune dépendance, Teams/Slack natif     | Message moins riche que l'email        |
| SMTP Semaphore PROD natif        | Intégré, aucun playbook à modifier       | Format de message fixe                 |

**Recommandation** : Activer les **alertes natives Semaphore PROD** pour la surveillance
de base, et ajouter un **webhook Teams/Slack** dans les playbooks avec escalade automatique
vers l'astreinte en cas d'échec.
