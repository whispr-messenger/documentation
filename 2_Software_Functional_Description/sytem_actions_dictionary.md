## Actions principalement traitées dans le frontend
### Interface et paramètres d'affichage

1. Formater le texte d'un message (gras, italique, etc.)
2. Prévisualiser un média
3. Marquer une conversation comme lue (localement)
4. Modifier la langue de l'application
5. Configurer le thème (clair/sombre)
6. Configurer la taille du texte
7. Trier sa liste de contacts (tri local)
8. Réorganiser les conversations épinglées (arrangement local)
9. Rechercher dans l'historique des messages déjà téléchargés
10. Consulter les médias déjà téléchargés
11. Ajouter une légende à un média avant envoi

#### Opérations cryptographiques (Signal)

12. Chiffrer les messages avant envoi
13. Déchiffrer les messages reçus
14. Générer des clés éphémères pour les sessions
15. Vérifier l'intégrité cryptographique des messages
16. Chiffrer les médias avant upload
17. Déchiffrer les médias après téléchargement
18. Générer le hash perceptuel d'un média pour modération
19. Vérifier les codes de sécurité entre appareils
20. Signer les messages avant envoi

### Stockage local

21. Stocker localement les messages pour consultation hors ligne
22. Gérer le cache des conversations
23. Gérer le stockage local des médias

## Actions nécessitant le backend

### Authentification et gestion de compte
24. S'inscrire avec un numéro de téléphone
25. Vérifier son compte via code SMS
26. Se connecter à l'application
27. Configurer l'authentification à deux facteurs
28. Changer de numéro de téléphone
29. Récupérer l'accès à son compte
30. Vérifier les appareils connectés à son compte
31. Déconnecter un appareil spécifique

### Gestion de profil et contacts (distribution)

32. Configurer/modifier sa photo de profil
33. Modifier son nom et prénom
34. Créer/modifier son nom d'utilisateur unique
35. Ajouter/modifier sa biographie
36. Configurer les paramètres de visibilité du profil
37. Importer des contacts depuis le téléphone
38. Rechercher un contact par numéro ou nom d'utilisateur
39. Ajouter un nouveau contact
40. Scanner un QR code pour ajouter un contact
41. Bloquer/débloquer un contact
42. Supprimer un contact

### Transmission de messages et médias

43. Envoyer un message texte (déjà chiffré côté client)
44. Transmettre un message à d'autres appareils du destinataire
45. Upload d'un média (déjà chiffré côté client)
46. Télécharger un média reçu
47. Programmer l'envoi d'un message pour plus tard
48. Supprimer un message pour tous (propagation)
49. Vérifier le statut d'envoi/lecture d'un message
50. Partager un média avec d'autres utilisateurs

### Gestion des groupes

51. Créer un nouveau groupe
52. Nommer le groupe et ajouter une photo
53. Ajouter des membres au groupe
54. Supprimer des membres du groupe
55. Quitter un groupe
56. Transférer les droits d'administration
57. Supprimer un groupe entier

### Notifications

58. Envoyer/recevoir des notifications push
59. Activer/désactiver les notifications pour une conversation
60. Mettre en sourdine temporairement une conversation
61. Configurer les notifications par catégorie

### Synchronisation et sauvegarde

62. Synchroniser les conversations entre appareils
63. Configurer les sauvegardes chiffrées
64. Restaurer depuis une sauvegarde

### Modération

65. Vérifier les hashs de médias contre la base de référence
66. Signaler un contenu inapproprié
67. Contester une décision de modération

### Gestion des PreKeys

68. Enregistrer de nouvelles PreKeys sur le serveur
69. Récupérer les PreKeys des destinataires
70. Rotation des clés d'identité
