# Stratégie de Rollback — Zone Production (PROD)

> Document dédié à la zone Production. Pour le rollback HP, voir [rollback-strategy_hp.md](./rollback-strategy_hp.md).

---

## Principe

Le rollback PROD repose sur les **tags Git sémantiques** (`v1.x.y`) et les
**sauvegardes BDD automatiques** créées avant chaque déploiement. Il s'effectue
via le **Bouton 3 PROD** dans Semaphore PROD.

```
v1.0.0  v1.1.0  v1.2.0  v1.3.0  v1.4.0 (prod actuelle)
  |       |       |       |       |
  o───────o───────o───────o───────o───── main
                                  ▲
                            Dernier déploiement
```

Si `v1.4.0` pose problème → rollback vers `v1.3.0` en un clic.

---

## Convention de nommage des tags

```
v<MAJEUR>.<MINEUR>.<PATCH>[-<suffixe>]

Exemples :
  v1.4.2          → tag standard (déployé en PROD)
  v1.4.2-hotfix   → correctif urgent

Tags de sauvegarde automatiques (créés avant chaque déploiement PROD) :
  backup/prod-pre-deploy-<date>-<heure>
  Exemple : backup/prod-pre-deploy-2026-04-20-14-30-00
```

Le tag est créé **après** validation HP, **avant** synchro PROD :

```bash
git tag -a v1.4.2 -m "Release 1.4.2 - Nouvelle fonctionnalité X"
git push gitlab v1.4.2
```

---

## Bouton 3 PROD — Rollback PROD

| Template      | Playbook                    | Paramètres Semaphore              |
|---------------|-----------------------------|-----------------------------------|
| Rollback PROD | `prod/03_rollback_prod.yml` | `repo_name` + `rollback_tag`     |

---

## Playbook de rollback : `prod/03_rollback_prod.yml`

```yaml
---
# ansible/playbooks/prod/03_rollback_prod.yml
# Bouton 3 PROD : Rollback Production
# Paramètres obligatoires : repo_name, rollback_tag (ex: v1.3.0)

- name: Rollback Production vers une version antérieure
  hosts: prod
  become: true
  become_user: deploy
  vars:
    app_dir: "{{ app_root }}"

  pre_tasks:
    - name: Vérifier que rollback_tag est défini
      ansible.builtin.fail:
        msg: >
          La variable 'rollback_tag' est obligatoire.
          Exemple : rollback_tag=v1.3.0
      when: rollback_tag is not defined or rollback_tag == ""

    - name: Afficher la version cible du rollback
      ansible.builtin.debug:
        msg: "ROLLBACK PROD vers la version : {{ rollback_tag }}"

    - name: Vérifier que le tag existe dans le dépôt local
      ansible.builtin.command:
        cmd: git tag -l "{{ rollback_tag }}"
        chdir: "{{ app_dir }}"
      register: tag_check
      changed_when: false

    - name: Échouer si le tag n'existe pas
      ansible.builtin.fail:
        msg: >
          Le tag '{{ rollback_tag }}' n'existe pas dans le dépôt local.
          Exécutez d'abord 'Synchro Git Prod' pour récupérer les tags récents.
      when: tag_check.stdout == ""

    - name: Enregistrer le commit actuel (pour logs d'audit)
      ansible.builtin.command:
        cmd: git rev-parse HEAD
        chdir: "{{ app_dir }}"
      register: current_commit
      changed_when: false

    - name: Afficher le commit actuel avant rollback
      ansible.builtin.debug:
        msg: "Commit actuel (avant rollback) : {{ current_commit.stdout }}"

  tasks:
    - name: Sauvegarde BDD avant rollback PROD
      ansible.builtin.command:
        cmd: >
          pg_dump -Fc {{ db_name_prod }} >
          /opt/backups/{{ repo_name }}_pre-rollback_{{ ansible_date_time.date }}.dump
      when: run_migrations | default(false)

    - name: Stopper le service avant rollback
      ansible.builtin.systemd:
        name: "{{ app_service_name }}"
        state: stopped
      become: true
      become_user: root

    - name: Checkout du tag de rollback
      ansible.builtin.command:
        cmd: git checkout "tags/{{ rollback_tag }}" --force
        chdir: "{{ app_dir }}"
      register: checkout_result

    - name: Afficher le résultat du checkout
      ansible.builtin.debug:
        var: checkout_result.stdout_lines

    - name: Installer les dépendances de la version cible
      ansible.builtin.include_role:
        name: deploy
        tasks_from: install_deps

    - name: Appliquer les migrations de rollback (si elles existent)
      ansible.builtin.include_role:
        name: deploy
        tasks_from: run_migrations_rollback

    - name: Redémarrer le service avec la version rollbackée
      ansible.builtin.systemd:
        name: "{{ app_service_name }}"
        state: restarted
      become: true
      become_user: root

    - name: Vérifier que le service est actif
      ansible.builtin.systemd:
        name: "{{ app_service_name }}"
      register: service_status
      failed_when: service_status.status.ActiveState != "active"

  post_tasks:
    - name: Afficher le résumé du rollback
      ansible.builtin.debug:
        msg:
          - "============================================"
          - "ROLLBACK PROD EFFECTUE AVEC SUCCES"
          - "Version antérieure : {{ current_commit.stdout[:8] }}"
          - "Version courante   : {{ rollback_tag }}"
          - "Serveur            : {{ inventory_hostname }}"
          - "Date               : {{ ansible_date_time.iso8601 }}"
          - "============================================"
```

