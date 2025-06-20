# Synthèse technique – Limites du serveur IRC CanaDuck

## 1. Ce que le serveur fait bien ✅

- **Simplicité de conception** : tout est contenu dans un seul script, facile à lire et à exécuter.
- **Réactivité** : les messages sont transmis en temps réel entre les clients.
- **Multi-clients** : supporte plusieurs connexions TCP simultanées grâce aux threads (`socketserver.ThreadingTCPServer`).
- **État global** : l’ensemble des utilisateurs, canaux et connexions est centralisé dans une structure JSON (`etat_serveur`).
- **Journalisation** : chaque action importante est écrite dans un fichier log.
- **Persistance minimale** : sauvegarde de l’état via `etat_serveur.json` (facilement inspectable).

---

## 2. Ce que le serveur gère mal ou cache ⚠️

### 2.1 Couplage et architecture

- **Fort couplage entre protocole, logique métier et réseau** : une même fonction (ex. `envoyer_message`) traite l’état, écrit dans le réseau (`wfile`) et journalise l’action.
- **Absence d’interface explicite** : le protocole n’est pas séparé des actions ; il ne peut pas être remplacé sans réécrire le cœur du serveur.
- **Pas de modularité** : toute la logique repose sur des chaînes de caractères, ce qui rend la réutilisation ou l’évolution difficile.

### 2.2 Gestion des erreurs

- **Gestion d’erreurs rudimentaire** : seul un message générique « Commande inconnue » est envoyé pour les erreurs de syntaxe.
- **Aucune différenciation des erreurs** : pas de distinction entre erreurs de syntaxe, droits insuffisants, ou état illégal (ex. `/msg` sans canal).
- **Canaux fantômes et clients zombies** : pas de nettoyage automatique si un client quitte brutalement.

### 2.3 Robustesse et testabilité

- **Testabilité limitée** : il est difficile de tester la logique métier sans socket réelle, car tout dépend des entrées réseau.
- **Comportements implicites non testés** : comme la disparition d’un canal vide, ou la gestion d’un client malveillant.
- **Aucune tolérance aux pannes** : perte du fichier log, erreur réseau ou corruption de l’état non anticipées.

---

## 3. Ce qu’il faudrait refactorer pour évoluer vers un système web ou micro-services 🔁

### 3.1 Séparation des responsabilités

- **Isoler les modules métier** : détacher la logique de traitement (join, msg, alert…) de la réception réseau (IRCHandler).
- **Créer des interfaces métier claires** : `GestionUtilisateur`, `Canaux`, `Journalisation`, etc.
- **Découpler le protocole** : permettre d’avoir une version HTTP/WebSocket en branchant un adaptateur sans réécrire la logique.

### 3.2 Vers une architecture distribuée

- **Externaliser certaines fonctionnalités** : 
  - `/log` → microservice de log
  - `/alert` → microservice de notification
- **Persistance robuste** : remplacer le JSON par une base de données ou une message queue (RabbitMQ, Redis…).
- **Rendre l’état distribuable** : découper les canaux ou utilisateurs sur plusieurs services (scalabilité horizontale).

### 3.3 Pratiques devops et industrialisation

- **Conteneurisation (Docker)** pour l’environnement d’exécution
- **Logs structurés** (JSON ou autre) pour une lecture automatisable
- **Monitoring** via Prometheus/Grafana
- **API REST/GraphQL** pour les clients externes
- **Tests unitaires et d’intégration** avec simulation de sockets

---

## Conclusion

Le serveur actuel est un bon prototype de chat en réseau TCP, mais ses **limites de conception freinent sa robustesse et son évolutivité**. Une refonte modulaire et orientée services permettrait d’aller vers un modèle **scalable, maintenable et interopérable**, adapté à une architecture micro-services moderne.
