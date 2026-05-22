# Stratégie de Branches et Workflow Git

## Règle fondamentale

**Jamais de push direct sur `main`.** Toujours via PR depuis une branche feature.

## Flow standard

```
  feature/ma-feature
         │
         │ PR (review + CI vert)
         ▼
  deploy/preprod  ──────► whispr-preprod.roadmvn.com
         │                (auto-CD via ArgoCD preprod-*)
         │
         │ tests live validés
         │ PR de release
         ▼
       main  ────────────► whispr-api.roadmvn.com (prod)
                           (auto-CD via ArgoCD citadel-*)
```

## Rôle de chaque branche

| Branche | Environnement | Utilisation |
|---|---|---|
| `feature/*` | local | Développement d'une fonctionnalité isolée |
| `deploy/preprod` | preprod | Branche de travail principale 99% du temps |
| `main` | prod | Merge uniquement après validation preprod |

## Branches partagées - mobile-app

La branche `deploy/preprod` de `mobile-app` est partagée avec l'équipe (Hou, xAPT42, DALM1).

**Rebase obligatoire avant chaque push :**
```bash
cd mobile-app
git pull --rebase origin deploy/preprod
git push origin deploy/preprod
```

Ne jamais force-push sur `deploy/preprod` sans prévenir l'équipe.

## Merge method

Toujours `--merge`. Jamais `--squash`.

```bash
# Merge standard
gh pr merge <num> --merge

# Si branch protection bloque et que tous les checks bloquants sont verts
gh pr merge <num> --merge --admin
```

Le squash écrase l'historique granulaire et peut faire disparaître les commits de bump d'image générés par le CD dans le repo `infrastructure`. Ces commits doivent rester traçables.

## Mapping ArgoCD

| ArgoCD namespace | Apps | Track |
|---|---|---|
| `argocd-preprod` | `preprod-*` | `infrastructure/deploy/preprod` |
| `argocd` | `citadel-*` | `infrastructure/main` |

## Conventions de commits

Format Conventional Commits :
```
<type>(<scope>): <sujet court>
```

Types courants : `feat`, `fix`, `chore`, `docs`, `perf`, `refactor`, `style`, `test`, `ci`.

Règles :
- Tirets `-` uniquement, jamais `—` ni `–`.
- Pas de mention IA, Claude, Co-Authored-By AI.
- Message en anglais ou français simple, pas de majuscule après les deux-points.
- Pseudo git : `roadman`.

Exemples corrects :
```
feat(messaging): add reaction support on group messages
fix(auth): handle OTP expiry edge case on retry
chore(infra): bump messaging-service to sha-abc123
```

## Incident connu : CD mobile-app et preprod

Le pipeline CD de `mobile-app` bumpe le manifest dans `infrastructure/main` uniquement.  
ArgoCD preprod watch `infrastructure/deploy/preprod` → le bump n'arrive jamais en preprod automatiquement.

**Cherry-pick manuel nécessaire :**
```bash
# 1. Récupérer le SHA du nouveau commit de bump sur infrastructure/main
git log --oneline origin/main -- deploy/mobile-app/deployment.yaml | head -1

# 2. Se placer sur deploy/preprod du repo infrastructure
git checkout deploy/preprod

# 3. Modifier manuellement l'image tag
sed -i 's|ghcr.io/whispr-messenger/mobile-app:sha-old|ghcr.io/whispr-messenger/mobile-app:sha-new|g' \
  deploy/mobile-app/deployment.yaml

# 4. Committer et pousser
git add deploy/mobile-app/deployment.yaml
git commit -m "chore(mobile-app): cherry-pick image bump sha-new to preprod"
git push origin deploy/preprod
```

## Commandes utiles

```bash
# Statut CI sur deploy/preprod
gh run list --branch deploy/preprod --limit 5

# Créer une PR feature → deploy/preprod
gh pr create --base deploy/preprod --title "feat(scope): ..." --body "..."

# Créer une PR de release deploy/preprod → main
gh pr create --base main --head deploy/preprod --title "release: ..."

# Voir les checks d'une PR avant merge
gh pr checks <num>

# Voir les commentaires bots avant merge
gh pr view <num> --json comments,reviews,statusCheckRollup
```