---

## Configuration du Template Rollback dans Semaphore PROD

Dans Semaphore PROD → **Task Templates** → **New Template** :

```yaml
Nom          : Rollback PROD
Playbook     : ansible/playbooks/prod/03_rollback_prod.yml
Inventaire   : ansible/inventory/prod/hosts.yml
Repository   : CI-CD
Environnement: env_prod

Survey Variables:
  - name        : repo_name
    title       : "Nom du dépôt (ex: myapp)"
    type        : String
    required    : true
  - name        : rollback_tag
    title       : "Tag cible du rollback (ex: v1.4.2 ou backup/prod-pre-deploy-2026-04-20-14-30-00)"
    description : "Tag Git de la version vers laquelle revenir"
    type        : String
    required    : true
```

---

## Procédure de rollback PROD pas à pas

### 1. Identification du problème

```
1. Alerte monitoring ou signalement utilisateur
2. Confirmer le problème en production
3. Décision de rollback prise par le responsable technique
```

### 2. Identification de la version cible

```bash
# Depuis Git+Ansible PROD ou tout poste avec accès Git
git log --oneline --decorate --tags --no-walk

# Exemple de sortie :
# a1b2c3d (tag: v1.4.2) Release 1.4.2
# e5f6g7h (tag: v1.4.1) Release 1.4.1
# i9j0k1l (tag: v1.4.0) Release 1.4.0

# Tags de sauvegarde automatiques (créés avant chaque déploiement PROD)
git tag -l 'backup/prod-pre-deploy-*'
```

### 3. Exécution du rollback PROD

```
1. Ouvrir Semaphore PROD → https://semaphore-prod.company.com
2. Cliquer sur le template "Rollback PROD"
3. Saisir : repo_name = myapp
4. Saisir : rollback_tag = v1.3.0 (ou le tag backup)
5. Confirmer et lancer
6. Surveiller les logs en temps réel dans Semaphore PROD
7. Vérifier le service en production
```

### 4. Durée estimée

| Étape                    | Durée typique |
|--------------------------|---------------|
| Sauvegarde BDD           | 1–5 minutes   |
| Checkout Git             | < 5 secondes  |
| Install dépendances      | 1–3 minutes   |
| Migrations rollback      | Variable       |
| Redémarrage service      | < 10 secondes |
| **Total**                | **2–10 min**  |

---

## Stratégie de sauvegarde complémentaire

### Snapshot avant déploiement PROD

Le playbook `prod/02_deploy_prod.yml` crée automatiquement un tag de sauvegarde
avant chaque déploiement :

```yaml
- name: Créer un tag de sauvegarde avant déploiement PROD
  ansible.builtin.command:
    cmd: >
      git tag -f "backup/prod-pre-deploy-{{ ansible_date_time.date }}-{{ ansible_date_time.time | replace(':', '-') }}"
    chdir: "{{ app_dir }}"
```

Cela garantit qu'on peut toujours revenir à l'état exact avant un déploiement,
même si aucun tag sémantique `v1.x.y` n'a été créé manuellement.

---

## Rollback de base de données

> Le rollback Git ne suffit pas si des **migrations de base de données
> irréversibles** ont été appliquées.

### Recommandations

1. **Migrations additive-only** : n'ajouter que des colonnes/tables, ne jamais
   supprimer lors d'un déploiement initial.

2. **Scripts de rollback dédiés** : chaque migration (`up`) doit avoir son
   équivalent (`down`) :
   ```
   migrations/
     001_add_users_table.up.sql
     001_add_users_table.down.sql
   ```

3. **Sauvegardes BDD automatiques** : le playbook `prod/02_deploy_prod.yml` inclut
   une tâche de dump avant migration :
   ```yaml
   - name: Sauvegarde BDD avant migration PROD
     ansible.builtin.command:
       cmd: >
         pg_dump -Fc {{ db_name_prod }} > /opt/backups/{{ repo_name }}_{{ ansible_date_time.date }}.dump
   ```

4. **Restauration BDD** : si le rollback applicatif ne suffit pas, restaurer
   le dump manuellement ou via un playbook dédié `04_restore_db.yml`.

---

## Matrice décisionnelle de rollback PROD

```
Problème détecté en PROD
        │
        ▼
Le service démarre-t-il ?
    │           │
   OUI          NON
    │            └──► Rollback immédiat (Bouton 3 PROD)
    │
    ▼
Problème de données ou de comportement ?
    │           │
   Données      Comportement
    │            └──► Rollback applicatif (Bouton 3 PROD)
    │                 puis surveiller
    │
    ▼
Migration BDD irréversible impliquée ?
    │           │
   OUI          NON
    │            └──► Rollback applicatif (Bouton 3 PROD)
    │
    ▼
Restaurer dump BDD + Rollback applicatif (Bouton 3 PROD)
Informer l'équipe des données potentiellement perdues
```
