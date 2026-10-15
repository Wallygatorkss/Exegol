---
author: Wallygator
date: 2026-10-06
version: 1.0.0
modification: Harmonisation du write-up
cve: Aucun CVE produit applicable ; CWE-427
---

# Documentation : Batman's Secret (Partie 2 - Élévation de privilèges)

## Documentation : Batman's Secret (Partie 2 - Élévation de privilèges)

### 1. Objectif de la mission

Après avoir récupéré le premier flag (`user.txt`) sur la machine cible (`10.10.0.12`), l'objectif de cette seconde partie était d'effectuer une élévation de privilèges (*privilege escalation*) pour passer de l'utilisateur `bruce` à l'administrateur suprême (`root`) et récupérer le fichier ultime : **`root.txt`**.

---

### 2. Analyse initiale des privilèges (`sudo -l`)

* **La découverte de la faille de configuration** : En vérifiant les commandes exécutables avec les droits de l'administrateur via la commande `sudo -l`, nous avons découvert que l'utilisateur `bruce` pouvait exécuter un script spécifique sans mot de passe, avec la particularité de conserver la variable d'environnement `PYTHONPATH` :


```text
User bruce may run the following commands on gotham:
    (root) NOPASSWD: /usr/bin/python3 /opt/scanner/gotham_scanner.py

```


* **Analyse du script cible** : En récupérant le code source de `/opt/scanner/gotham_scanner.py`, nous avons identifié qu'il importait des modules Python dès son démarrage, notamment `time` et `re`.



---

### 3. Les échecs et leçons apprises (*Trial & Error*)

Notre parcours a été jalonné de plusieurs difficultés techniques riches en enseignements :

* **Le piège des redirections Bash (fichiers à 0 octet)** : Les tentatives d'injection de commandes avec des chevrons multiples (`>`) dans le sous-shell non interactif de la hotline (`alert.py`) entraient en conflit avec la syntaxe de redirection du serveur, créant des fichiers intermédiaires vides et vidant systématiquement le fichier de log `report.txt`.


* **Le piège des modules intégrés Python (*Built-in Modules*)** :
* Nos premières tentatives ciblaient le module `time`. Cependant, `time` est un module intégré codé en dur en C dans le cœur de Python, ce qui signifie que Python l'ignore totalement dans le `PYTHONPATH`.
* *La solution de réflexion* : Il fallait impérativement cibler un module standard écrit en Python, comme **`re`**, pour que le système obéisse au `PYTHONPATH` et charge notre faux fichier.





---

### 4. La démarche de résolution (*PYTHONPATH Hijacking*)

Pour exploiter cette configuration et exécuter notre code en tant que `root`, la méthode pas à pas a été la suivante :

1. **Création du faux module piégé** :
En écrivant un faux module `re.py` dans le dossier `/tmp` à l'aide de la commande `tee` pour éviter les erreurs de redirection Bash, nous y avons intégré l'exécution de la lecture du fichier protégé :


```bash
echo "import os; os.system(\"cat /root/root.txt\")" | tee /tmp/re.py
```[cite: 1]

```


2. **Exécution avec les privilèges Root** :
En forçant Python à charger notre module personnalisé via la variable d'environnement autorisée par sudo :


```bash
PYTHONPATH=/tmp sudo /usr/bin/python3 /opt/scanner/gotham_scanner.py
```[cite: 1]

```


3. **Récupération du résultat** :
Lorsque le script légitime a fait `import re`, Python a chargé notre faux module en mémoire avec les privilèges `root`, exécutant la commande et exfiltrant le contenu secret dans les rapports accessibles.



---

### 5. Flag Final

Le drapeau administrateur a été exfiltré avec succès :

* **Flag Root** : `EPI{8ruc3_W4yn3_15_7ru1Y_84tm4n}`

## 6. Contre-mesures

### `sudo` avec `PYTHONPATH` contrôlable

- Ne pas conserver `SETENV` pour une commande Python exécutée en `root`.
- Utiliser une règle sudo ciblée sans variables contrôlables et avec un script possédé par `root`, non modifiable par l'utilisateur.
- Lancer Python avec un environnement nettoyé et des imports provenant uniquement de chemins système maîtrisés.
- Éviter d'autoriser directement des interpréteurs ; préférer une action métier bornée et journalisée.