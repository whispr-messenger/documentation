# Secrets et Configuration k8s

## Pattern général

Les secrets sont injectés via `envFrom.secretRef` dans les deployments. Aucune valeur sensible n'est commitée en clair dans le repo `infrastructure`.

```yaml
# Exemple deployment.yaml
envFrom:
  - secretRef:
      name: auth-service-env
```

## Nomenclature

Les secrets suivent la convention `<service>-env` en preprod et prod.

Exemples : `auth-service-env`, `messaging-service-env`, `user-service-env`, `notification-service-env`.

## Inventaire des secrets par service

| Secret | Clés importantes | Usage |
|---|---|---|
| `auth-service-env` | `JWT_ISSUER`, `JWT_AUDIENCE`, `OTP_BYPASS_CODE`, `SECRET_KEY_BASE` | `OTP_BYPASS_CODE=123456` en preprod uniquement - ne jamais mettre en prod |
| `messaging-service-env` | `JWT_ISSUER`, `JWT_AUDIENCE`, `INTERNAL_API_TOKEN` | Validation des tokens, auth inter-services |
| `user-service-env` | `JWT_ISSUER`, `JWT_AUDIENCE`, `SEED_ADMIN_USER_IDS`, `BREVO_API_KEY`, `BREVO_WAITLIST_LIST_ID` | Profil, mailing Brevo (waitlist) |
| `notification-service-env` | `AUTH_JWKS_URL`, `JWT_ISSUER`, `JWT_AUDIENCE`, `CORS_ALLOWED_ORIGINS` | Validation JWT, CORS des connexions push |
| `shared-internal-api-token` | `INTERNAL_API_TOKEN` | Token partagé messaging ↔ user ↔ scheduling ↔ notification |
| `notification-fcm-keyfile` | JSON Service Account Firebase complet | Push Android natif (post-démo) |
| `notification-service-apns` | `AuthKey.p8`, `APNS_KEY_ID`, `APNS_TEAM_ID` | Push iOS natif (post-démo) |
| `livekit-keys` | `LIVEKIT_API_KEY`, `LIVEKIT_API_SECRET`, `LIVEKIT_WEBHOOK_SECRET` | Signaling LiveKit (calls-service) |

## ArgoCD et selfHeal

`selfHeal: true` est actif sur toutes les apps ArgoCD.

- Les ressources trackées (Deployment, ConfigMap, Service...) sont revertées en **~10 min** si patchées en live.
- Pour un changement durable → committer dans `infrastructure/<branche>`.
- Les **Secrets sont NON trackés** par ArgoCD → un `kubectl patch secret` est permanent et ne sera pas revert.

## Commandes secrets

```bash
# Lire la valeur d'une clé dans un secret
kubectl get secret auth-service-env -n whispr-preprod \
  -o jsonpath='{.data.OTP_BYPASS_CODE}' | base64 -d

# Patcher une clé dans un secret existant
kubectl patch secret messaging-service-env -n whispr-preprod \
  --type merge \
  -p '{"data":{"INTERNAL_API_TOKEN":"<base64-encoded-value>"}}'

# Encoder une valeur en base64 pour le patch
echo -n "ma-valeur-secrete" | base64

# Lister tous les secrets d'un namespace
kubectl get secrets -n whispr-preprod

# Voir les clés d'un secret (sans les valeurs)
kubectl get secret auth-service-env -n whispr-preprod -o jsonpath='{.data}' | python3 -m json.tool
```

## Points d'attention

- `OTP_BYPASS_CODE=123456` est défini dans `auth-service-env` en preprod pour faciliter les tests. Ne jamais le mettre en prod ni le committer.
- `notification-fcm-keyfile` contient le JSON complet du Service Account Firebase - rotation nécessaire si leaké.
- `notification-service-apns` contient la clé privée Apple (`AuthKey.p8`) - non renouvelable facilement.
- En cas de rotation du `INTERNAL_API_TOKEN`, patcher `shared-internal-api-token` ET redémarrer tous les services qui le consomment (messaging, user, scheduling, notification).
