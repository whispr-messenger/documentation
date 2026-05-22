# Pipeline CI/CD et GitOps

## Flow complet

```
  developer push
       │
       ▼
┌─────────────────────────────────────────────────┐
│           GitHub Actions (CI)                   │
│  1. tests + lint                                │
│  2. docker build                                │
│  3. trivy scan (CVE)                            │
│  4. SBOM generation                             │
│  5. push image → ghcr.io/whispr-messenger/<svc> │
└─────────────────────┬───────────────────────────┘
                      │ image pushed
                      ▼
┌─────────────────────────────────────────────────┐
│           GitHub Actions (CD)                   │
│  bump manifest infrastructure/                  │
│  → edit deployment.yaml image tag               │
│  → git commit + push                            │
└─────────────────────┬───────────────────────────┘
                      │
          ┌───────────┴───────────┐
          │                       │
          ▼                       ▼
   branch: main           branch: deploy/preprod
          │                       │
          ▼                       ▼
  ArgoCD (ns argocd)    ArgoCD (ns argocd-preprod)
  apps: citadel-*        apps: preprod-*
          │                       │
          ▼                       ▼
  kubectl apply          kubectl apply
  (whispr-prod)          (whispr-preprod)
          │                       │
          ▼                       ▼
  rolling update          rolling update
  pods prod               pods preprod
```

## Stratégie de branches

| Branche | Environnement | ArgoCD namespace | ArgoCD apps |
|---|---|---|---|
| `deploy/preprod` | `whispr-preprod` | `argocd-preprod` | `preprod-*` |
| `main` | `whispr-prod` | `argocd` | `citadel-*` |

- Travail quotidien sur `deploy/preprod`.
- Merge vers `main` uniquement après validation fonctionnelle sur preprod.
- Jamais de push direct sur `main` — toujours via PR.

## Problème connu : CD mobile-app

Le pipeline CD de `mobile-app` bumpe le manifest dans `infrastructure/main` uniquement.  
ArgoCD preprod track `infrastructure/deploy/preprod` → le bump n'arrive pas en preprod automatiquement.

**Workaround manuel :**
```bash
# Récupérer le nouveau SHA d'image depuis infrastructure/main
git log --oneline infrastructure/main -- mobile-app/deployment.yaml

# Appliquer sur deploy/preprod
git checkout deploy/preprod
sed -i 's|sha-old|sha-new|g' infrastructure/deploy/preprod/mobile-app/deployment.yaml
git add infrastructure/deploy/preprod/mobile-app/deployment.yaml
git commit -m "chore(mobile-app): cherry-pick image bump sha-new"
git push origin deploy/preprod
```

## selfHeal ArgoCD

`selfHeal: true` est activé sur toutes les apps ArgoCD.  
- Tout `kubectl apply` ou `kubectl patch` direct sur une ressource trackée sera **revert en max ~10 min**.
- Pour un patch permanent → committer le changement dans le repo `infrastructure` sur la bonne branche.
- **Exception** : les Secrets ne sont PAS trackés par ArgoCD → un `kubectl patch secret` est permanent.

## Commandes de vérification

```bash
# Statut CI sur une branche
gh run list --branch <branch> -L 3

# Logs d'un run en échec
gh run view <run-id> --log-failed

# Image réellement déployée sur un service
kubectl get deploy <svc> -n whispr-preprod \
  -o jsonpath='{.spec.template.spec.containers[0].image}'

# Statut ArgoCD preprod
kubectl get app -n argocd-preprod

# Sync manuel si ArgoCD est OutOfSync
argocd app sync preprod-<svc> --server argocd.roadmvn.com
```

## Checklist post-push

- [ ] CI vert : `gh run list --branch <branch> -L 1` → status `completed / success`
- [ ] Lire les logs (`gh run view --log-failed`) — warnings cachent des fails
- [ ] CD bump vérifié : commit de bump présent dans `infrastructure/<branche>`
- [ ] ArgoCD Synced + Healthy : `kubectl get app -n argocd-preprod`
- [ ] Image déployée correcte : `kubectl get deploy <svc> -o jsonpath=...`
- [ ] Smoke test : `curl https://whispr-preprod.roadmvn.com/health` (ou endpoint du service)
