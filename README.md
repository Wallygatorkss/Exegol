# Exegol — Cybersecurity CTF Write-ups

![Status](https://img.shields.io/badge/status-active-success?style=flat-square)
![Focus](https://img.shields.io/badge/focus-cybersecurity-7b68ee?style=flat-square)
![Linux](https://img.shields.io/badge/platform-Linux-fcc624?style=flat-square&logo=linux&logoColor=black)
![Python](https://img.shields.io/badge/tools-Python-3776AB?style=flat-square&logo=python&logoColor=white)
![License](https://img.shields.io/badge/license-educational-lightgrey?style=flat-square)

> Une collection structurée de write-ups dédiés à l’analyse et à la résolution de challenges **Capture The Flag (CTF)** dans un environnement de laboratoire autorisé.

## Aperçu du projet

**Exegol** regroupe des comptes rendus techniques de parcours CTF réalisés dans le cadre d’un apprentissage pratique de la cybersécurité. Chaque write-up documente une chaîne complète d’investigation : reconnaissance de la cible, identification de la surface d’attaque, analyse de la vulnérabilité, exploitation contrôlée et récupération du flag.

Le dépôt sert à la fois de support de révision et de portfolio technique. Il met l’accent sur la compréhension du raisonnement derrière chaque étape, les erreurs rencontrées et les contre-mesures permettant de corriger les vulnérabilités étudiées.

> **Avertissement :** les techniques et commandes présentées sont destinées exclusivement à des machines de lab, des challenges CTF et des systèmes pour lesquels une autorisation explicite a été obtenue. Ne les exécutez pas contre une cible réelle sans autorisation.

## Fonctionnalités clés

- **Reconnaissance et énumération** : scans Nmap, identification de services, exploration de ports non standards et analyse des endpoints web.
- **Exploitation web et système** : injections SQL et commande, contournement d’authentification, exposition de dépôts Git, uploads dangereux et RCE.
- **Analyse de binaires et de code** : reverse engineering ELF, désassemblage, inspection de sections mémoire, décompilation de bytecode Python et analyse de fonctions XOR.
- **Fuzzing et recherche de contenu** : découverte de répertoires, fichiers cachés, ressources exposées et routes applicatives.
- **Documentation pédagogique** : commandes reproductibles, difficultés rencontrées, raisonnement d’exploitation et recommandations de remédiation.

## Organisation du dépôt

```text
Exegol/
├── Injection/       # Injections SQL/commande, RCE et contournements
├── cracking/        # Décodage, récupération et analyse de secrets
├── fuzz/            # Fuzzing web et découverte de ressources
└── reverse/         # Reverse engineering et analyse de binaires
```

### Parcours documentés

| Catégorie | Challenges |
| --- | --- |
| Injection | [Batman’s Secret](./Injection/Batman's_Secret.md), [Daemon Slayer](./Injection/Daemon_Slayer.md), [Fun With Functional](./Injection/Fun_With_Functional.md), [Year 5739](./Injection/Year_5739.md) |
| Cracking | [Yer a Wizard](./cracking/Yer_a_Wizard.md) |
| Fuzzing | [Grandline](./fuzz/Grandline.md), [H4ck3rz](./fuzz/h4ck3rz.md), [Toss a coin](./fuzz/Toss_a_coin.md) |
| Reverse engineering | [Chosen Undead](./reverse/Chosen_Undead.md), [Jormungandr](./reverse/jormungandr.md) |

Les fichiers suffixés `_2` ou `#2` correspondent à des variantes, des approfondissements ou des étapes complémentaires d’un même parcours.

## Prérequis

Le projet ne contient pas d’application à compiler ni de dépendances à installer : il s’agit d’une documentation Markdown. Pour reproduire les parcours dans un environnement de laboratoire, les outils suivants sont recommandés :

- un environnement Linux, une VM dédiée ou un conteneur isolé ;
- [Nmap](https://nmap.org/) pour la reconnaissance réseau ;
- `curl`, `netcat`, `ftp`, `ssh` et les utilitaires Unix courants ;
- [ffuf](https://github.com/ffuf/ffuf) et une wordlist adaptée pour le fuzzing web ;
- `objdump`, `strings`, `file`, `xxd` et un décompilateur pour l’analyse de binaires ;
- Python 3 pour les scripts d’analyse et de décodage.

## Installation

Cloner le dépôt :

```bash
git clone https://github.com/Wallygatorkss/Exegol.git
cd Exegol
```

Afficher la structure du projet :

```bash
find . -maxdepth 2 -type f -name '*.md' | sort
```

## Utilisation

### Consulter un write-up

Ouvrir directement le fichier souhaité dans un éditeur Markdown :

```bash
less 'Injection/Batman'"'"'s_Secret.md'
```

Ou démarrer un aperçu local avec un outil Markdown de votre choix, par exemple :

```bash
python3 -m http.server 8000
```

Puis ouvrir [http://localhost:8000](http://localhost:8000) dans un navigateur.

### Reproduire un parcours en laboratoire

1. Déployer la machine CTF dans un réseau isolé et relever son adresse IP.
2. Lire le write-up correspondant pour comprendre l’objectif et le contexte.
3. Remplacer l’adresse IP d’exemple par celle de votre cible de laboratoire.
4. Rejouer les étapes dans l’ordre, en validant les résultats intermédiaires.
5. Documenter toute différence de version, de service ou de comportement observée.

Exemple de première reconnaissance :

```bash
nmap -sC -sV -p- <IP_DE_LA_CIBLE>
```

Les adresses IP, identifiants et secrets éventuellement présents dans les write-ups appartiennent au contexte des labs documentés. Ils ne doivent jamais être réutilisés sur une cible non autorisée.

## Ce que j’ai appris / Défis techniques

### Compétences développées

- **Méthodologie d’audit** : passer d’une reconnaissance large à une hypothèse vérifiable, puis confirmer chaque étape avec des preuves techniques.
- **Énumération multi-services** : corréler les informations issues de HTTP, FTP, SSH et de services TCP personnalisés plutôt que d’analyser chaque service isolément.
- **Lecture de code et de binaires** : comprendre le flux d’exécution, relier le désassemblage aux données en mémoire et identifier les mécanismes de chiffrement faibles comme le XOR répétitif.
- **Analyse des erreurs** : distinguer une mauvaise piste d’un échec d’exploitation, vérifier le code réellement exécuté et ne pas se limiter aux informations historiques d’un dépôt exposé.
- **Reproductibilité** : transformer une résolution ponctuelle en procédure claire, avec commandes, résultats attendus, limites et hypothèses explicites.
- **Remédiation** : relier chaque vulnérabilité à une mesure défensive concrète : validation des entrées, suppression des secrets codés en dur, isolation de l’exécution et restriction des fichiers exposés.

### Défis rencontrés

Les parcours demandent notamment de repérer des services sur des ports inhabituels, de découvrir des fichiers cachés, de comprendre des applications volontairement trompeuses et de suivre une chaîne d’exploitation composée de plusieurs vulnérabilités. Certains challenges nécessitent également de faire la différence entre un artefact historique et le code réellement utilisé par le service.

Cette démarche a renforcé une approche essentielle en cybersécurité : **observer, formuler une hypothèse, la tester proprement, puis documenter aussi bien les réussites que les fausses pistes**.

## Contribution

Les suggestions d’amélioration sont les bienvenues. Pour contribuer :

```bash
git checkout -b docs/improve-writeup
# Modifier ou ajouter un write-up Markdown
git diff --check
git commit -m "docs: improve CTF write-up"
git push origin docs/improve-writeup
```

Avant de proposer une modification, vérifier qu’elle :

- reste limitée à un environnement légal et pédagogique ;
- explique le raisonnement, et pas uniquement la charge utile ;
- distingue clairement les observations, les hypothèses et les résultats ;
- ne divulgue pas de secret réel ou de donnée appartenant à un tiers.

## Licence

Ce dépôt est partagé à des fins éducatives et de portfolio. Les contenus et commandes doivent être utilisés uniquement dans un cadre autorisé.
