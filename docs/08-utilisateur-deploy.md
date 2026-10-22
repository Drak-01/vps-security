# Gestion des utilisateurs — compte de déploiement (Docker)

## 1. Objectif

Créer un compte dédié `deploy`, séparé de votre compte personnel, utilisé uniquement pour déployer les applications via Docker. Ce compte a une clé SSH propre, révocable indépendamment de la vôtre, et des droits limités au strict nécessaire.

> ⚠️ Point important à garder en tête tout du long : sur ce serveur, les applications tournent sous Docker. Comme expliqué en section 5, appartenir au groupe `docker` équivaut de fait à un accès root — ce guide en tient compte plutôt que de l'ignorer.

---

## 2. Créer l'utilisateur

```bash
sudo adduser deploy
```

`adduser` (plutôt que `useradd`) crée automatiquement le répertoire home et configure les permissions de base proprement.

---

## 3. Configurer l'accès SSH par clé

```bash
sudo mkdir -p /home/deploy/.ssh
sudo nano /home/deploy/.ssh/authorized_keys
# Collez la clé PUBLIQUE du système/service qui va se connecter (CI/CD, poste de déploiement...)

sudo chown -R deploy:deploy /home/deploy/.ssh
sudo chmod 700 /home/deploy/.ssh
sudo chmod 600 /home/deploy/.ssh/authorized_keys
```

Les permissions strictes (`700`/`600`) sont **obligatoires** : SSH refuse silencieusement une clé si les permissions du dossier `.ssh` ou du fichier sont trop ouvertes.

> Utilisez une clé **dédiée** à ce compte, générée spécifiquement pour le déploiement, jamais réutilisée ailleurs. Elle doit être facilement révocable : en cas de doute, il suffit de supprimer la ligne correspondante dans `authorized_keys`.

---

## 4. Ajouter le compte à la liste blanche SSH

Dans `/etc/ssh/sshd_config.d/hardening.conf`, ajouter `deploy` à côté de votre compte existant :

```
AllowUsers ubuntu deploy
```

```bash
sudo sshd -t
sudo systemctl restart ssh
```

---

## 5. Accès Docker : le point critique

Pour que `deploy` puisse lancer des conteneurs sans `sudo` à chaque commande, la pratique courante est de l'ajouter au groupe `docker` :

```bash
sudo usermod -aG docker deploy
```

**Attention** : être dans le groupe `docker` équivaut à un accès root complet à la machine, même sans jamais taper `sudo`. Un conteneur peut monter n'importe quel chemin de l'hôte, par exemple :

```bash
docker run -v /:/host -it alpine chroot /host sh
```

Ce n'est pas un bug, c'est le fonctionnement normal du daemon Docker (qui tourne lui-même en root). Aucune règle `sudoers` classique ne peut restreindre ça finement (les options `-v`, `--privileged`, `--entrypoint` permettent de contourner tout filtrage basé sur le nom de la commande).

### Décision à prendre

| Option | Niveau de sécurité | Effort |
|---|---|---|
| **A. Groupe `docker` classique** | Compromis acceptable si `deploy` est un compte dédié, avec clé propre et usage limité au déploiement | Faible (une commande) |
| **B. Docker rootless** | Isole Docker dans un namespace utilisateur, sans aucun privilège root | Plus élevé (réinstallation du daemon en mode rootless, adaptation des scripts) |

Ce guide part sur l'**option A**, cohérente avec le fait que `deploy` est déjà un compte à usage unique et surveillé — mais gardez cette limite en tête si l'usage du compte évolue.

```bash
sudo usermod -aG docker deploy
```

Vérifier que le socket Docker n'écoute qu'en local (jamais exposé sur le réseau) :

```bash
sudo systemctl status docker | grep -i listen
```

---

## 6. Droits `sudo` en dehors de Docker

`deploy` ne doit pouvoir faire, en dehors de Docker, **que** ce qui est strictement nécessaire au déploiement — par exemple recharger un reverse proxy hôte si vous en utilisez un devant vos conteneurs.

```bash
sudo visudo -f /etc/sudoers.d/deploy
```

```
deploy ALL=(root) NOPASSWD: /usr/bin/systemctl reload nginx
```

> `NOPASSWD` est pertinent si un pipeline CI/CD se connecte de façon non interactive — mais dans ce cas la sécurité repose entièrement sur la clé SSH (voir section 3). N'ajoutez ici aucune commande générique pouvant échapper vers un shell (`vim`, `less`, `bash -c`...).

Si le déploiement ne nécessite aucune commande root en dehors de Docker, cette étape est inutile : ne pas créer de fichier dans `sudoers.d` pour ce compte.

---

## 7. Restreindre l'accès au répertoire de déploiement

Si `deploy` doit aussi pousser des fichiers (configuration, fichiers statiques) sans passer par Docker pour tout :

```bash
sudo chown deploy:deploy /var/www/monapp
```

Permissions Unix classiques, aucune entrée `sudoers` nécessaire pour ce cas.

---

## 8. Vérifier

```bash
# Droits sudo effectifs (doit être vide ou minimal si l'essentiel passe par Docker)
sudo -l -U deploy

# Groupes du compte (doit inclure "docker")
groups deploy

# Connexion depuis la machine possédant la clé privée
ssh deploy@<votre_ip>

# Test docker sans sudo, une fois connecté en tant que deploy
docker ps
```

---

## 9. Points de vigilance

- **Groupe `docker` = accès root de fait.** Traitez ce compte avec la même vigilance qu'un compte root : rotation de clé en cas de doute, surveillance des connexions (`/var/log/auth.log`, `sudo.log`).
- **Clé SSH dédiée**, jamais réutilisée pour un autre usage ni un autre compte.
- **`NOPASSWD` en dehors de Docker** : à limiter à des commandes précises avec chemin absolu, jamais un binaire permettant un échappement shell.
- **Docker rootless** (option B) reste la solution la plus propre si ce compte doit un jour être partagé avec d'autres personnes ou automatisations moins maîtrisées.