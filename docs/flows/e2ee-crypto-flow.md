# Flow E2EE - Chiffrement de Bout en Bout

## Architecture générale

Le chiffrement est intégralement côté client (mobile-app). Le serveur (messaging-service) stocke et transporte des blobs opaques - il ne voit jamais le contenu en clair.

**Bibliothèque** : NaCl box (libsodium via `tweetnacl` ou `libsodium-wrappers`).

**Format payload chiffré :**
```json
{
  "v": 1,
  "t": "whispr_e2ee_v1",
  "cipher": {
    "nonce": "<base64>",
    "box": "<base64>"
  }
}
```

## Flow d'envoi d'un message

```
Expéditeur (mobile-app)
        │
        │ 1. Charger ou générer l'identity key locale
        │    (SecureStore / localStorage prefixé)
        │
        │ 2. GET /auth/v1/signal/keys/:userId/devices
        │    → liste des devices du destinataire
        │
        │ 3. GET /auth/v1/signal/keys/:userId/devices/:deviceId
        │    → prekey bundle (identityKey, signedPreKey, oneTimePreKey)
        │
        │ 4. X3DH key agreement
        │    → calcul du sharedKey
        │
        │ 5. NaCl box encrypt(message, nonce, sharedKey)
        │    → cipher blob base64
        │
        │ 6. POST messaging-service avec payload JSON
        │    { "v":1, "t":"whispr_e2ee_v1", "cipher": {...} }
        ▼
messaging-service stocke le blob opaque
```

## Flow de réception

```
Destinataire (mobile-app)
        │
        │ 1. Réception message via WebSocket Phoenix
        │
        │ 2. E2EEService.isEncryptedPayload(content)
        │    → détecter si le message est chiffré
        │
        │ 3. Déchiffrer avec l'identity key locale
        │    NaCl box open(cipher.box, cipher.nonce, sharedKey)
        │
        │ 4. Mise à jour state via setMessages()
        │    → affichage en clair dans l'UI
        ▼
```

## Replenishment des prekeys

Le service `signalKeyReplenisher.ts` gère automatiquement le stock de one-time prekeys côté auth-service.

- Vérification au **login**.
- Vérification via **cron quotidien**.
- Si le nombre de prekeys disponibles passe sous le seuil → upload automatique :
  ```
  POST /auth/v1/signal/keys
  ```

## Stockage des clés

| Plateforme | Mécanisme | Clé de stockage |
|---|---|---|
| iOS / Android natif | `SecureStore` (Expo) | préfixe projet |
| Web PWA | `localStorage` | préfixe `SecureStore_` |

Les clés privées ne quittent jamais l'appareil.

## Fallback opportuniste

Si les clés du destinataire sont indisponibles au moment de l'envoi (prékeys épuisées, device non enregistré) :
- Le message est envoyé **en clair** sans erreur côté utilisateur.
- Comportement intentionnel pour ne pas bloquer l'UX, à revoir avant production.

## Médias E2EE

```
mobile-app (upload)
        │
        │ 1. encrypt fichier (NaCl secretbox)
        │    → chiffre le binaire, pas juste les métadonnées
        │
        │ 2. POST media-service → stocke le fichier chiffré (S3/MinIO)
        │    → retourne media_url
        │
        │ 3. Inclure dans le message :
        │    { media_url, media_key (base64), media_nonce (base64) }
        │    → ces métadonnées sont elles-mêmes chiffrées dans le payload E2EE
        ▼

mobile-app (réception)
        │
        │ 1. Déchiffrer le payload message → extraire media_key + media_nonce
        │ 2. GET media_url → télécharger le blob chiffré
        │ 3. Hook useE2EEMedia : NaCl secretbox open(blob, nonce, media_key)
        │ 4. Afficher le média déchiffré (image, vidéo, fichier)
        ▼
```

Le hook `useE2EEMedia` gère le déchiffrement à la volée avant affichage, sans stocker le contenu déchiffré sur disque.

## Statut actuel

- E2EE activé par défaut sur toutes les conversations directes (1v1).
- PR #238 : activation sur toutes les conversations.
- Groupes : chiffrement de groupe non implémenté (Signal's Sender Keys protocol = post-démo).
