# NetworkPolicies Kubernetes Whispr

## Principe général

Chaque namespace applique une politique **default-deny** : tout trafic entrant et sortant est bloqué par défaut. Les communications autorisées sont définies explicitement via des `NetworkPolicy`.

## Règle des 3 directions

Pour chaque flux autorisé entre service A et service B, 3 règles sont nécessaires :

1. **Egress** sur A : autoriser A → B:port
2. **Ingress** sur B : autoriser B ← A
3. **DNS egress** global : autoriser UDP/TCP port 53 vers kube-dns (sinon les pods ne résolvent rien)

## DNS egress global

Une NetworkPolicy `allow-dns-egress` est appliquée dans `whispr-preprod` et `whispr-prod` :

```yaml
# s'applique à tous les pods du namespace
egress:
  - ports:
      - port: 53
        protocol: UDP
      - port: 53
        protocol: TCP
    to:
      - namespaceSelector:
          matchLabels:
            kubernetes.io/metadata.name: kube-system
```

## Matrice des communications autorisées

### messaging-service (egress)

| Destination | Port | Protocole | Raison |
|---|---|---|---|
| auth-service | 3010 | TCP | Validation JWKS |
| user-service | 3011 | TCP | Vérification contacts |
| notification-service | 4011 | TCP | Trigger push REST |
| notification-service | 40011 | TCP | Trigger push gRPC |
| postgresql | 5432 | TCP | Base de données |
| redis | 6379 | TCP | Cache + pub/sub |

### user-service (egress)

| Destination | Port | Protocole | Raison |
|---|---|---|---|
| media-service | 3012 | TCP | Upload avatar, médias profil |
| postgresql | 5432 | TCP | Base de données |
| redis | 6379 | TCP | Cache |

### calls-service (egress)

| Destination | Port | Protocole | Raison |
|---|---|---|---|
| messaging-service | 40010 | TCP | gRPC : métadonnées appel |
| auth-service | 3010 | TCP | Validation JWT |
| livekit | 7880 | TCP | LiveKit API |
| livekit | 7881 | TCP | LiveKit RTC |

### scheduling-service (egress)

| Destination | Port | Protocole | Raison |
|---|---|---|---|
| messaging-service | 4010 | TCP | Envoi du message programmé |
| user-service | 3011 | TCP | Vérification utilisateur |

### auth-service (egress)

| Destination | Port | Protocole | Raison |
|---|---|---|---|
| postgresql | 5432 | TCP | Stockage users, prekeys |
| redis | 6379 | TCP | Cache OTP |

### notification-service (egress)

| Destination | Port | Protocole | Raison |
|---|---|---|---|
| auth-service | 3010 | TCP | JWKS pour validation |
| redis | 6379 | TCP | State de notification |
| internet (FCM/APNs) | 443 | TCP | Push externe |

## Incident connu - 2026-05-10 (PR #227)

**Symptôme** : outage complet de 27h sur preprod. Aucun service ne pouvait accéder à PostgreSQL, Redis ou MinIO.

**Cause** : la NetworkPolicy `default-deny` dans `whispr-preprod` avait un `podSelector: {}` qui sélectionnait également les pods des namespaces `postgresql`, `redis`, `minio` (mauvais ciblage namespace).

**Fix appliqué** : ajout de la NetworkPolicy `infra-services-allow-ingress` ciblant spécifiquement les pods infra :

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: infra-services-allow-ingress
  namespace: postgresql   # répété pour redis et minio
spec:
  podSelector:
    matchExpressions:
      - key: app
        operator: In
        values: [postgresql, redis, minio]
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: whispr-preprod
```

## Commandes de debug

```bash
# Lister les NetworkPolicies d'un namespace
kubectl describe networkpolicy -n whispr-preprod

# Tester la connectivité depuis un pod
kubectl exec <pod-name> -n whispr-preprod -- curl <service>:<port>/health

# Vérifier les logs de rejection (si Calico ou Cilium installé)
kubectl logs -n kube-system -l k8s-app=calico-node --tail=50

# Tester DNS depuis un pod
kubectl exec <pod-name> -n whispr-preprod -- nslookup postgresql.postgresql.svc.cluster.local
```
