# Auth Flow — Whispr Messenger

## Vue d'ensemble

L'authentification Whispr repose sur un OTP SMS sans mot de passe. Un access token + refresh token sont émis à la connexion. La 2FA TOTP est optionnelle.

Service : `auth-service` (NestJS)
Base URL preprod : `https://whispr-preprod.roadmvn.com`

---

## 1. Login OTP

```
[Client]  POST /auth/v1/verify/login/request  { phoneNumber }
          <-- 200 OK  (SMS OTP envoyé)

[Client]  POST /auth/v1/verify/login/confirm  { phoneNumber, otp }
          <-- 200 OK  { verified: true, sessionToken }

[Client]  POST /auth/v1/login                 { phoneNumber, sessionToken, deviceName }
          <-- 200 OK  { accessToken, refreshToken, deviceId, userId, requires2FA }
```

En preprod, le code OTP de bypass est `123456` (var `OTP_BYPASS_CODE` dans le secret `auth-service-env`).
Un banner affiche ce code directement dans l'UI preprod.

### Rate limiting adaptatif

- 5 échecs OTP consécutifs sur une même IP → blocage temporaire
- Clé Redis : `adaptive-rate:/auth/v1/verify/login/confirm:<ip>`
- Le blocage est levé automatiquement après expiration du TTL

---

## 2. 2FA optionnel (TOTP)

Si la réponse du login contient `requires2FA: true` :

```
[Client]  affiche écran TOTP (code 6 chiffres depuis app authenticator)

[Client]  POST /auth/v1/2fa/verify  { userId, totpCode }
          <-- 200 OK  { accessToken, refreshToken, deviceId }
```

L'écran TOTP n'est présenté que si l'utilisateur a activé la 2FA dans ses réglages.

---

## 3. Refresh de token

```
[Client]  POST /auth/v1/tokens/refresh  { refreshToken }
          <-- 200 OK  { accessToken, refreshToken }
```

- Durée de vie refresh token : 30 jours glissants
- Cap absolu depuis `first_issued_at` : 90 jours
- Au-delà du cap, l'utilisateur doit se reconnecter

---

## 4. Logout

```
[Client]  POST /auth/v1/logout  { deviceId }
          <-- 200 OK
```

Révoque le refresh token et supprime l'appareil de la session active.

---

## 5. Gestion des appareils

```
GET /auth/v1/device          --> liste des appareils connectés
DELETE /auth/v1/device/:id   --> révoquer un appareil spécifique
```

- Maximum 10 appareils par compte
- Si le 11e appareil tente de se connecter : `pruneOldestDevice` supprime automatiquement le moins récent
- Chaque appareil a un `deviceId` unique associé à son refresh token

---

## 6. Signal Pre-Keys (chiffrement E2EE)

Au login, le replenisher vérifie le stock de pre-keys Signal disponibles côté serveur.
Si le nombre de one-time prekeys est sous le seuil configuré, le client upload un nouveau lot :

```
POST /auth/v1/signal/prekeys  { identityKey, signedPreKey, oneTimePreKeys: [...] }
```

Ceci se produit en arrière-plan, transparent pour l'utilisateur.

---

## Diagramme de séquence (login complet)

```
Client          auth-service        SMS gateway       Redis
  |                  |                   |               |
  |-- /verify/request -->                |               |
  |                  |-- send OTP ------>|               |
  |                  |<-- 200 OK --------|               |
  |<-- 200 OK -------|                   |               |
  |                  |                   |               |
  |-- /verify/confirm (otp) ------------>|               |
  |                  |-- check rate limit ------------->|
  |                  |<-- OK (or blocked) --------------|
  |                  |                   |               |
  |<-- 200 { verified, sessionToken } ---|               |
  |                  |                   |               |
  |-- /login (sessionToken) ------------>|               |
  |<-- 200 { accessToken, refreshToken, deviceId } ------|
```

---

## Erreurs courantes

| Code | Cause | Resolution |
|------|-------|------------|
| 401  | OTP invalide ou expiré | Redemander un OTP |
| 429  | Rate limit OTP atteint | Attendre l'expiration Redis |
| 403  | `requires2FA: true` non géré | Rediriger vers écran TOTP |
| 401  | Refresh token expiré (> 90j) | Reconnecter l'utilisateur |
