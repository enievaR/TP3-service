# Réponses aux questions - TP 3 Serveur TCP/IP

## 2.1 Cohérence concurrente et synchronisation
- **Problèmes de concurrence** : Conflits d'accès aux structures partagées (liste des canaux, utilisateurs).
- **Rejoindre/quitter un canal** : Risque d'incohérence (ex. : utilisateur ajouté/supprimé deux fois) sans synchronisation.
- **Vulnérabilités et solutions** : Conditions de course possibles. Solutions : verrous (mutex), sémaphores, structures thread-safe.

## 2.2 Modularité et séparation des responsabilités
- **Responsabilités fonctionnelles** : Gestion clients (connexion/déconnexion), commandes (/msg, /join), envoi messages, logs, persistance.
- **Séparation logique métier/réseau** : Oui, isoler logique métier (traitement commandes) de la gestion sockets (entrées/sorties).
- **Gestion erreurs** : Couche de traitement des commandes valide les entrées, renvoie erreurs via couche réseau.

## 2.3 Scalabilité et capacité à évoluer
- **Ajout commande** : Modifier module de traitement pour parser/exécuter nouvelle commande (ex. : /topic, /invite).
- **Centaines de clients** : Utiliser pool de threads/modèle asynchrone (select, epoll), optimiser mémoire, envisager load balancer.
- **Limitations** : Code monothreadé, structures non optimisées (listes linéaires), absence cache/persistance distribuée.

## 2.4 Portabilité de l'architecture
- **Adaptation HTTP** : Conserver logique métier, remplacer TCP par serveur HTTP (ex. : Flask), adapter protocole aux requêtes HTTP.
- **Modules micro-services** : Gestion utilisateurs, canaux, logs, persistance comme services indépendants via API.
- **Découplage utilisateurs/canaux** : Bases séparées, services dédiés communiquant via messages (ex. : Kafka, REST).

## 2.5 Fiabilité, tolérance aux erreurs, robustesse
- **Déconnexion brutale** : Détectable via erreurs socket (ex. : read/write échoue). Récupération : supprimer client de la liste.
- **Message non livré** : Détecter via exceptions socket (ex. : ECONNRESET), logger erreur, informer autres clients si besoin.
- **Traçabilité** : Logs pour enregistrer tentatives d'envoi/erreurs.

## 2.6 Protocole : structuration et évolutivité
- **Règles implicites** : Ligne = commande préfixée par / (ex. : /msg <dest> <texte>, /join <canal>). Arguments séparés par espaces.
- **Robustesse** : Gérer cas invalides (ex. : /msg sans texte renvoie erreur, /join canal invalide refuse accès) via validation.
- **Formalisation** : Possible avec ABNF : `COMMAND = "/" CMD SP (ARG *(SP ARG))`. Documenter commandes, arguments, réponses.
- **Différence REST/HTTP** : Protocole actuel stateful, textuel, TCP. REST/HTTP stateless, structuré (verbes GET/POST, JSON), URLs pour ressources.