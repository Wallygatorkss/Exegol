---
author: Wallygator
date: 2026-10-06
version: 1.0.0
modification: Harmonisation du write-up
cve: CVE-2023-25577 applicable seulement à Werkzeug < 2.2.3, version non documentée ; CWE-94
---

# Documentation : Fun With Functional

### Documentation : Fun With Functional

#### 1. Phase de Reconnaissance

La première étape consiste à analyser la cible à l'aide d'un scan `nmap` sur l'adresse IP `10.10.0.6` pour identifier les ports ouverts et les services actifs :
    
* **Port 22 (SSH)** : OpenSSH sur Debian.


* **Port 5001 (HTTP)** : Un serveur web propulsé par **Werkzeug / Python**.


* **Indice textuel** : La page d'accueil affiche un message orienté sur la programmation fonctionnelle (*« Why bother yourself with security when you can code in Haskell ? »*) et oriente vers la route `/homework`.



---

#### 2. Investigation et Analyse de la route `/homework`

En accédant à l'application web, la page des devoirs révèle son fonctionnement :

* **Le mécanisme** : Un formulaire d'upload (`multipart/form-data`) qui accepte les fichiers portant l'extension `.hs` (Haskell).


* **Le traitement** : L'énoncé indique que le fichier soumis est transmis à un outil d'autocorrection (« *autocorrect tool* »).



---

#### 3. Analyse de la Vulnérabilité (RCE via Haskell)

Derrière cette interface se cache un comportement d'exécution dynamique :

* Le serveur distant prend le fichier `.hs` téléchargé, le compile à l'aide du compilateur GHC (`ghc`), puis exécute le binaire généré.


* La sortie standard (`stdout`) et les messages de compilation sont renvoyés directement dans la réponse HTML de la page web.


* Cela conduit à une vulnérabilité critique d'**Exécution de Code à Distance (RCE)**.



En soumettant un script Haskell basique utilisant le module `System.Cmd` pour exécuter une recherche système, nous avons pu localiser immédiatement le fichier recherché :

* **Chemin identifié** : `/home/prof/user.txt`.



---

#### 4. Exploitation et Résolution Finale (`read_flag.hs`)

Pour récupérer le contenu du fichier cible, un script Haskell spécifique (`read_flag.hs`) a été conçu pour exécuter la commande `cat` :

```haskell
import System.Cmd (system)

main :: IO ()
main = do
    system "cat /home/prof/user.txt"
    return ()
```

## 5. Contre-mesures

### Compilation et exécution de code fourni par l'utilisateur

- Ne jamais compiler puis exécuter directement un fichier envoyé par un utilisateur sur le serveur principal.
- Si cette fonctionnalité est indispensable, utiliser un bac à sable isolé : utilisateur non privilégié, réseau désactivé, limites CPU/mémoire/temps et système de fichiers minimal en lecture seule.
- Utiliser une liste blanche de fonctionnalités au lieu d'autoriser les imports et appels système arbitraires.
- Ne pas renvoyer les détails du compilateur au client ; journaliser côté serveur et limiter les informations d'erreur.