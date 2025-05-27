# Routes API REST (API Gateway vers Services)

## Identity Service

- POST /api/v1/auth/register - Inscription d'un nouvel utilisateur
- POST /api/v1/auth/verify - Vérification de code SMS
- POST /api/v1/auth/login - Connexion utilisateur
- POST /api/v1/auth/refresh - Rafraîchir le token d'authentification
- POST /api/v1/auth/logout - Déconnexion utilisateur
- GET /api/v1/auth/devices - Lister les appareils connectés
- DELETE /api/v1/auth/devices/{deviceId} - Déconnecter un appareil spécifique
- POST /api/v1/auth/2fa/enable - Activer l'authentification à deux facteurs
- POST /api/v1/auth/2fa/disable - Désactiver l'authentification à deux facteurs
- POST /api/v1/auth/2fa/verify - Vérifier un code 2FA
- POST /api/v1/auth/recover - Initier la récupération de compte
- PUT /api/v1/auth/phone - Changer de numéro de téléphone

## User Service

- GET /api/v1/users/me - Obtenir son profil
- PUT /api/v1/users/me - Mettre à jour son profil
- PUT /api/v1/users/me/username - Modifier son nom d'utilisateur
- PUT /api/v1/users/me/picture - Mettre à jour sa photo de profil
- PUT /api/v1/users/me/privacy - Mettre à jour les paramètres de confidentialité
- GET /api/v1/users/search - Rechercher des utilisateurs
- GET /api/v1/users/{userId} - Obtenir le profil d'un utilisateur
- GET /api/v1/contacts - Lister ses contacts
- POST /api/v1/contacts - Ajouter un contact
- PUT /api/v1/contacts/{contactId} - Modifier un contact
- DELETE /api/v1/contacts/{contactId} - Supprimer un contact
- POST /api/v1/contacts/import - Importer des contacts
- POST /api/v1/contacts/block/{userId} - Bloquer un utilisateur
- DELETE /api/v1/contacts/block/{userId} - Débloquer un utilisateur
- GET /api/v1/contacts/blocked - Lister les utilisateurs bloqués

## Delivery Service

- GET /api/v1/conversations - Lister les conversations
- GET /api/v1/conversations/{conversationId} - Obtenir une conversation
- POST /api/v1/conversations - Créer une conversation
- PUT /api/v1/conversations/{conversationId} - Mettre à jour une conversation
- DELETE /api/v1/conversations/{conversationId} - Supprimer une conversation
- POST /api/v1/conversations/{conversationId}/pin - Épingler une conversation
- DELETE /api/v1/conversations/{conversationId}/pin - Désépingler une conversation
- POST /api/v1/conversations/{conversationId}/archive - Archiver une conversation
- DELETE /api/v1/conversations/{conversationId}/archive - Désarchiver une conversation
- GET /api/v1/conversations/{conversationId}/messages - Obtenir les messages d'une conversation
- POST /api/v1/messages - Envoyer un message
- PUT /api/v1/messages/{messageId} - Éditer un message
- DELETE /api/v1/messages/{messageId} - Supprimer un message pour soi
- DELETE /api/v1/messages/{messageId}/everyone - Supprimer un message pour tous
- POST /api/v1/messages/scheduled - Programmer un message
- PUT /api/v1/messages/scheduled/{messageId} - Modifier un message programmé
- DELETE /api/v1/messages/scheduled/{messageId} - Annuler un message programmé
- POST /api/v1/messages/{messageId}/pin - Épingler un message
- DELETE /api/v1/messages/{messageId}/pin - Désépingler un message
- POST /api/v1/messages/{messageId}/read - Marquer un message comme lu
- POST /api/v1/messages/{messageId}/reaction - Ajouter une réaction
- DELETE /api/v1/messages/{messageId}/reaction/{reactionId} - Supprimer une réaction

## Group Service (sous-partie du Delivery Service)

- POST /api/v1/groups - Créer un groupe
- GET /api/v1/groups/{groupId} - Obtenir les informations d'un groupe
- PUT /api/v1/groups/{groupId} - Mettre à jour un groupe
- DELETE /api/v1/groups/{groupId} - Supprimer un groupe
- POST /api/v1/groups/{groupId}/members - Ajouter des membres
- DELETE /api/v1/groups/{groupId}/members/{userId} - Supprimer un membre
- POST /api/v1/groups/{groupId}/leave - Quitter un groupe
- POST /api/v1/groups/{groupId}/admin/{userId} - Transférer les droits d'admin

## Media Service

- POST /api/v1/media/upload - Téléverser un média (multipart)
- POST /api/v1/media/hash-check - Vérifier un hash avant upload
- GET /api/v1/media/{mediaId} - Télécharger un média
- GET /api/v1/media/{mediaId}/preview - Obtenir l'aperçu d'un média
- DELETE /api/v1/media/{mediaId} - Supprimer un média
- GET /api/v1/media/storage - Obtenir les statistiques d'utilisation du stockage

## Notification Service

- PUT /api/v1/notifications/settings - Mettre à jour les paramètres de notification
- PUT /api/v1/notifications/settings/{conversationId} - Paramétrer les notifications d'une conversation
- POST /api/v1/notifications/mute/{conversationId} - Mettre en sourdine une conversation
- DELETE /api/v1/notifications/mute/{conversationId} - Réactiver les notifications d'une conversation
- PUT /api/v1/notifications/token - Enregistrer le token de notification push

## Moderation Service

- POST /api/v1/moderation/report - Signaler un contenu
- POST /api/v1/moderation/appeal/{decisionId} - Contester une décision de modération
