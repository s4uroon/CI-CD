# Stratégie de Rollback

## Principe

Le rollback repose sur les **tags Git sémantiques** (`v1.x.y`) créés au moment
de chaque déploiement en production. Chaque tag pointe vers un commit précis,
permettant de revenir à n'importe quelle version antérieure de manière
**déterministe et reproductible**.

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
v<MAJEUR>.<MINEUR>.<PATCH>[-<env>]

Exemples :
  v1.4.2          → tag standard (déployé en prod)
  v1.4.2-rc1      → release candidate (déployé en hors-prod uniquement)
  v1.4.2-hotfix   → correctif urgent
```

Le tag est créé **après** validation en hors-prod, **avant** synchro Prod :

```bash
# Après validation hors-prod :
git tag -a v1.4.2 -m "Release 1.4.2 - Nouvelle fonctionnalité X"
git push gitlab v1.4.2
```

---

## Les 5 Task Templates dans Semaphore

| # | Template            | Playbook                    | Paramètre requis     |
|---|---------------------|-----------------------------|----------------------|
| 1 | Synchro Hors-Prod   | `01_sync_hors_prod.yml`     | branch/tag optionnel |
| 2 | Déployer Hors-Prod  | `02_deploy_hors_prod.yml`   | —                    |
| 3 | Synchro Prod        | `03_sync_prod.yml`          | —                    |
| 4 | Déployer Prod       | `04_deploy_prod.yml`        | —                    |
| 5 | **Rollback Prod**   | `05_rollback_prod.yml`      | `target_version`     |

---

## Playbook de rollback : `05_rollback_prod.yml`

```yaml
---
# ansible/playbooks/05_rollback_prod.yml
# Bouton 5 Semaphore : Rollback Production
# Paramètre obligatoire : target_version (ex: v1.3.0)

- name: Rollback Production vers une version antérieure
  hosts: prod
  become: true
  become_user: deploy
  vars:
    # Semaphore passe cette variable via "Extra Variables" dans le template
    # target_version: "v1.3.0"   # défini dans Semaphore Survey/Variables
    app_dir: "{{ app_root }}"

  pre_tasks:
    - name: Vérifier que target_version est défini
      ansible.builtin.fail:
        msg: >
          La variable 'target_version' est obligatoire.
          Exemple : target_version=v1.3.0
      when: target_version is not defined or target_version == ""

    - name: Afficher la version cible du rollback
      ansible.builtin.debug:
        msg: "ROLLBACK vers la version : {{ target_version }}"

    - name: Vérifier que le tag existe dans le dépôt local
      ansible.builtin.command:
        cmd: git tag -l "{{ target_version }}"
        chdir: "{{ app_dir }}"
      register: tag_check
      changed_when: false

    - name: Échouer si le tag n'existe pas
      ansible.builtin.fail:
        msg: >
          Le tag '{{ target_version }}' n'existe pas dans le dépôt local.
          Exécutez d'abord 'Synchro Prod' pour récupérer les tags récents.
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
    - name: Stopper le service avant rollback
      ansible.builtin.systemd:
        name: "{{ app_service_name }}"
        state: stopped
      become: true
      become_user: root

    - name: Checkout du tag de rollback
      ansible.builtin.command:
        cmd: git checkout "tags/{{ target_version }}" --force
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
          - "ROLLBACK EFFECTUE AVEC SUCCES"
          - "Version antérieure : {{ current_commit.stdout[:8] }}"
          - "Version courante   : {{ target_version }}"
          - "Serveur            : {{ inventory_hostname }}"
          - "Date               : {{ ansible_date_time.iso8601 }}"
          - "============================================"
```

---

## Configuration du Template Rollback dans Semaphore

### Création du template

Dans Semaphore → **Task Templates** → **New Template** :

```yaml
Nom          : Rollback Prod
Playbook     : ansible/playbooks/05_rollback_prod.yml
Inventaire   : hosts.yml
Repository   : CI-CD (ce dépôt)
Environnement: env_prod

