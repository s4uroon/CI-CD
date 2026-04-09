# Flag de validation HP → Production

## Concept

Le flag de validation est le mécanisme qui garantit qu'aucun déploiement en
Production ne peut se faire sans validation humaine explicite du déploiement HP.

Il prend la forme d'un **tag git léger** posé directement dans le dépôt bare
du serveur Git+Ansible HP.

---

## Cycle de vie du flag

```
[HP] Bouton 3 — Valider HP
  └─ git tag -f "validated/hp/<repo_name>" <commit_hash>
       └─ posé dans /opt/git/<repo_name>.git sur git-ansible-hp

[PROD] Bouton 1 — Synchro Git Prod
  └─ ssh deploy@git-ansible-hp \
       "git -C /opt/git/<repo_name>.git tag -l 'validated/hp/<repo_name>'"
       └─ si absent  → BLOQUÉ avec message explicite
       └─ si présent → fetch autorisé, commit validé affiché
```

---

## Format du tag

```
validated/hp/<repo_name>
```

Exemples :
- `validated/hp/myapp`
- `validated/hp/api-backend`
- `validated/hp/frontend`

Le tag est **lightweight** (pas annoté) et pointe sur le commit HEAD de la
branche au moment de la validation. Il est **remplacé** (`-f`) à chaque
nouvelle validation, ce qui permet de re-valider après un correctif.

---

## Workflow complet

### Étapes HP (Semaphore HP)

1. **Bouton 1 — Synchro Git HP**
   - Git+Ansible HP fetche le dépôt depuis GitLab
   - Le miroir bare HP est mis à jour

2. **Bouton 2 — Déployer HP**
   - Les serveurs App HP fetchent depuis Git+Ansible HP
   - L'application est déployée en environnement de recette

3. **Bouton 3 — Valider HP** ← *validation humaine*
   - Exécuté par l'équipe après tests manuels/automatiques en HP
   - Crée le tag `validated/hp/<repo_name>` sur le commit actuel
   - **Sans ce bouton, le Prod est bloqué**

### Étapes Prod (Semaphore Prod)

4. **Bouton 1 — Synchro Git Prod**
   - Vérifie la présence du tag `validated/hp/<repo_name>` via SSH
   - Si absent : échec avec liste des boutons HP à exécuter
   - Si présent : fetch depuis Git+Ansible HP, affiche le commit validé

5. **Bouton 2 — Déployer Prod**
   - Sauvegarde BDD, déploiement, migrations, redémarrage service

---

## Vérification manuelle du flag

Depuis le serveur Git+Ansible Prod, vérifier si un flag existe :

```bash
ssh -i /opt/keys/id_gitprod_to_githp deploy@git-ansible-hp \
  "git -C /opt/git/myapp.git tag -l 'validated/hp/myapp'"
```

Afficher le commit pointé par le flag :

```bash
ssh -i /opt/keys/id_gitprod_to_githp deploy@git-ansible-hp \
  "git -C /opt/git/myapp.git rev-parse 'validated/hp/myapp'"
```

Supprimer un flag (pour forcer une re-validation) :

```bash
# Sur git-ansible-hp
git -C /opt/git/myapp.git tag -d validated/hp/myapp
```

---

## Cas d'usage

### Re-validation après correctif en HP

Si un bug est découvert après la validation :

1. Déployer le correctif en HP (Bouton 1 + Bouton 2 HP)
2. Tester à nouveau
3. Relancer le **Bouton 3 HP** → le tag est déplacé sur le nouveau commit
4. Relancer le **Bouton 1 Prod** → le nouveau commit validé est détecté

### Blocage intentionnel de la Production

Pour empêcher temporairement tout déploiement en Prod pour un dépôt :

```bash
# Sur git-ansible-hp
git -C /opt/git/myapp.git tag -d validated/hp/myapp
```

La Prod sera bloquée jusqu'à la prochaine validation HP.

---

## Sécurité

- Le tag est posé **côté HP uniquement** — la Prod n'a qu'un accès SSH read-only
- La clé SSH utilisée par Git+Ansible Prod (Clé F) est restreinte à
  `git-upload-pack` et `git tag -l` en lecture seule
- Aucun écriture n'est possible depuis la Prod vers HP
- Le tag ne peut être créé que par Semaphore HP (Clé A → Git+Ansible HP)

---

## Exemple de sortie lors d'un blocage

```
BLOQUE : Aucun flag de validation HP trouvé pour myapp.

Actions requises sur Semaphore HP :
  1. Bouton 1 — Synchro Git HP    (si pas encore fait)
  2. Bouton 2 — Deployer HP       (déployer en recette)
  3. Bouton 3 — Valider HP        (valider après tests)

Ensuite relancez ce bouton.
```
