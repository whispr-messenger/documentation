# Notifications Flow — Whispr Messenger

## Vue d'ensemble

La stratégie de notification est en deux phases :
- **Maintenant (démo)** : foreground only, via WebSocket — pas de push système OS
- **Post-démo** : FCM (Android) + APNs (iOS) pour les notifications en background

Service : `notification-service` (Elixir/Phoenix)
Service user settings : `user-service` (NestJS)

---

## 1. Stratégie actuelle — Foreground only

Aucun push système OS n'est requis pour la démo Epitech mi-mai 2026.

Toutes les notifications passent par la connexion WebSocket Phoenix active. Si l'app est en background ou fermée, aucune notification n'est affichée.

---

## 2. In-app banner notification

Quand un message arrive via WebSocket et que l'utilisateur n'est pas sur l'écran de cette conversation :

```
[WS event "new_message" reçu]
         |
         v
InAppNotificationProvider
         |
         | écran actif != conversation source ?
         |
    oui --+
         |
         v
    affiche banner en haut de l'écran
    { displayName, aperçu du message (tronqué) }
         |
         | tap sur le banner ?
         |
    oui ---> navigate vers la conversation
```

Le banner disparaît automatiquement après ~3 secondes.
Si le message est E2EE, l'aperçu affiche "Message chiffré" sans déchiffrer.

---

## 3. Badge de messages non lus

Le compteur de messages non lus est maintenu en temps réel via Phoenix Presence sur le channel `user:<userId>`.

```
channel "user:<userId>"
event: "unread_count_updated"
payload: { conversationId: "...", unreadCount: 3, totalUnread: 7 }
```

Le composant `ConversationsListScreen` écoute cet event et met à jour les badges sans refetch HTTP.

---

## 4. Post-démo — Android FCM

Configuration requise dans le namespace `whispr-preprod` :

```
secret: notification-fcm-keyfile
  -> JSON Service Account Firebase (clé privée SA)
```

Activer le dispatcher dans la config notification-service :

```elixir
fcm_dispatcher: true
```

Vérification :
```bash
kubectl logs -l app=notification-service -n whispr-preprod | grep "fcm_dispatcher"
# attendu : fcm_dispatcher: enabled
```

Format payload FCM :
```json
{
  "to": "<fcm_device_token>",
  "notification": {
    "title": "Alice",
    "body": "Nouveau message"
  },
  "data": {
    "conversationId": "...",
    "type": "new_message"
  }
}
```

---

## 5. Post-démo — iOS APNs

Configuration requise dans le namespace `whispr-preprod` :

```
secret: notification-service-apns
  -> .p8 (clé privée APNs)
  -> KEY_ID
  -> TEAM_ID
```

Activer le dispatcher :

```elixir
apns_dispatcher: true
```

Vérification :
```bash
kubectl logs -l app=notification-service -n whispr-preprod | grep "apns_dispatcher"
# attendu : apns_dispatcher: enabled
```

Pour les appels entrants iOS en background, un ticket séparé couvre l'intégration VoIP PushKit (WHISPR-1174/1175).

---

## 6. Mute d'une conversation

```
POST /user/v1/notification-settings
{
  "conversationId": "<uuid>",
  "muted": true,
  "mutedUntil": "2026-05-29T00:00:00Z"  // optionnel, null = permanent
}
```

### Lire les réglages de notifications

```
GET /user/v1/notification-settings/:conversationId
```

### Démuter

```
POST /user/v1/notification-settings
{
  "conversationId": "<uuid>",
  "muted": false
}
```

Quand une conversation est mute :
- Le banner in-app n'est pas affiché
- Le badge reste visible mais n'incrémente pas l'icône OS
- Post-démo : les push FCM/APNs ne sont pas envoyés

---

## 7. Résumé des sources de notification par phase

| Phase | Background iOS | Background Android | Foreground |
|-------|---------------|-------------------|-----------|
| Démo (actuel) | - | - | WS banner |
| Post-démo | APNs + VoIP PushKit | FCM | WS banner |

---

## Vérification dispatchers

```bash
# Vérifier l'état des dispatchers en preprod
kubectl logs -n whispr-preprod -l app=notification-service --tail=100 \
  | grep -E "fcm_dispatcher|apns_dispatcher|dispatcher"
```

Résultat attendu post-activation :
```
[info] FCM dispatcher: enabled, project_id=whispr-prod
[info] APNs dispatcher: enabled, team_id=XXXXXXXXXX
```
