# Messaging Flow — Whispr Messenger

## Vue d'ensemble

La messagerie temps réel repose sur Phoenix Channels (WebSocket) pour le transport et une API REST pour l'envoi/historique. Le chiffrement E2EE est automatique sur les conversations directes.

Service : `messaging-service` (Elixir/Phoenix)
Base URL preprod : `https://whispr-preprod.roadmvn.com`

---

## 1. Connexion WebSocket

```
wss://whispr-preprod.roadmvn.com/messaging/websocket?token=<jwt>
```

Après connexion, rejoindre le channel de conversation :

```js
// Phoenix JS client
socket.channel("conversation:<conversation_id>")
channel.join()
  .receive("ok", resp => { /* connecté */ })
  .receive("error", resp => { /* token invalide ou accès refusé */ })
```

Les événements entrants sur le channel :
- `new_message` — nouveau message reçu
- `message_deleted` — suppression pour tous
- `reaction_added` / `reaction_removed` — réactions
- `read_receipt` — accusé de lecture d'un autre membre
- `typing` — indicateur de frappe

---

## 2. Envoyer un message

```
POST /messaging/api/v1/conversations/:conversationId/messages

{
  "content": "Texte du message",
  "message_type": "text",
  "client_random": "uuid-v4-unique-par-envoi"
}
```

`client_random` garantit l'idempotence : un double-envoi réseau ne crée pas de doublon.

### Types de messages

| `message_type` | Description |
|----------------|-------------|
| `text`         | Texte brut ou chiffré |
| `media`        | Référence vers un mediaId (upload préalable via media-service) |
| `system`       | Événement système (join, leave, rename) — généré par le serveur |

---

## 3. Chiffrement E2EE

Le chiffrement est automatique sur toutes les conversations directes (1-to-1).

**Flux côté client :**
1. Récupérer la clé publique du destinataire (Signal pre-key bundle)
2. Dériver une clé partagée avec NaCl box
3. Chiffrer le contenu → blob opaque stocké tel quel côté serveur

**Format du blob chiffré :**
```json
{
  "v": 1,
  "t": "whispr_e2ee_v1",
  "cipher": {
    "nonce": "<base64>",
    "ciphertext": "<base64>"
  }
}
```

Le serveur ne déchiffre jamais ce blob. Il le stocke et le retransmet.

Les conversations de groupe ne sont pas E2EE dans la version actuelle.

---

## 4. Réactions

```
POST /messaging/api/v1/messages/:messageId/reactions
{ "reaction": "❤️" }

DELETE /messaging/api/v1/messages/:messageId/reactions/:reactionId
```

Une seule réaction par utilisateur par message. Changer de réaction remplace l'existante.

---

## 5. Supprimer un message

```
DELETE /messaging/api/v1/messages/:messageId?deleteForEveryone=true
```

- `deleteForEveryone=true` : supprime pour tous les membres (event `message_deleted` pushé sur le channel)
- Sans ce paramètre : suppression locale uniquement (côté expéditeur)
- Délai limite pour supprimer pour tous : configurable (défaut 10 min après envoi)

---

## 6. Accusés de lecture

```
POST /messaging/api/v1/conversations/:conversationId/read
{ "lastReadMessageId": "<message_id>" }
```

Marque tous les messages jusqu'à `lastReadMessageId` comme lus. Déclenche un event `read_receipt` sur le channel pour les autres membres.

---

## 7. Messages programmés

```
POST /scheduling/api/v1/scheduled-messages
{
  "conversationId": "<id>",
  "content": "Texte",
  "scheduledAt": "2026-05-23T09:00:00Z"
}
```

Service : `scheduling-service` (NestJS). Le message est injecté dans le flux messaging-service à l'heure prévue.

---

## 8. Conversations de groupe

```
POST /messaging/api/v1/conversations
{
  "type": "group",
  "name": "Nom du groupe",
  "member_ids": ["uuid1", "uuid2", "uuid3"]
}
```

- Limite : 256 membres par groupe
- Le créateur devient admin du groupe
- Actions admin : `PATCH /messaging/api/v1/conversations/:id` (rename, photo)
- Ajouter un membre : `POST /messaging/api/v1/conversations/:id/members`
- Quitter : `DELETE /messaging/api/v1/conversations/:id/members/me`

---

## 9. Historique des messages

```
GET /messaging/api/v1/conversations/:id/messages?before=<cursor>&limit=50
```

Pagination par curseur. `before` est le `message_id` du plus ancien message déjà chargé.

---

## Diagramme — envoi d'un message E2EE

```
Client A                messaging-service            Client B (WS connecté)
   |                          |                             |
   |-- GET pre-key bundle ---->|                             |
   |<-- { publicKey, ... } ----|                             |
   |                          |                             |
   | [chiffrement NaCl local] |                             |
   |                          |                             |
   |-- POST /messages { cipher blob } -->                    |
   |<-- 201 { messageId } ----|                             |
   |                          |-- push "new_message" ------>|
   |                          |   { cipher blob }           |
   |                          |                             |
   |                          |         [déchiffrement NaCl local]
```
