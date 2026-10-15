---
author: Paul
date: 2026-10-06
version: 1.0.0
modification: Harmonisation du write-up
cve: Aucun CVE produit applicable ; CWE-798
---

# Récupération du `user.txt`


/t/o/s/s/_/a/_/c/o/i/n/_/t/o/_/y/o/u/r/_/w/i/t/c/h/e/r/_/o/h/_/v/a/l/l/e/y/_/o/f/_/p/l/e/n/t/y*

jaskier:YouHaveTheMostIncredibleNeckItsLikeASexyGoose


# Récupération du `user.txt`

## 1 / Énumération des services

On commence par effectuer une énumération des ports et services disponibles avec Nmap.
```bash
nmap -sC -sV -p- <IP>
```


Le scan révèle notamment deux services intéressants :

* **HTTP** : serveur web
* **SSH** : service permettant une connexion distante

L'objectif est donc d'abord d'énumérer le serveur web afin de trouver une éventuelle information permettant d'accéder au SSH.

---

## 2 / Énumération du serveur web

En accédant au serveur HTTP, on obtient le message suivant :

```text
Toss a coin to your witcher
Don't fight it, you know you want to sing it, it might even help you out.
```

Ce message fait référence à la chanson **"Toss a Coin to Your Witcher"**, associée au personnage de **Jaskier** dans *The Witcher*.

L'indice semble donc nous encourager à utiliser les paroles de cette chanson pour poursuivre l'énumération.

---

## 3 / Énumération des chemins

En testant les différents éléments du texte sous forme de chemins, on obtient notamment :

```text
/t/o/s/s/_/a/_/c/o/i/n/_/t/o/_/y/o/u/r/_/w/i/t/c/h/e/r/_/o/h/_/v/a/l/l/e/y/_/o/f/_/p/l/e/n/t/y/
```

À chaque étape, le serveur renvoie le message :

```text
Do you know the lyrics? this one is pretty famous!
```

Cela indique que l'application attend probablement la suite correcte des paroles.

En continuant l'énumération jusqu'à obtenir la séquence complète, on obtient une nouvelle page contenant les informations suivantes après inspection du code source :

```text
jaskier:YouHaveTheMostIncredibleNeckItsLikeASexyGoose
```

On peut alors identifier les informations suivantes :

```text
Username : jaskier
Password : YouHaveTheMostIncredibleNeckItsLikeASexyGoose
```

---

## 4 / Connexion SSH

Le service SSH découvert lors de l'énumération initiale peut maintenant être utilisé avec les identifiants récupérés :

```bash
ssh jaskier@<IP>
```

On renseigne ensuite le mot de passe :

```text
YouHaveTheMostIncredibleNeckItsLikeASexyGoose
```

La connexion SSH fonctionne et nous obtenons un shell avec l'utilisateur `jaskier`.

---

## 5 / Récupération du `user.txt`

Une fois connecté, on vérifie le contenu du répertoire courant :

```bash
ls -la
```

Le fichier `user.txt` est présent.

On peut alors afficher son contenu avec :

```bash
cat user.txt
```

Le flag utilisateur est ainsi récupéré.

---

## Résumé

```text
Serveur HTTP
     │
     ▼
"Toss a coin to your witcher"
     │
     ▼
Indice vers les paroles de la chanson
     │
     ▼
Énumération des chemins
     │
     ▼
Identifiants récupérés
jaskier:<password>
     │
     ▼
Connexion SSH
     │
     ▼
Utilisateur jaskier
     │
     ▼
user.txt
```

## Remédiation

- Ne pas publier d'informations d'identification dans le contenu HTML ou dans des chemins prévisibles.
- Protéger les routes sensibles par une authentification et limiter les réponses aux informations nécessaires.
- Ne jamais réutiliser un mot de passe découvert ; le révoquer et le remplacer immédiatement.

## Reproductibilité

Le parcours est reproductible avec une cible configurée, son adresse IP et les commandes affichées dans chaque étape. Les réponses HTTP, les identifiants découverts et le résultat de la connexion SSH sont conservés dans les blocs de résultats ci-dessus.
