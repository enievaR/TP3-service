# Fiche de protocole – IRC CanaDuck

Complétez cette fiche pour décrire comment les clients interagissent avec le serveur IRC actuel.

## Format général

- Chaque commande est envoyée par le client sous forme d’une **ligne texte** terminée par `\n`.

## Commandes supportées

| **Commande**  | **Syntaxe exacte**        | **Effet attendu**                         | **Réponse du serveur**                                   | **Responsable côté serveur**     |
|---------------|----------------------------|--------------------------------------------|-----------------------------------------------------------|----------------------------------|
| `/nick`       | `/nick <pseudo>`           | Attribue un pseudo unique au client        | `Bienvenue, <pseudo> !` ou `Pseudo déjà pris.`            | `set_pseudo()`                   |
| `/join`       | `/join <canal>`            | Rejoint ou crée un canal                   | `Canal <canal> rejoint.`                                  | `rejoindre_canal()`              |
| `/msg`        | `/msg <texte>`             | Envoie un message sur le canal courant     | Message diffusé à tous les membres du canal               | `envoyer_message()`              |
| `/read`       | `/read`                    | Rappel que tout est en direct              | Message d'information                                     | `lire_messages()`                |
| `/log`        | `/log`                     | Affiche les 10 dernières lignes du log     | Lignes texte du fichier `serveur.log`                     | `lire_logs()`                    |
| `/alert`      | `/alert <texte>`           | Envoie une alerte globale                  | Diffusion + confirmation si autorisé                      | `envoyer_alerte()`               |
| `/quit`       | `/quit`                    | Déconnecte proprement le client            | `Au revoir !` (et log de la déconnexion)                  | `handle()`                       |

---


## Exemples d’interactions

### Exemple 1 : choix du pseudo

```
Client > /nick ginette
Serveur > Bienvenue, ginette !
```

### Exemple 2 : rejoindre un canal
```

Client > /join canard
Serveur > Canal canard rejoint.
 
```

### Exemple 3 : envoyer un message
```

Client >  /msg Salut tout le monde !
Serveur > (rien – les autres clients du canal reçoivent le message)

```

### Exemple 4 : message sans canal
```

Client >  /msg test
Serveur > Vous n'avez pas rejoint de canal.

```

### Exemple 5 : alerte non autorisée
```

Client >  /alert danger
Serveur > Vous n'avez pas les droits pour envoyer une alerte.
 
```

...

## Structure interne – Qui fait quoi ?

| Élément                      | Rôle dans l’architecture                                               |
|------------------------------|------------------------------------------------------------------------|
| `IRCHandler.handle()`        | Lit les commandes et appelle les fonctions métier                      |
| `etat_serveur`               | Stocke les utilisateurs, canaux, rôles et flux `wfile`                 |
| `log()`                      | Journalise chaque événement texte dans `serveur.log`                  |
| `broadcast_system_message()` | Envoie un message à tous les clients connectés                        |
| `ThreadingTCPServer`         | Serveur multi-clients basé sur des threads                            |
| `threading.Lock()`           | Protège les modifications concurrentes de `etat_serveur`              |
| `charger_etat()`             | Restaure l’état depuis `etat_serveur.json`                            |
| `sauvegarder_etat()`         | Sauvegarde l’état (sans `wfile`) à l’arrêt du serveur   

# Points de défaillance potentiels
---
> Complétez cette section à partir de votre lecture du code.

## Points de défaillance potentiels

| **Zone fragile**                | **Cause possible**                         | **Conséquence attendue**                             | **Présence de gestion d’erreur ?** |
|---------------------------------|--------------------------------------------|------------------------------------------------------|-------------------------------------|
| `wfile.write(...)`              | Déconnexion du client                      | Exception, plantage ou diffusion partielle           | Partiellement géré (`try/except`)   |
| Modification d’`etat_serveur`   | Concurrence entre threads                  | Incohérence ou conflit d’état                        | Oui, protégé par `threading.Lock()` |
| Lecture du fichier log          | Fichier absent, corrompu                   | Message d'erreur au client                           | Oui (message d’erreur explicite)   |
| Pseudo déjà pris (`/nick`)      | Collision entre deux clients               | Rejet du second client avec message explicite        | Oui                                |
| Utilisateur sans canal courant  | `/msg` sans `/join`                        | Message d’erreur envoyé                              | Oui                                |
| Déconnexion sans `/quit`        | Fermeture de la socket (client muet)       | Nettoyage manuel dans `finally`, mais peut rater     | Partiellement géré                 |
| `/alert` par utilisateur non autorisé | Absence de rôle `admin/moderator`     | Refus avec message d’erreur                          | Oui                                |

---


## Remarques ou cas particuliers

- Les commandes sont envoyées **en texte brut**, sans typage ni validation formelle.
- L’état du serveur est **centralisé** et non distribué → limite la scalabilité.
- Les erreurs ne sont **pas catégorisées** (syntaxique vs logique vs technique).
- Les rôles (`admin`, `moderator`) sont définis en dur, sans gestion d’authentification.
- Les flux clients (`wfile`) ne sont **pas sérialisables** → perte lors de la persistance.
- Le protocole ne fournit **aucun code de réponse standardisé**.
