# Laboratoire 5 — 6GEI311 Architecture des logiciels

**Équipe :** Cheikh Ahmadou Bamba Lô (LOXC30050300) — Babacar Diasse (DIAB11090309)

**Machines :**

| Rôle | Machine | Adresse IP | Port Flask |
|---|---|---|---|
| Membre A (application + nginx) | PC de Cheikh | 172.20.10.3 | 3000 |
| Membre B (copie de l'application) | PC de Babacar | 172.20.10.2 | 3001 |
| Distributeur de charge nginx | PC de Cheikh | 172.20.10.3 | 8181 |

---

## 1. Résumé de ce que nous avons appris

Dans ce laboratoire, nous avons déployé une petite application web Flask sur deux machines
différentes et nous l'avons exposée derrière un distributeur de charge (nginx) configuré en
*reverse proxy*.

- **Déploiement :** une application web n'est accessible que tant que son processus tourne et
  écoute sur un port. Avec `host='0.0.0.0'`, Flask écoute sur toutes les interfaces réseau,
  ce qui permet aux autres machines du réseau d'y accéder par l'adresse IP de la machine.
- **Partage du code :** GitHub nous a permis de distribuer le même code sur deux machines
  (`git push` par le membre A, `git clone` par le membre B), comme on le ferait pour déployer
  plusieurs instances d'un même service.
- **Distribution de charge :** le bloc `upstream labcluster` de nginx regroupe plusieurs
  instances de l'application. Par défaut, nginx utilise l'algorithme *round-robin* : chaque
  nouvelle requête est envoyée à tour de rôle à l'instance suivante. Le client ne connaît que
  l'adresse du distributeur (port 8181) et ignore quelle machine lui répond.
- **Tolérance aux pannes :** quand une instance tombe, nginx détecte l'échec de connexion et
  redirige la requête vers une instance encore disponible. Le service reste donc accessible
  tant qu'au moins une instance fonctionne.
- **Mise à l'échelle :** pour absorber plus de requêtes, il suffit d'ajouter une ligne
  `server IP:PORT;` dans l'`upstream`, sans modifier l'application ni l'adresse utilisée par
  les clients.

---

## 2. Réponses aux questions

### Étape 6 — On arrête l'application Python et on recharge `http://172.20.10.3:3000/`. Qu'est-ce qu'on remarque ?

La page ne s'affiche plus. Le navigateur affiche une erreur du type « Ce site est inaccessible /
connexion refusée » (`ERR_CONNECTION_REFUSED`). Comme le processus Flask est arrêté, plus aucun
programme n'écoute sur le port 3000 : la machine refuse la connexion. L'application n'existe
« sur le réseau » que tant que son serveur est en cours d'exécution.

### Étape 16 — On navigue vers `http://172.20.10.3:8181/` et on rafraîchit plusieurs fois. Qu'est-ce qu'on remarque ?

Le message affiché alterne entre les deux machines :

- `Hello, World! From BL: 172.20.10.3`
- `Hello, World! From BL: 172.20.10.2`

On accède toujours à la même adresse (celle de nginx, port 8181), mais nginx répartit les
requêtes à tour de rôle (*round-robin*) entre les deux serveurs déclarés dans
`upstream labcluster` (172.20.10.3:3000 et 172.20.10.2:3001). Les initiales restent les mêmes
parce que c'est le même code ; seule l'adresse IP change, ce qui permet de voir quelle machine
a traité chaque requête.

### Étape 17a — Après l'arrêt de l'application du membre A, est-elle accessible via `http://172.20.10.3:3000/` ?

**Non.** On obtient une erreur de connexion refusée, pour la même raison qu'à l'étape 6 :
plus aucun processus n'écoute sur le port 3000 de la machine A.

### Étape 17b — Est-elle toujours accessible via `http://172.20.10.3:8181/` ?

**Oui.** La page s'affiche toujours, mais toutes les réponses proviennent maintenant de la
machine B (`... : 172.20.10.2`). Quand nginx tente de joindre 172.20.10.3:3000, la connexion
échoue ; il considère alors ce serveur comme indisponible pendant un court moment et transmet
la requête au serveur suivant de l'`upstream`, 172.20.10.2:3001. Pour l'utilisateur, la panne
d'une instance est invisible : c'est la **tolérance aux pannes** apportée par le distributeur
de charge. (Le service ne deviendrait inaccessible que si les deux instances, ou nginx
lui-même, étaient arrêtées.)
