# Aether Royal – Documentation d’Architecture

## Vue d’ensemble

Ce document décrit le flux complet entre le client Unreal Engine, le backend AdonisJS, Agones et Quilkin pour :

* authentifier/inscrir un joueur en HTTPS
* allouer dynamiquement un GameServer
* sécuriser la communication UDP
* router correctement les paquets UDP vers le bon serveur
* notifier les joueurs en temps réel via WebSocket

L’architecture est conçue pour être :

* **Scalable** (allocation via Agones Fleet)
* **Sécurisée** (auth HTTPS + chiffrement UDP)
* **Temps réel** (WebSocket pour la coordination des matchs)
* **Isolée par match** (token + clé uniques par GameServer)

<br /><br />

---

<br /><br />

# 1) Authentification du joueur (HTTPS)

## Inscription (Sign Up)

Si le joueur n’a pas encore de compte, le client Unreal envoie une requête HTTPS avec :
* email
* username
* password

Le backend AdonisJS :
* crée l’utilisateur dans MariaDB
* hash automatiquement le mot de passe en base de donnée

Si l'inscription à réussi, le backend renvoie une response de type JSON :
```json
{
  message: 'Account created successfully'
}
```

Le client du jeu redirige automatiquement vers la "scène" : Login (après une inscription valide, pour qu'il se connecte)

<br /><br />

---

<br /><br />

## Connexion (Sign In)

Le client envoie une requête HTTPS avec :

* email/password
  ou
* username/password

Si les identifiants sont valides, le backend renvoie une response de type JSON :

```json
{
  token: {
    type: 'bearer'
    value: string
    expiresAt: string | null
  },
  quilkin_dns: string,
  quilkin_port: number,
}
```

Ce retour contient :

* Un token d’accès API (Bearer) pour les prochaines requêtes HTTPS qui demande d'être authentifier pour certaines routes.
* L’adresse publique de Quilkin :
  * DNS
  * Port UDP

À ce stade :

* Le joueur est authentifié
* Côté client on stock le : token, quilkin_dns, quilkin_port
* Le client du jeu redirige automatiquement vers la "scène" : Menu (après une connexion valide)

<br /><br />

---

<br /><br />

# 2) Le joueur clique sur le boutton du mode de jeu "Play Solo"

Quand le joueur clique sur le mode de jeu **Play Solo** :

Le client envoie une requête HTTPS au backend.

Le backend AdonisJS :
1. Middleware auth ou le backend check si le token bearer est valide avant de continuer.
2. Vérifie si un GameServer est déjà alloué pour ce mode de jeu, en regardant dans la base de donnée MariaDB.
3. Si oui :
   * Réutilise ce GameServer TANT QUE : il reste de la place et que la partie n'as pas encore commencer.
4. Sinon le backend AdonisJS :
   * Appelle le service `agones_allocator` pour allouer dynamiquement un GameServer dans la Fleet Agones.

<br /><br />

---

<br /><br />

# 3) Allocation du GameServer (Backend → Agones)

Lors de l’allocation, le backend génère **deux éléments critiques et uniques** pour ce match :
* `matchTokenBase64` (token de routage Quilkin, 16 bytes aléatoires encodés en base64)
* `udpEncryptionKeyBase64` (clé symétrique XChaCha20-Poly1305, 32 bytes, encodée en base64, utilisée pour chiffrer les paquets UDP client ↔ GameServer)

Ces deux valeurs sont :
* **Uniques par GameServer / match**
* Générées à chaque nouvelle allocation d'un GameServer
* Non réutilisées entre deux parties

Elles sont ensuite :
* Injectées dans les annotations Kubernetes du GameServer via Agones
* Stockées côté base de donnée (MariaDB) afin de :
  * permettre à d’autres joueurs de rejoindre le même match
  * leur transmettre exactement les mêmes valeurs

Exemple de structure retournée par le service : agones_allocation :
```ts
{
  matchTokenBase64: string
  udpEncryptionKeyBase64: string
}
```

<br /><br />

---

<br /><br />

# 4) Pourquoi stocker ces données en base de donnée

Le backend peut stocker en MariaDB :
* ID du GameServer
* matchTokenBase64
* udpEncryptionKeyBase64
* nombre de joueurs
* état du match (partie déjà commencer ou non)

Cela permet :

* aux prochains joueurs qui cliquent sur le mode de jeu "Play Solo" :
  * de rejoindre un GameServer existant
  * de recevoir le même token et la même clé
* d’éviter d’allouer un nouveau serveur à chaque clic

<br /><br />

---

<br /><br />

# 5) Transmission au client (WebSocket, pas HTTP)

⚠️ Important :
Les données de sécurité **ne sont pas renvoyées via la réponse HTTP**.

Pourquoi ?

* L’allocation n’est pas instantanée
* Le serveur peut être réutilisé
* Le backend gère l’état des matchs en continu

Le backend utilise donc un **WebSocket** pour notifier le client.

Quand le GameServer est prêt, le backend envoie au client via WebSocket :

```ts
{
  matchTokenBase64,
  udpEncryptionKeyBase64
}
```

Ce mécanisme permet :

* d’attendre que le serveur soit réellement alloué
* d’éviter de relancer une allocation à chaque clic
* de synchroniser plusieurs joueurs sur un même match

<br /><br />

---

<br /><br />

# 6) Rôle du matchTokenBase64

Le `matchTokenBase64` :

* Représente 16 bytes aléatoires
* Est unique par GameServer / match
* Est utilisé par Quilkin pour router les paquets UDP

Flux :

1. Le backend l’injecte dans :
   ```
   quilkin.dev/tokens
   ```
2. Le client le reçoit via WebSocket.
3. Le client décode le base64 → 16 bytes.
4. Il ajoute ces 16 bytes à la fin de chaque paquet UDP.

Quilkin :

* Lit le token
* Route vers le bon GameServer

<br /><br />

---

<br /><br />

# 7) Rôle du udpEncryptionKeyBase64 (XChaCha20Poly1305)

La clé de chiffrement UDP :

* Est générée aléatoirement par match
* Est unique pour chaque GameServer
* Sert à chiffrer la communication UDP client ↔ serveur

Elle est :

* envoyée au client via WebSocket
* injectée dans les annotations du GameServer
* récupérée côté GameServer via le SDK Agones

Ainsi :

* Le client chiffre le payload UDP
* Le GameServer le déchiffre avec la même clé

<br /><br />

---

<br /><br />

# 8) Format des paquets UDP

Chaque paquet envoyé par le client suit cette structure :

```
[ encrypted_payload ] + [ timestamp_8_bytes ] + [ token_16_bytes ]
```

* `encrypted_payload` :

  * données de jeu chiffrées avec XChaCha20-Poly1305
* `timestamp` :

  * utilisé pour les metrics et futures protections anti-replay
* `token` :

  * utilisé par Quilkin pour router le paquet

<br /><br />

---

<br /><br />

# 9) Résumé global du flux

1. Login HTTPS
   → récupération du token API + endpoint/portudp Quilkin

2. Le joueur clique sur "Play Solo"
   → requête HTTPS

3. Backend :

* réutilise un GameServer existant
  ou
* en alloue un nouveau via Agones

4. Backend stocke dans MariaDB :

* matchTokenBase64
* udpEncryptionKeyBase64

5. Backend notifie le client via WebSocket :

```
matchTokenBase64
udpEncryptionKeyBase64
```

6. Le client se connecte en UDP à Quilkin et :

* chiffre le payload
* ajoute timestamp + token

7. Quilkin route vers le bon GameServer.
