# Profile Flow — Whispr Messenger

## Vue d'ensemble

La gestion du profil couvre la création initiale (premier login), les modifications, la confidentialité, les contacts et le blocage d'utilisateurs.

Services impliqués :
- `user-service` (NestJS) — profil, contacts, blocage, confidentialité
- `media-service` (NestJS) — upload avatar

Base URL preprod : `https://whispr-preprod.roadmvn.com`

---

## 1. Création du profil (premier login)

Après le premier login OTP réussi, le client détecte l'absence de `displayName` et redirige vers l'écran `ProfileSetupScreen`.

```
[écran ProfileSetupScreen]

PATCH /user/v1/profile
{
  "displayName": "Alice",
  "avatarMediaId": "<optional>"   // si photo uploadée
}
```

Ce PATCH est obligatoire pour accéder à l'application. La navigation vers l'accueil est bloquée tant que `displayName` est vide.

---

## 2. Upload d'un avatar

Avant de PATCH le profil avec un avatar, uploader le fichier via media-service :

```
POST /media/v1/upload
Content-Type: multipart/form-data

file=<image_data>
mediaType=avatar
```

Réponse :
```json
{
  "mediaId": "med_abc123",
  "url": "https://cdn.roadmvn.com/media/med_abc123",
  "thumbnailUrl": "https://cdn.roadmvn.com/media/med_abc123/thumb"
}
```

Utiliser ensuite `mediaId` dans le PATCH profil.

---

## 3. Modifier le profil

```
PATCH /user/v1/profile
{
  "displayName": "Alice B.",
  "bio": "Développeuse iOS",
  "avatarMediaId": "med_abc123"
}
```

Tous les champs sont optionnels — envoyer uniquement ce qui change.

### Lire le profil courant

```
GET /user/v1/profile/me
```

### Lire le profil d'un autre utilisateur

```
GET /user/v1/profile/:userId
```

Les champs retournés dépendent des réglages de confidentialité de l'utilisateur cible.

---

## 4. Confidentialité

```
PATCH /user/v1/privacy
{
  "lastSeen": "everyone" | "contacts" | "nobody",
  "profilePhoto": "everyone" | "contacts" | "nobody",
  "readReceipts": true | false
}
```

- `lastSeen` : contrôle qui voit la dernière connexion
- `profilePhoto` : contrôle qui voit l'avatar
- `readReceipts` : si `false`, les accusés de lecture ne sont ni envoyés ni reçus

---

## 5. Contacts

### Ajouter un contact par numéro

```
POST /user/v1/contacts
{ "phoneNumber": "+33612345678" }
```

Crée une relation bilatérale si le numéro est enregistré sur Whispr.
Si le numéro n'existe pas : `404 Not Found`.

### Lister ses contacts

```
GET /user/v1/contacts
```

### Supprimer un contact

```
DELETE /user/v1/contacts/:contactId
```

---

## 6. Bloquer un utilisateur

### Bloquer

```
POST /user/v1/blocked-users
{ "userId": "<uuid>" }
```

Effets du blocage :
- L'utilisateur bloqué ne peut plus envoyer de messages
- Les appels sont refusés (403 côté calls-service)
- Le profil n'est plus visible par l'utilisateur bloqué
- La relation de contact est suspendue

### Débloquer

```
DELETE /user/v1/blocked-users/:blockedUserId
```

### Lister les utilisateurs bloqués

```
GET /user/v1/blocked-users
```

---

## Diagramme — configuration initiale du profil

```
[Premier login OTP réussi]
          |
          v
    displayName vide ?
          |
    oui --+
          |
          v
    ProfileSetupScreen
    - saisir displayName
    - optionnel : prendre/choisir photo
          |
          | photo choisie ?
          |
    oui --+--> POST /media/v1/upload --> { mediaId }
          |
          v
    PATCH /user/v1/profile { displayName, avatarMediaId? }
          |
          v
    Redirection vers ConversationsListScreen
```

---

## Contraintes de validation

| Champ | Contrainte |
|-------|-----------|
| `displayName` | 1-50 caractères, non vide |
| `bio` | 0-150 caractères |
| Avatar | JPEG/PNG/WEBP, max 5 MB |
| `phoneNumber` | Format E.164 international |
