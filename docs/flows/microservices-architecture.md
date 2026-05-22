# Architecture Microservices Whispr

## Vue d'ensemble

Whispr est une messagerie multi-service (style Signal/WhatsApp) composée de 9 services indépendants.

## Tableau des services

| Service | Stack | Port HTTP | Port gRPC | Namespace k8s | Rôle |
|---|---|---|---|---|---|
| auth-service | NestJS | 3010 | - | whispr-preprod / whispr-prod | JWT, OTP, 2FA, Signal pre-keys |
| user-service | NestJS | 3011 | 50011 | whispr-preprod / whispr-prod | Profil, contacts, sanctions, appeals |
| media-service | NestJS | 3012 | - | whispr-preprod / whispr-prod | Upload S3/MinIO, quotas, thumbnails |
| messaging-service | Elixir/Phoenix | 4010 | 40010 | whispr-preprod / whispr-prod | Conversations, messages, WebSocket, réactions |
| notification-service | Elixir/Phoenix | 4011 | 40011 | whispr-preprod / whispr-prod | Push FCM/APNs, badges, mute settings |
| calls-service | Elixir/Phoenix | 4012 | - | whispr-preprod / whispr-prod | LiveKit signaling, appels 1v1 et groupe |
| scheduling-service | NestJS | 3013 | - | whispr-preprod / whispr-prod | Messages programmés |
| moderation-service | Python | 3014 | - | whispr-preprod / whispr-prod | Classifier NSFW/violence, sanctions, appeals review |
| mobile-app | React Native Expo | - | - | - | iOS, Android, Web PWA |

## Diagramme ASCII des communications inter-services

```
                          ┌─────────────┐
                          │  mobile-app │
                          │ (RN Expo)   │
                          └──────┬──────┘
                                 │ REST + WebSocket
              ┌──────────────────┼──────────────────┐
              │                  │                  │
              ▼                  ▼                  ▼
       ┌────────────┐   ┌──────────────────┐  ┌────────────┐
       │auth-service│   │messaging-service │  │user-service│
       │  :3010     │◄──│  :4010 / :40010  │─►│:3011/:50011│
       └────────────┘   └────────┬─────────┘  └─────┬──────┘
              ▲                  │                   │
              │ JWKS             │ pub/sub Redis      │ gRPC
              │                  ▼                   │
       ┌────────────┐   ┌──────────────────┐        │
       │calls-service│  │notification-svc  │        │
       │  :4012     │  │  :4011 / :40011  │        │
       └────────────┘   └──────────────────┘        │
              │                                      │
              │ gRPC :40010                          ▼
              └──────────────────────────►  ┌───────────────┐
                                            │media-service  │
                                            │    :3012      │
                                            └───────────────┘
                                                    │
                                            ┌───────────────┐
                                            │scheduling-svc │
                                            │    :3013      │
                                            └───────────────┘
                                                    │
                                            ┌───────────────┐
                                            │moderation-svc │
                                            │    :3014      │
                                            └───────────────┘
```

## Patterns de communication

### REST interne (INTERNAL_API_TOKEN)
- Toutes les communications service-à-service en REST utilisent le header `Authorization: Bearer <INTERNAL_API_TOKEN>`.
- Ce token est partagé via le secret k8s `shared-internal-api-token`.

### gRPC
- `user-service :50011` ↔ `messaging-service :40010` : vérification des contacts, résolution de profils.
- `calls-service` → `messaging-service :40010` : signaling LiveKit, métadonnées d'appel.
- `notification-service :40011` : réception des events de push depuis messaging-service.

### WebSocket Phoenix
- `messaging-service :4010` ↔ `mobile-app` : transport principal pour les messages en temps réel, réactions, presence.
- Canal Phoenix Channels (topics : `conversation:<id>`, `user:<id>`).

### Redis pub/sub
- `messaging-service` publie les events → `notification-service` consomme pour déclencher FCM/APNs.
- Base Redis partagée dans le namespace `redis`.

## Namespaces Kubernetes

| Namespace | Contenu |
|---|---|
| `whispr-preprod` | Tous les services Whispr en preprod |
| `whispr-prod` | Tous les services Whispr en prod |
| `postgresql` | Base de données PostgreSQL partagée |
| `redis` | Redis (cache + pub/sub) |
| `minio` | MinIO (stockage objets S3-compatible) |
| `argocd` | ArgoCD prod (apps `citadel-*`) |
| `argocd-preprod` | ArgoCD preprod (apps `preprod-*`) |

## Points d'attention

### WHISPR-1224 : `check_users_are_contacts` intermittent
Le check de vérification de contact dans messaging-service retourne `false` alors que la relation existe en DB (user-service). Root cause probable : race condition sur le cache Redis ou appel gRPC timeout. À investiguer avant la démo.

### WHISPR-838 à 842 : failles sécurité messaging-service
- CORS wildcard (`*`) sur les endpoints Phoenix.
- `check_origin: false` sur le WebSocket → n'importe quel domaine peut se connecter.
- Ces failles sont en attente de fix prioritaire.