# Dans l'onglet "Survey Variables" (formulaire interactif) :
Survey Variables:
  - name        : target_version
    title       : "Version cible du rollback (ex: v1.4.2)"
    description : "Tag Git de la version vers laquelle revenir"
    type        : String
    required    : true
```

L'interface Semaphore affichera alors un champ texte à remplir avant
de lancer le rollback.

---

## Procédure de rollback pas à pas

### Identification du problème

```
1. Alerte monitoring ou signalement utilisateur
2. Confirmer le problème en production
3. Décision de rollback prise par le responsable technique
```

### Identification de la version cible

```bash
# Depuis l'Orchestrateur ou n'importe quel poste avec accès Git :
git log --oneline --decorate --tags --no-walk

# Exemple de sortie :
# a1b2c3d (tag: v1.4.2) Release 1.4.2
# e5f6g7h (tag: v1.4.1) Release 1.4.1
# i9j0k1l (tag: v1.4.0) Release 1.4.0
# ...
```

### Exécution du rollback

```
1. Ouvrir Semaphore
2. Cliquer sur "Rollback Prod"
3. Saisir la version cible (ex: v1.3.0)
4. Confirmer et lancer
5. Surveiller les logs en temps réel dans Semaphore
6. Vérifier le service en production
```

### Durée estimée

| Étape                    | Durée typique |
|--------------------------|---------------|
| Checkout Git             | < 5 secondes  |
| Install dépendances      | 1–3 minutes   |
| Migrations rollback      | Variable       |
| Redémarrage service      | < 10 secondes |
| **Total**                | **2–5 min**   |

---

## Stratégie de sauvegarde complémentaire

### Snapshot avant déploiement Prod

Le playbook `04_deploy_prod.yml` crée automatiquement un tag de sauvegarde
avant chaque déploiement :

```yaml
- name: Créer un tag de sauvegarde avant déploiement
  ansible.builtin.command:
    cmd: >
      git tag -f "backup/pre-deploy-{{ ansible_date_time.date }}-{{ ansible_date_time.time | replace(':', '-') }}"
    chdir: "{{ app_dir }}"
```

Cela garantit qu'on peut toujours revenir à l'état exact avant un déploiement,
même si aucun tag sémantique n'a été créé.

---

## Rollback de base de données

> Le rollback Git ne suffit pas si des **migrations de base de données
> irréversibles** ont été appliquées.

### Recommandations

1. **Migrations additive-only** : n'ajouter que des colonnes/tables, ne jamais
   supprimer lors d'un déploiement initial. La suppression se fait dans une
   release ultérieure après validation.

2. **Scripts de rollback dédiés** : chaque migration (`up`) doit avoir son
   équivalent (`down`) :
   ```
   migrations/
     001_add_users_table.up.sql
     001_add_users_table.down.sql
   ```

3. **Sauvegardes BDD automatiques** : le playbook `04_deploy_prod.yml` inclut
   une tâche de dump avant migration :
   ```yaml
   - name: Sauvegarde BDD avant migration
     ansible.builtin.command:
       cmd: >
         pg_dump -Fc myapp > /opt/backups/myapp_{{ ansible_date_time.date }}.dump
   ```

4. **Restauration BDD** : si le rollback applicatif ne suffit pas, restaurer
   le dump manuellement ou via un playbook dédié `06_restore_db.yml`.

---

## Matrice décisionnelle de rollback

```
Problème détecté en Prod
        │
        ▼
Le service démarre-t-il ?
    │           │
   OUI          NON
    │            └──► Rollback immédiat (Bouton 5)
    │
    ▼
Problème de données ou de comportement ?
    │           │
   Données      Comportement
    │            └──► Rollback applicatif (Bouton 5)
    │                 puis surveiller
    │
    ▼
Migration BDD irréversible impliquée ?
    │           │
   OUI          NON
    │            └──► Rollback applicatif (Bouton 5)
    │
    ▼
Restaurer dump BDD + Rollback applicatif
Informer l'équipe des données perdues potentielles
```
