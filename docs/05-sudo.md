# Guide de durcissement de `sudo` / `sudoers`

## 1. Objectif

`sudo` est un point de passage obligé vers les privilèges root : mal configuré, il devient soit une porte dérobée (droits trop larges, pas de traçabilité), soit un moyen d'escalade de privilèges pour un compte compromis. L'objectif du durcissement est de :

- **limiter** ce que chaque utilisateur peut exécuter en root, au strict nécessaire ;
- **tracer** chaque usage de `sudo` (qui, quoi, quand) ;
- **fermer** les vecteurs classiques de contournement (variables d'environnement, TTY, chemins de binaires).

---

## 2. Principe : ne jamais éditer `sudoers` directement

Le fichier `/etc/sudoers` ne doit **jamais** être édité avec un éditeur classique (`nano`, `vi`) : une erreur de syntaxe peut casser `sudo` pour tout le monde, y compris root, sans possibilité de le corriger via `sudo` lui-même.

```bash
sudo visudo
```

`visudo` verrouille le fichier pendant l'édition et **vérifie la syntaxe avant d'enregistrer** — si une erreur est détectée, il propose de corriger avant de quitter, plutôt que de sauvegarder un fichier cassé.

Comme pour SSH/PAM/sysctl, on préfère un fichier séparé plutôt que d'éditer le fichier principal :

```bash
sudo visudo -f /etc/sudoers.d/hardening
```

> `/etc/sudoers.d/` est inclus automatiquement par la ligne `#includedir /etc/sudoers.d` présente en fin de `/etc/sudoers` par défaut sur Debian/Ubuntu et RHEL récents — à vérifier si ce n'est pas le cas.

---

## 3. Paramètres recommandés

| Directive | Valeur | Explication |
|---|---|---|
| `Defaults use_pty` | activé | Force l'exécution de toute commande `sudo` dans un pseudo-terminal dédié, ce qui permet de capturer fidèlement tout ce qui s'affiche (utile pour l'audit, empêche certains détournements de shell). |
| `Defaults logfile` | `/var/log/sudo.log` | Centralise les logs `sudo` dans un fichier dédié, en plus de `syslog`/`journalctl`, plus simple à surveiller et à archiver. |
| `Defaults timestamp_timeout` | `5` | Réduit la fenêtre pendant laquelle `sudo` ne redemande pas le mot de passe après une authentification réussie (défaut souvent 15 min). Limite le risque qu'une session laissée ouverte soit détournée. |
| `Defaults env_reset` | activé (déjà par défaut) | Réinitialise l'environnement à l'exécution de `sudo`, empêchant l'injection de variables (`LD_PRELOAD`, `PATH` piégé) depuis la session utilisateur. |
| `Defaults secure_path` | `/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin` | Force `sudo` à utiliser un `PATH` fixe et connu plutôt que celui de l'utilisateur, empêchant l'exécution d'un binaire piégé placé plus tôt dans un `PATH` personnalisé. |
| `Defaults requiretty` | activé (avec prudence) | Exige un terminal interactif réel pour utiliser `sudo`, bloquant certains scripts d'exploitation automatisés à distance. À tester : casse les usages `sudo` via cron ou certains outils d'automatisation (Ansible, CI/CD). |
| `Defaults passwd_tries` | `3` | Limite le nombre de tentatives de mot de passe avant échec, comme `MaxAuthTries` côté SSH. |
| `Defaults badpass_message` | message personnalisé | Optionnel, purement informatif. |
| `%admins ALL=(ALL) ALL` → remplacé par des règles ciblées | — | Éviter un accès root total par défaut ; préférer des règles précises par commande (section 4). |

### Exemple de fichier `/etc/sudoers.d/hardening`

```
Defaults use_pty
Defaults timestamp_timeout=5
Defaults env_reset
Defaults secure_path="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
Defaults passwd_tries=3
```

> `requiretty` volontairement omis de l'exemple par défaut : à n'ajouter que si aucun script/automatisation ne dépend de `sudo` en non-interactif sur ce serveur (voir section 6).

---

## 4. Limiter les droits par utilisateur (moindre privilège)

Éviter les règles larges type `alice ALL=(ALL) ALL` (équivalent à un accès root total). Préférer des règles précises par commande, dans un fichier dédié par utilisateur ou groupe :

```bash
sudo visudo -f /etc/sudoers.d/alice
```

```
# alice peut redémarrer les services et consulter les logs, rien d'autre
alice ALL=(root) /usr/bin/systemctl restart nginx, /usr/bin/systemctl restart php8.3-fpm
alice ALL=(root) /usr/bin/journalctl
```

### Bonnes pratiques associées

- **Utiliser des chemins absolus** pour chaque commande autorisée (`/usr/bin/systemctl`, pas `systemctl`), pour éviter qu'un binaire du même nom placé ailleurs dans le `PATH` ne soit exécuté à la place.
- **Éviter les wildcards larges** (`/usr/bin/*`) qui annulent l'intérêt du filtrage.
- **Se méfier des commandes qui permettent d'échapper vers un shell** (`vim`, `less`, `find -exec`, `awk`, certains éditeurs) : si elles sont autorisées sans restriction, elles permettent souvent de lancer un shell root malgré une règle en apparence restrictive. Vérifier chaque commande autorisée sur [GTFOBins](https://gtfobins.github.io/) avant de l'ajouter à une règle `sudo`.
- **Grouper par rôle** (`%deploy`, `%dba`, etc.) plutôt que par utilisateur individuel, pour garder les règles lisibles et maintenables.

---

## 5. Audit et traçabilité

Consulter les logs `sudo` :

```bash
sudo tail -f /var/log/sudo.log
# ou, si non configuré en fichier dédié :
sudo journalctl _COMM=sudo
```

Lister les droits effectifs d'un utilisateur :

```bash
sudo -l -U alice
```

Vérifier la syntaxe de l'ensemble des fichiers sudoers sans les modifier :

```bash
sudo visudo -c
```

---

## 6. Ordre d'application recommandé

1. Vérifier que `/etc/sudoers` inclut bien `#includedir /etc/sudoers.d`.
2. Créer `/etc/sudoers.d/hardening` avec les `Defaults` de la section 3 (sans `requiretty` dans un premier temps).
3. Vérifier qu'aucun script cron, CI/CD ou outil d'automatisation n'appelle `sudo` en root sur ce serveur avant d'envisager `requiretty`.
4. Remplacer progressivement les règles larges (`ALL=(ALL) ALL`) par des règles ciblées par commande (section 4), en commençant par les comptes les plus sensibles.
5. Chaque modification : `sudo visudo -f <fichier>` (jamais d'édition directe), qui valide la syntaxe automatiquement.
6. Tester avec `sudo -l -U <user>` que les droits obtenus correspondent bien à l'intention.
7. Surveiller `/var/log/sudo.log` après mise en place pour repérer d'éventuels refus inattendus.

---

## 7. Points de vigilance

- **Ne jamais éditer `sudoers` sans `visudo`** : une erreur de syntaxe peut rendre `sudo` totalement inutilisable, y compris pour le corriger.
- **Garder une session root ouverte** (console VPS du fournisseur, ou session `su` active) pendant les tests de restriction de droits, au cas où une règle serait trop restrictive.
- **`requiretty`** peut casser des automatisations existantes (cron, Ansible, CI/CD) — à valider avant activation.
- **Commandes permettant un échappement shell** (éditeurs, pagers, interpréteurs) : ne jamais les autoriser sans restriction stricte des arguments, sous peine d'annuler tout l'intérêt du filtrage par commande.
- **`timestamp_timeout` très bas** (0 ou négatif) redemande le mot de passe à chaque appel `sudo`, plus sûr mais plus contraignant — à équilibrer avec l'usage réel du serveur.