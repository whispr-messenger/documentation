--------------------------------------------------
WHISPER X STORM MICRO SERVICE
--------------------------------------------------

1. INTRODUCTION

Whisper X Storm Micro Service est une application de chat en temps réel conçue comme une initiative de recherche dans le cadre de la création d'une application mobile. L'objectif de ce projet est de pré-créer et d'éprouver la gestion d'un micro service de chat, tout en explorant différentes stratégies pour aboutir au micro service de chat ultime destiné à une application mobile de prochaine génération. L'application se compose d'un backend Ruby, responsable de la logique de chat, de l'authentification et de la gestion des salons (threads), et d'un front-end de bureau développé avec Electron. Ce document détaille la stack technique, l'architecture, les fonctionnalités, la sécurité, la performance et les perspectives d'évolution.

--------------------------------------------------
2. STACK TECHNIQUE

- Langage et Frameworks
  • Ruby (3.2) – Langage principal du backend.
  • Sinatra – Framework léger pour le serveur d'authentification.
  • websocket-driver – Gestion des connexions WebSocket pour le chat en temps réel.

- Base de données
  • PostgreSQL – Stockage des comptes utilisateurs dans la table "users", avec chiffrement des mots de passe via bcrypt.

- Conteneurisation
  • Docker – Permet le déploiement et la scalabilité du backend sur divers environnements (VPS, cloud).

- Client Desktop
  • Electron – Fournit une interface de bureau en HTML/CSS/JavaScript.
  • Webpack (prévu pour le futur) – Pour le bundling et la minimisation des assets front-end, améliorant ainsi les performances de chargement.

- Outils complémentaires
  • Bundler – Gestion des dépendances Ruby.
  • Electron Builder – Packaging et distribution du front-end.

--------------------------------------------------
3. ARCHITECTURE

3.1 Composants

- Backend (Storm Micro Service)
  • auth_app.rb (Sinatra) : API pour l'enregistrement et la connexion des utilisateurs via des commandes d'authentification intégrées (/register, /login).
  • server.rb (WebSocket) : Gère les connexions en temps réel et les interactions de chat via diverses commandes (création de salons, bannissement, transfert de rôle, etc.).
  • Controllers et Models
    - controllers/chat_controller.rb : Logique de traitement des commandes et gestion de la communication entre utilisateurs.
    - models/chat_room.rb : Gestion des salons (clients, historique, permissions, personnalisation, etc.).

- Base de données
  • PostgreSQL est utilisé pour stocker les comptes utilisateurs.

- Front-End (Whisper X Storm Micro Service Desktop)
  • Electron – Interface utilisateur de bureau avec HTML, CSS et JavaScript.
  • renderer.js et main.js gèrent la communication WebSocket et la logique d'affichage.
  • Optimisation future avec Webpack pour regrouper et minifier les fichiers, réduisant la taille des assets et accélérant le chargement.

3.2 Flux Global

1. Déploiement :
   - Le backend est containerisé avec Docker et déployé sur un VPS Debian.
   - Le front-end est distribué en tant qu'application Electron, se connectant au serveur WebSocket via l'IP publique du VPS.
2. Connexion :
   - Un utilisateur se connecte via le client Electron et saisit son pseudo ou se logge via des commandes d'auth (/register, /login).
3. Interaction :
   - Les commandes (ex : /cr, /cd, /ban, /powerto, /typo, /textcolor, etc.) sont traitées par le backend.
   - Les messages et commandes de personnalisation (changement de couleur, background, police, etc.) sont transmis en temps réel.
4. Authentification :
   - Des commandes intégrées (/register et /login) permettent de créer et de valider des comptes utilisateurs.
5. Optimisation et Minimisation :
   - Des améliorations futures incluront l'intégration de Webpack pour optimiser le bundling et la minimisation des fichiers front-end.

--------------------------------------------------
4. FONCTIONNALITÉS PRINCIPALES

- Gestion des Salons (Threads)
  • Création d'un nouveau thread via /cr <nom> <pass>, où le créateur devient automatiquement le propriétaire.
  • Changement de thread avec /cd <nom> <pass>, avec une transition automatique vers le nouveau thread.
  • Commande /info : affiche le nom du thread, le créateur et la liste des utilisateurs présents.
  • Commande /list : liste les utilisateurs connectés dans le thread.

- Commandes de Communication
  • /help : affiche la liste des commandes disponibles.
  • /history : affiche l'historique des messages avec des timestamps au format HH:MM.
  • /dm <pseudo> <message> : envoi d'un message privé à un utilisateur.
  • /quit : quitte le thread.

