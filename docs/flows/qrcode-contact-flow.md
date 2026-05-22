# QR Code Contact Flow — Whispr Messenger

## Vue d'ensemble

Le QR code permet d'ajouter un contact en présentiel sans saisir de numéro de téléphone. Un lien signé est encodé dans le QR. Le scanner valide la signature HMAC-SHA256 avant de créer le contact.

Service : `user-service` (NestJS)
Librairie QR mobile : `react-native-qrcode-svg`

---

## 1. Générer son QR code

```
GET /user/v1/qr-code
Authorization: Bearer <accessToken>
```

Réponse :
```json
{
  "url": "whispr://add-contact?userId=<uuid>&token=<signed_token>",
  "expiresAt": "2026-05-22T16:00:00Z"
}
```

Le `token` est signé HMAC-SHA256 avec une clé secrète serveur et contient :
- `userId`
- `issuedAt` (timestamp)
- `expiresAt` (TTL court, typ. 24h)

Le token est à usage unique : une fois scanné et utilisé, il est invalidé.

---

## 2. Afficher le QR code

Le client passe l'URL dans `react-native-qrcode-svg` :

```jsx
import QRCode from 'react-native-qrcode-svg';

<QRCode
  value="whispr://add-contact?userId=abc&token=xyz"
  size={200}
/>
```

L'écran affiche également `expiresAt` pour informer l'utilisateur de la durée de validité.

---

## 3. Scanner un QR code

Navigation : **Contacts → icône scan** (coin supérieur droit)

L'écran de scan ouvre la caméra. Les permissions caméra doivent être accordées.

```
[caméra active]
     |
     | QR détecté
     v
decode URL → extraire userId + token
     |
     v
POST /user/v1/contacts/qr
{ "userId": "<uuid>", "token": "<signed_token>" }
```

---

## 4. Traitement côté serveur

```
POST /user/v1/contacts/qr
Authorization: Bearer <scannerAccessToken>
{
  "userId": "fab8817a-...",
  "token": "hmac_signed_token"
}
```

Vérifications effectuées par user-service :
1. Signature HMAC-SHA256 valide
2. Token non expiré (`expiresAt` > now)
3. Token non déjà utilisé (lookup Redis/DB)
4. `userId` existe en base
5. L'utilisateur qui scanne n'a pas bloqué la cible (et vice-versa)

Si toutes les vérifications passent :
- Création de la relation de contact (bilatérale)
- Token marqué comme consommé

Réponse succès :
```json
{
  "contact": {
    "userId": "fab8817a-...",
    "displayName": "Alice",
    "avatarUrl": "..."
  }
}
```

---

## 5. Cas d'erreur

| Erreur | Cause | Message affiché |
|--------|-------|-----------------|
| `TOKEN_EXPIRED` | `expiresAt` dépassé | "QR code expiré, demandez-en un nouveau" |
| `TOKEN_ALREADY_USED` | Token consommé | "Ce QR code a déjà été utilisé" |
| `TOKEN_INVALID` | Signature HMAC incorrecte | "QR code invalide" |
| `USER_NOT_FOUND` | `userId` inconnu | "Utilisateur introuvable" |
| `BLOCKED` | Blocage mutuel | "Impossible d'ajouter ce contact" |
| `ALREADY_CONTACT` | Relation déjà existante | "Contact déjà ajouté" |

---

## 6. Sécurité

- Signature HMAC-SHA256 empêche la falsification du `userId` ou du `token`
- TTL court (typ. 24h) limite la fenêtre d'utilisation d'un QR volé/screenshot
- Usage unique : un QR scanné ne peut pas être rejoué
- Le token ne contient pas d'informations sensibles — il identifie uniquement la demande

---

## Diagramme de séquence

```
Utilisateur A (affiche QR)       Utilisateur B (scanne)        user-service
        |                               |                           |
        |-- GET /qr-code -------------->|                           |
        |<-- { url, expiresAt } --------|                           |
        |                               |                           |
        | [affiche QR sur son écran]    |                           |
        |                               |                           |
        |                               |-- ouvre caméra            |
        |                               |-- scanne le QR            |
        |                               |-- decode URL              |
        |                               |                           |
        |                               |-- POST /contacts/qr ----->|
        |                               |   { userId, token }       |
        |                               |                           |-- verify HMAC
        |                               |                           |-- check TTL
        |                               |                           |-- check used
        |                               |                           |-- create contact
        |                               |<-- 200 { contact } -------|
        |                               |                           |
        |                               | [affiche profil de A]     |
```
