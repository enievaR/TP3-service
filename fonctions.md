
# Analyse fonctionnelle du serveur IRC - CanaDuck

## 1. Qui traite les commandes ?

- Les commandes sont traitées dans la méthode `handle()` de la classe `IRCHandler`, appelée par `StreamRequestHandler` pour chaque client.
- À chaque ligne reçue (`/nick`, `/join`, `/msg`, etc.), la commande est analysée avec `startswith()` puis déléguée à une méthode dédiée :
  - `/nick` → `set_pseudo()`
  - `/join` → `rejoindre_canal()`
  - `/msg` → `envoyer_message()`
  - etc.

Toutes les modifications d’état (utilisateurs, canaux, etc.) utilisent un **verrou partagé (`threading.Lock`)** via `with etat_serveur["lock"]`.

---

## 2. Où sont stockées les infos ?

### Canal courant d’un utilisateur

Le canal actuel d’un utilisateur est stocké dans le dictionnaire `etat_serveur["utilisateurs"]`, dans la sous-clé `"canal"` :

```python
etat_serveur["utilisateurs"][pseudo]["canal"] = canal
```

C’est la méthode `rejoindre_canal()` qui met à jour cette valeur.

---

### Flux de sortie (`wfile`) de chaque client

Chaque client possède un `wfile` (flux de sortie vers sa socket TCP), récupéré via l’héritage de `StreamRequestHandler`.

Dès qu’un utilisateur choisit un pseudo via `/nick`, on enregistre ce `wfile` :

```python
etat_serveur["utilisateurs"][pseudo] = {
    "canal": None,
    "wfile": self.wfile,
    "role": "user"
}
```

Cela permet ensuite d’envoyer des messages à n’importe quel client via son `wfile`, par exemple dans `envoyer_message()`.

---

### Liste des canaux et de leurs membres

Le dictionnaire `etat_serveur["canaux"]` contient une entrée par canal, avec une liste de pseudos inscrits :

```python
etat_serveur["canaux"] = {
    "#général": ["alice", "bob"],
    "#tech": ["charlie"]
}
```

Quand un utilisateur rejoint un canal, son pseudo est ajouté à la liste correspondante (`setdefault(...).append(...)`).

---

### Résumé visuel de la structure `etat_serveur`

```python
etat_serveur = {
    "utilisateurs": {
        "alice": {
            "canal": "#général",
            "wfile": <objet wfile>,
            "role": "user"
        },
        ...
    },
    "canaux": {
        "#général": ["alice", "bob"],
        "#tech": ["charlie"]
    },
    "lock": <verrou threading.Lock>
}
```

Toutes les écritures ou lectures dans cette structure sont protégées par :
```python
with etat_serveur["lock"]:
```

Cela évite les conditions de course entre threads lors des accès concurrents.

---

## 3. Qui peut planter ?

### Déconnexion sans `/quit`

Si le client ferme la socket sans envoyer `/quit`, `readline()` retourne une chaîne vide et on sort proprement de la boucle.

Le nettoyage est ensuite effectué :
- Le pseudo est supprimé de `etat_serveur["utilisateurs"]`
- Il est également retiré du canal auquel il appartenait

Mais **aucune vérification n'est faite pour supprimer un canal vide.**

---

### Échec d’un `write()`

- Tous les `wfile.write()` sont protégés par un `try/except`, mais les exceptions sont ignorées silencieusement.
- Aucun log ni retrait du client n'est effectué.

Cela peut entraîner des clients "fantômes" toujours présents dans l’état, mais plus accessibles.

---

### Canaux vides non nettoyés

Quand un utilisateur quitte un canal ou se déconnecte :
- Il est bien retiré de la liste du canal.
- Mais **le canal reste enregistré**, même s’il est vide.

Il faudrait ajouter une vérification pour le supprimer de `etat_serveur["canaux"]` si la liste devient vide.