- Commandes de Personnalisation
  • /color <couleur> : change la couleur du pseudo de l'utilisateur.
  • /background <url> : change le fond pour tous les utilisateurs.
  • /typo <font> : change la police globale du chat.
  • /textcolor <couleur> : change la couleur globale du texte de l'application.

- Gestion des Rôles
  • Le créateur d'un thread peut bannir, expulser ou transférer le rôle via /ban, /kick et /powerto <pseudo>.

- Commandes d'Authentification
  • /register <email> <password> <pseudo> : crée un compte utilisateur en base.
  • /login <email> <password> : permet à un utilisateur de se connecter et de mettre à jour son pseudo dans le chat.

- Historique
  • Chaque thread conserve un historique en mémoire accessible via /history.

--------------------------------------------------
5. SÉCURITÉ

- Chiffrement des mots de passe
  • Utilisation de BCrypt pour stocker les mots de passe dans la table "users".
- Permissions et Rôles
  • Seul le créateur d'un thread peut effectuer des actions sensibles (ban, kick, changement de mot de passe, transfert de rôle).
- Validation et Authentification
  • Les commandes /register et /login assurent l'unicité des pseudos et la validité de l'authentification.
- Communication
  • La communication se fait actuellement en ws:// (non chiffré). Pour la production, il est recommandé d'utiliser WSS via un proxy HTTPS pour sécuriser les échanges.

--------------------------------------------------
6. PERFORMANCE

- Backend Ruby
  • Chaque connexion WebSocket est gérée par un thread, ce qui est adapté pour un nombre modéré d'utilisateurs (dizaines à centaines). Pour un volume plus important, un modèle événementiel ou un clustering serait envisagé.
- Optimisation Front-End
  • L'intégration future de Webpack permettra de regrouper et minifier les assets, réduisant ainsi la taille des fichiers et améliorant le temps de chargement.
- Base de Données
  • PostgreSQL est optimisé pour des opérations de lecture/écriture fréquentes, avec possibilité d'optimisations via indexation et caching.

--------------------------------------------------
7. DÉPLOIEMENT & STRATÉGIE

- Conteneurisation avec Docker
  • Le backend est containerisé avec Docker pour faciliter le déploiement et garantir la portabilité.
  • Un script start_hermes.sh permet de builder et lancer le conteneur.
- Orchestration
  • À l'avenir, un fichier Docker Compose pourra être utilisé pour orchestrer plusieurs services (backend, base de données, proxy).
- Monitoring et Redémarrage
  • Des outils tels que systemd ou des scripts de surveillance peuvent être mis en place pour assurer le redémarrage automatique en cas de crash.
- Minimisation et Bundling (Webpack)
  • Pour le front-end, Webpack sera intégré afin d'optimiser le bundling et la minimisation des fichiers, améliorant ainsi les performances.
- Initiative de Recherche
  • Ce projet sert à explorer et tester différentes stratégies de gestion d'un micro service de chat pour aboutir à la solution ultime, notamment dans le cadre du développement d'une application mobile.

--------------------------------------------------
8. PERSPECTIVES D'AMÉLIORATION

- Sauvegarde de l'Historique
  • Stocker l'historique des messages dans une base de données pour le rendre persistant lors des redémarrages.
- Sécurisation de la Communication
  • Passage à WSS (WebSocket Secure) via un reverse-proxy (par exemple, Nginx) pour sécuriser les échanges.
- Interface Utilisateur Améliorée
  • Intégrer un framework moderne (React, Vue.js) dans Electron pour une interface plus riche et interactive.
- Système de Rôles Avancé
  • Élargir le modèle de permissions pour inclure des modérateurs ou administrateurs globaux.
- Optimisation Front-End via Webpack
  • Intégrer Webpack pour regrouper et minifier les assets, optimisant ainsi le temps de chargement et la performance.
- Recherche et Développement
  • Tester différentes stratégies de gestion d'un micro service de chat pour aboutir à la solution ultime, en particulier pour le déploiement d'une application mobile.

--------------------------------------------------
9. CONCLUSION

Whisper X Storm Micro Service est une solution de chat en temps réel conçue comme une initiative de recherche pour explorer et perfectionner la gestion d’un micro service de chat. Grâce à son backend Ruby containerisé, à son système d’authentification intégré via commandes, et à son front-end Electron (avec des optimisations futures prévues via Webpack), l'application offre une plateforme évolutive et personnalisable. Le système se distingue par sa gestion avancée des rôles, sa personnalisation visuelle (couleur, background, police) et sa structure modulaire, ouvrant la voie à de futures améliorations pour créer le micro service de chat ultime dans le cadre du développement d'une application mobile de prochaine génération.

--------------------------------------------------
