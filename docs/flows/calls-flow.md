# Calls Flow — Whispr Messenger

## Vue d'ensemble

Les appels audio/vidéo utilisent LiveKit (WebRTC SFU) comme media server, avec `calls-service` (Elixir) pour la signalisation. Les appels sont disponibles uniquement sur iOS et Android natif — le web est désactivé.

Service signalisation : `calls-service` (Elixir/Phoenix)
Media server : LiveKit (auto-hébergé sur Citadel)
Base URL preprod : `https://whispr-preprod.roadmvn.com`

---

## 1. Initier un appel

```
POST /calls/api/v1/calls
{
  "conversation_id": "<uuid>",
  "type": "audio" | "video"
}
```

Réponse :
```json
{
  "callId": "<uuid>",
  "livekitToken": "<jwt>",
  "livekitUrl": "wss://livekit.roadmvn.com",
  "roomName": "<generated>"
}
```

Le caller utilise `livekitToken` pour rejoindre la room LiveKit côté SDK natif.

---

## 2. Flux complet d'un appel 1v1

```
Caller                  calls-service           Callee (WS connecté)       LiveKit
  |                          |                        |                       |
  |-- POST /calls ---------->|                        |                       |
  |<-- { livekitToken, url } |                        |                       |
  |                          |-- push "incoming_call" -->                     |
  |                          |   { callId, callerId, type }                   |
  |                          |                        |                       |
  |-- JOIN LiveKit room --------------------------------------------->|       |
  |                          |                        |                       |
  |                          |                        | [accepte ou refuse]   |
  |                          |<-- POST /calls/:id/accept (callee) ----        |
  |                          |-- push "call_accepted" -->                     |
  |                          |                        |                       |
  |                          |                        |-- JOIN LiveKit ------>|
  |                          |                        |                       |
  |  <============ media audio/video peer-to-peer via LiveKit ===============>|
```

---

## 3. Appel de groupe

Même flow que le 1v1, avec plusieurs participants qui rejoignent la même room LiveKit.
Les participants rejoignent à leur propre rythme — pas de synchronisation forcée.

```
POST /calls/api/v1/calls
{
  "conversation_id": "<group_conversation_id>",
  "type": "video"
}
```

Chaque participant reçoit son propre `livekitToken` (scoped à la room).

---

## 4. Refuser un appel

```
POST /calls/api/v1/calls/:callId/decline
```

Envoie un event `call_declined` au caller via WS. La room LiveKit reste ouverte un court délai puis est fermée si personne ne rejoint.

---

## 5. Terminer un appel

```
POST /calls/api/v1/calls/:callId/end
```

- Ferme la room LiveKit
- Push `call_ended` à tous les participants WS
- Enregistre la durée dans l'historique de la conversation

---

## 6. Restrictions

### Web désactivé

Les modules natifs LiveKit (`livekit-client` SDK React Native natif) ne sont pas disponibles sur le build web/PWA. Toute tentative d'appel depuis l'interface web affiche une erreur :

> "Les appels sont disponibles uniquement sur l'application mobile iOS/Android."

### Blocage mutuel

Si l'un des participants a bloqué l'autre (ou inversement), l'API `/calls` retourne `403 Forbidden`.
Le caller voit un message générique "Appel impossible".

---

## 7. Permissions mobiles requises

| Plateforme | Permission | Moment |
|------------|-----------|--------|
| iOS        | `NSMicrophoneUsageDescription` | Premier appel audio |
| iOS        | `NSCameraUsageDescription` | Premier appel vidéo |
| iOS        | VoIP pushkit (post-démo) | Appels entrants en background |
| Android    | `RECORD_AUDIO` | Premier appel audio |
| Android    | `CAMERA` | Premier appel vidéo |

---

## 8. État d'un appel

```
GET /calls/api/v1/calls/:callId
```

```json
{
  "callId": "...",
  "status": "ringing" | "ongoing" | "ended" | "missed",
  "type": "audio" | "video",
  "startedAt": "...",
  "endedAt": "...",
  "duration": 125
}
```

---

## Architecture LiveKit (schéma simplifié)

```
[iOS/Android App]                [LiveKit SFU]
  LiveKit SDK natif  <-- WebRTC media -->  [Room]
                                              ^
                                              |
[iOS/Android App]                            |
  LiveKit SDK natif  <-- WebRTC media ------>|
```

calls-service ne transporte pas de media. Il gère uniquement :
- La création/fermeture des rooms LiveKit
- La génération des tokens LiveKit signés
- La signalisation (invite, accept, decline, end) via Phoenix Channels
