# Stratégie de Rollback — Zone Hors-Prod (HP)

> Document dédié à la zone HP. Pour le rollback Production, voir [rollback-strategy_prod.md](./rollback-strategy_prod.md).

---

## Principe du rollback HP

Le rollback HP permet de revenir à une version précédente sur les App Server(s) HP
**sans sauvegarde de base de données** (la HP n'est pas l'environnement de référence
pour les données). Il s'effectue via le **Bouton 4 HP** dans Semaphore HP.

```
v1.0.0  v1.1.0  v1.2.0  v1.3.0  v1.4.0 (HP actuelle)
  |       |       |       |       |
  o───────o───────o───────o───────o───── main

Si v1.4.0 pose problème en HP → rollback vers v1.3.0 en un clic.
```

> **Important** : Après un rollback HP, re-exécuter le **Bouton 3 HP** (Valider HP)
> pour re-créer le flag `validated/hp/<repo>` sur la version rollbackée.
> Sans cette étape, le déploiement Production reste bloqué.

---

## Bouton 4 HP — Rollback HP

| Template  | Playbook                | Paramètres Semaphore              |
|-----------|-------------------------|-----------------------------------|
| Rollback HP | `hp/04_rollback_hp.yml` | `repo_name` + `rollback_tag`     |

### Configuration du template dans Semaphore HP

Dans Semaphore HP → **Task Templates** → **New Template** :

```yaml
Nom          : Rollback HP
Playbook     : ansible/playbooks/hp/04_rollback_hp.yml
Inventaire   : ansible/inventory/hp/hosts.yml
Repository   : CI-CD
Environnement: env_hp

Survey Variables:
  - name        : repo_name
    title       : "Nom du dépôt (ex: myapp)"
    type        : String
    required    : true
  - name        : rollback_tag
    title       : "Tag cible du rollback (ex: v1.3.0 ou backup/hp-pre-deploy-2026-04-20)"
    type        : String
    required    : true
```

---

## Convention de nommage des tags HP

```
v<MAJEUR>.<MINEUR>.<PATCH>[-<suffixe>]

Exemples :
  v1.4.2          → tag de release (testé en HP, prêt pour PROD)
  v1.4.2-rc1      → release candidate (test HP uniquement)
  v1.4.2-hotfix   → correctif urgent testé en HP
```

Les tags de sauvegarde automatiques créés avant chaque déploiement HP :
```
backup/hp-pre-deploy-<date>-<heure>
Exemple : backup/hp-pre-deploy-2026-04-20-14-30-00
```

---

## Procédure de rollback HP pas à pas

### 1. Identification du problème

```
1. Anomalie détectée lors des tests QA sur HP
2. Confirmer le problème (log applicatif, comportement inattendu)
3. Décision de rollback HP prise par le responsable technique
```

### 2. Identification de la version cible

```bash
# Depuis Git+Ansible HP ou tout poste avec accès Git
git log --oneline --decorate --tags --no-walk

# Exemple de sortie :
# a1b2c3d (tag: v1.4.2) Release 1.4.2
# e5f6g7h (tag: v1.4.1) Release 1.4.1
# i9j0k1l (tag: v1.4.0) Release 1.4.0

# Tags de sauvegarde automatiques
git tag -l 'backup/hp-pre-deploy-*'
```

### 3. Exécution du rollback HP

```
1. Ouvrir Semaphore HP → https://semaphore-hp.company.com
2. Cliquer sur le template "Rollback HP"
3. Saisir : repo_name = myapp
4. Saisir : rollback_tag = v1.3.0 (ou le tag de sauvegarde)
5. Confirmer et lancer
6. Surveiller les logs en temps réel dans Semaphore HP
7. Vérifier l'application sur les App Server(s) HP
```

### 4. Re-validation HP après rollback

**Étape obligatoire** pour débloquer la PROD :

```
→ Exécuter le Bouton 3 HP (Valider HP)
  → repo_name = myapp
  → Le tag validated/hp/myapp est re-créé sur le commit rollbacké
```

---

## Durée estimée du rollback HP

| Étape                    | Durée typique |
|--------------------------|---------------|
| Checkout Git             | < 5 secondes  |
| Install dépendances      | 1–3 minutes   |
| Redémarrage service      | < 10 secondes |
| **Total**                | **1–4 min**   |

---

## Cas d'usage HP spécifiques

### Rollback suite à un bug détecté en recette

```
1. Bouton 4 HP → rollback vers v1.3.0
2. Bouton 3 HP → re-valider v1.3.0
3. Informer l'équipe dev (le déploiement PROD est bloqué sur v1.3.0)
4. L'équipe dev corrige et créé v1.4.3
5. Bouton 1 HP → sync
6. Bouton 2 HP → deploy v1.4.3
7. Tests QA → OK
8. Bouton 3 HP → valider v1.4.3
9. Procéder au déploiement PROD
```

### Rollback pour retester une version précédente

```
1. Bouton 4 HP → rollback vers v1.2.0 (pour comparaison)
2. Tests comparatifs
3. Si OK : Bouton 3 HP → valider v1.2.0
4. Sinon : Bouton 1 HP → re-sync HEAD, Bouton 2 HP → re-deploy
```
