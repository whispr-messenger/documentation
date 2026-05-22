# Moderation & Sanctions Flow — Whispr Messenger

## Vue d'ensemble

Le système de modération combine une IA auto-flag (moderation-service Python) et une review manuelle via admin/modérateur. Les utilisateurs peuvent contester les sanctions via un flow d'appel.

Services impliqués :
- `moderation-service` (Python) — analyse IA des médias
- `user-service` (NestJS) — sanctions, appels, rôles
- `messaging-service` (Elixir) — signalement de messages

---

## 1. Signalement d'un message

Depuis l'interface mobile (long press sur un message) :

```
POST /messaging/api/v1/reports
{
  "messageId": "<uuid>",
  "reason": "spam" | "harassment" | "nsfw" | "violence" | "other",
  "comment": "Optionnel"
}
```

Le signalement est enregistré. L'émetteur du message n'est pas notifié. La review est placée dans la queue admin.

---

## 2. Auto-flag IA (moderation-service)

Au moment d'un upload média (`POST /media/v1/upload`), le media-service envoie le fichier au moderation-service Python pour analyse :

```
[media-service]
     |
     |-- POST moderation-service/analyze { mediaUrl, mediaId }
     |
     v
moderation-service (Python)
  - analyse NSFW (nudité, contenu adulte)
  - analyse violence / gore
  - retourne { flagged: true/false, category, confidence }
     |
     | flagged: true ?
     |
oui --+
     |
     v
messaging-service bloque le message côté destinataire
affiche : "Message bloqué par la modération automatique"
     |
     v
Propose à l'émetteur : "Contester cette décision ?"
```

Si `flagged: false`, le message est livré normalement.

---

## 3. Review admin

Accessible uniquement aux utilisateurs avec le rôle `admin` ou `moderator`.

### Queue de review

```
GET /user/v1/appeals/queue
Authorization: Bearer <adminToken>
```

Retourne la liste des signalements et flags en attente.

### Rôles

```
GET /user/v1/roles/me
```

```json
{
  "role": "admin" | "moderator" | "user"
}
```

- `admin` : accès complet (sanctions, review appels, stats)
- `moderator` : review uniquement (pas de sanctions directes)
- `user` : accès utilisateur standard

L'utilisateur admin de test "Alice Mod" a l'UUID `fab8817a-27a0-4537-89c1-be05f783150b` (configuré dans `ADMIN_USER_IDS` de messaging-service).

---

## 4. Sanctionner un utilisateur

```
POST /user/v1/sanctions
{
  "userId": "<uuid>",
  "type": "warn" | "temp_ban" | "ban",
  "reason": "Contenu NSFW répété",
  "duration": 86400
}
```

| Type | Description | `duration` |
|------|-------------|-----------|
| `warn` | Avertissement visible dans le profil | ignoré |
| `temp_ban` | Suspension temporaire | durée en secondes |
| `ban` | Bannissement permanent | ignoré |

L'utilisateur sanctionné reçoit une notification in-app et peut contester.

---

## 5. Flow de contestation (appeal)

### Déposer une contestation

Navigation : **Réglages → Mes sanctions → "Contester"**

```
POST /user/v1/appeals
{
  "sanctionId": "<uuid>",
  "reason": "Ce contenu n'est pas inapproprié car...",
  "evidence": "https://..." | null
}
```

### Review d'une contestation (admin/moderator)

```
PATCH /user/v1/appeals/:appealId
{
  "status": "approved" | "rejected",
  "reviewNote": "Contestation acceptée, le contenu respecte les CGU"
}
```

### Résultat si approuvée

- Sanction levée immédiatement
- L'utilisateur reçoit une notification : "Votre contestation a été acceptée"
- Accès restauré

### Résultat si rejetée

- Sanction maintenue
- L'utilisateur reçoit : "Votre contestation a été examinée et rejetée"
- `reviewNote` visible dans Mes sanctions

---

## 6. Lire ses sanctions et contestations

```
GET /user/v1/sanctions/me          --> liste des sanctions de l'utilisateur connecté
GET /user/v1/appeals/me            --> liste de ses contestations avec statuts
GET /user/v1/appeals/stats         --> stats globales (admin uniquement)
```

---

## Diagramme — cycle complet modération

```
[Utilisateur envoie un média]
          |
          v
    moderation-service analyse
          |
    flagged ?
          |
 non ----+----> livraison normale
          |
 oui      |
          v
    message bloqué côté destinataire
    émetteur notifié
          |
          | conteste ?
          |
 oui      v
    POST /appeals { sanctionId, reason }
          |
          v
    admin reçoit l'appel dans /appeals/queue
          |
    PATCH /appeals/:id { approved | rejected }
          |
    +-----+-----+
    |           |
approved      rejected
    |           |
sanction     sanction
levée        maintenue
    |           |
notif       notif
user        user
```

---

## Stats de modération (admin)

```
GET /user/v1/appeals/stats
```

```json
{
  "totalReports": 142,
  "pendingReview": 8,
  "totalSanctions": 23,
  "appealsApproved": 5,
  "appealsRejected": 12,
  "appealsPending": 3
}
```
