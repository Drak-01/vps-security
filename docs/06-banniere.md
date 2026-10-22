# Guide : banner légal SSH (`/etc/issue.net`)

## 1. Objectif

Afficher un message d'avertissement **avant authentification** sur toute connexion SSH. Ce banner a une valeur à la fois dissuasive (l'attaquant sait que l'accès est surveillé) et parfois juridique (preuve que l'accès non autorisé était explicitement signalé).

---

## 2. Créer le fichier

```bash
printf '%s\n' \
"AVERTISSEMENT" \
"" \
"Cet accès est réservé aux personnes autorisées." \
"Toute connexion et toute action effectuée sur ce système sont journalisées." \
"Toute tentative d'accès non autorisé constitue une infraction et sera signalée aux autorités compétentes." \
| sudo tee /etc/issue.net > /dev/null
```

> `printf '%s\n' ligne1 ligne2 ...` écrit chaque argument sur sa propre ligne — plus lisible qu'un seul long message, et évite les soucis d'échappement liés aux apostrophes/accents qu'on peut avoir avec `echo`.

---

## 3. Activer l'affichage côté SSH

Le fichier `/etc/issue.net` seul ne suffit pas : SSH ne l'affiche que si on le déclare explicitement.

```bash
sudo nano /etc/ssh/sshd_config.d/hardening.conf
```

```
Banner /etc/issue.net
```

Puis, comme pour toute modification `sshd_config` :

```bash
sudo sshd -t
sudo systemctl restart ssh
```

---

## 4. Vérifier

```bash
ssh -o "PreferredAuthentications=none" localhost
```

Cette commande force un échec d'authentification volontaire, ce qui permet de voir le banner s'afficher sans avoir besoin de se déconnecter réellement.

---

## 5. Faire de même pour les connexions locales (`/etc/issue`)

`/etc/issue` (sans `.net`) est un fichier différent, affiché sur les connexions **locales** (console physique/série), pas via SSH. Pour un message cohérent partout :

```bash
sudo cp /etc/issue.net /etc/issue
```

---

## 6. Points de vigilance

- Le banner s'affiche **avant** authentification : n'y mettre aucune information sur la version de l'OS, du noyau ou du serveur SSH, qui faciliterait le ciblage d'une vulnérabilité connue.
- Un banner à visée juridique doit refléter la politique réelle de l'organisation (mentions légales, RGPD si journalisation de données personnelles) — à faire valider si le serveur est en usage professionnel.
- Vérifier que `Banner` ne pointe pas déjà vers un autre fichier dans une configuration existante (`sshd_config` principal), pour éviter un conflit silencieux.