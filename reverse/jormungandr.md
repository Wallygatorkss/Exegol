---
author: Wallygator
date: 2026-10-06
version: 1.0.0
modification: Harmonisation du write-up
cve: Aucun CVE produit applicable ; CWE-798
---

# CTF — Mjollnir

## 1. Énumération

Après avoir obtenu un accès à la machine, on commence par rechercher les fichiers présents dans le répertoire courant.

On trouve notamment un fichier Python compilé :

```bash
mjollnir.pyc
```

L'extension `.pyc` indique qu'il s'agit d'un fichier contenant du **bytecode Python compilé**.

On trouve également un fichier :

```bash
.yggdrasil.cache
```

Ce fichier contient des données binaires encodées, mais le fichier intéressant pour la suite du challenge est `mjollnir.pyc`.

---

## 2. Analyse du fichier `mjollnir.pyc`

Pour observer le contenu brut du fichier, on peut utiliser `xxd` :

```bash
xxd mjollnir.pyc
```

### À quoi sert `xxd` ?

`xxd` permet d'afficher le contenu binaire d'un fichier sous forme hexadécimale.

Par exemple :

```text
00000000: 550d 0d0a 0000 0000 e516 e861 b108 0000
```

On retrouve généralement trois informations :

```text
00000000   550d 0d0a ...   ...
   │             │           │
   │             │           └── représentation ASCII
   │             └────────────── octets en hexadécimal
   └──────────────────────────── offset dans le fichier
```

`xxd` ne décompile pas le fichier. Il permet simplement d'observer les octets qui le composent.

---

## 3. Décompilation du `.pyc`

Comme `mjollnir.pyc` est du bytecode Python, on peut essayer de retrouver le code Python original avec un décompilateur.

On installe `decompyle3` :

```bash
pip install decompyle3 --break-system-packages
```

Le programme est installé dans :

```text
~/.local/bin/decompyle3
```

On peut donc lancer :

```bash
~/.local/bin/decompyle3 mjollnir.pyc
```

La décompilation permet de récupérer une représentation beaucoup plus lisible du programme.

---

## 4. Analyse du code

Le début du fichier contient :

```python
from Crypto.Util.number import bytes_to_long, long_to_bytes
```

On remarque surtout l'utilisation de :

```python
long_to_bytes()
```

Plus bas, deux variables sont définies :

```python
username = long_to_bytes(128672430374905338731062386)
password = long_to_bytes(33643248411776211125062560554130339746417025421355137140099645078910060553582)
```

Les identifiants ne sont donc pas écrits directement sous forme de chaînes de caractères.

Ils sont stockés sous forme de **grands entiers**, puis reconvertis en bytes avec `long_to_bytes()`.

---

## 5. Comprendre `long_to_bytes()`

La fonction :

```python
long_to_bytes()
```

fait l'opération inverse de :

```python
bytes_to_long()
```

Par exemple, si on possède les bytes :

```python
b"ABC"
```

ils peuvent être représentés par un entier.

`long_to_bytes()` permet ensuite de refaire le chemin inverse :

```text
entier
  ↓
long_to_bytes()
  ↓
bytes
  ↓
texte ASCII
```

Dans notre cas :

```python
long_to_bytes(128672430374905338731062386)
```

donne :

```text
b'jormungandr'
```

Et :

```python
long_to_bytes(33643248411776211125062560554130339746417025421355137140099645078910060553582)
```

donne :

```text
b'Jag_ar_Jormungandr_midgardsormen'
```

---

## 6. Récupération des identifiants

On obtient donc :

```text
Username : jormungandr
Password : Jag_ar_Jormungandr_midgardsormen
```

Le challenge utilise donc une **obfuscation simple** : les identifiants sont stockés sous forme d'entiers plutôt que directement sous forme de chaînes de caractères.

Il ne s'agit pas ici d'un chiffrement RSA à casser. Il suffit de comprendre comment les nombres sont reconvertis en bytes.

---

## 7. Récupération de User.txt

Il suffit donc de :
```bash
cat /home/jormungandr/user.txt
```



## 7. Résumé du cheminement

```text
mjollnir.pyc
     │ 
     ├── xxd
     │     └── observation du bytecode brut
     │
     ├── decompyle3
     │     └── récupération du code Python
     │
     ├── découverte de long_to_bytes()
     │
     ├── récupération des deux grands entiers
     │
     └── conversion entier → bytes
             │
             ├── username → jormungandr
             │
             └── password → Jag_ar_Jormungandr_midgardsormen
```

## 10. Identifiants récupérés

```text
Username : jormungandr
Password : Jag_ar_Jormungandr_midgardsormen
```

La principale difficulté du challenge était donc de comprendre que les identifiants étaient **encodés sous forme d'entiers** dans le bytecode et qu'ils pouvaient être récupérés directement grâce à `long_to_bytes()`.

## Remédiation

- Ne jamais stocker des identifiants dans un fichier `.pyc`, même sous forme d'entiers ou d'une autre obfuscation réversible.
- Retirer les fichiers de bytecode et caches du répertoire servi ou accessible par des utilisateurs non autorisés.
- Stocker les secrets dans un gestionnaire dédié et utiliser des mots de passe uniques, renouvelables et protégés.
- Limiter les permissions de lecture sur les fichiers applicatifs et journaliser les accès anormaux.

## Reproductibilité

Les commandes `xxd`, `decompyle3`, `long_to_bytes()` et la connexion SSH sont accompagnées de leurs résultats. Un tiers peut reprendre les étapes dans l'ordre sur la même cible en adaptant uniquement l'adresse IP.
