# Sécurisation d'un serveur Linux

Documentation technique pour **mettre en place et durcir un serveur Linux**, qu'il s'agisse d'un **VPS** ou d'un **serveur physique**.

Chaque guide explique **pourquoi** un réglage existe (l'attaque qu'il bloque), **comment** l'appliquer sans casser l'accès au serveur, et **comment vérifier** qu'il fonctionne. Les exemples ciblent Debian/Ubuntu ; les différences avec RHEL sont signalées quand elles existent.

>  Documentation en cours de rédaction : de nouveaux guides sont ajoutés au fil de l'avancement.

---

## Sommaire

| # | Guide | Thème | Statut |
|---|---|---|---|
| 00 | [Préparation du serveur](docs/00-preparation.md) | Mises à jour, création et déploiement d'une clé SSH | ✅ |
| 01 | [Durcissement SSH](docs/01-ssh.md) | `sshd_config`, algorithmes crypto, audit `ssh-audit` | ✅ |
| 02 | [Durcissement PAM](docs/02-pam.md) | Mots de passe, verrouillage `faillock`, limites, `pam_access` | ✅ |
| 03 | [Durcissement du noyau (sysctl)](docs/03-sysctl.md) | Réseau, noyau, système de fichiers | ✅ |
| 04 | [Durcissement de `/tmp`](docs/04-tmp.md) | Options de montage `noexec,nosuid,nodev` | ✅ |
| 05 | [Durcissement de `sudo`](docs/05-sudo.md) | `sudoers`, moindre privilège, traçabilité | ✅ |
| 06 | [Bannière légale SSH](docs/06-banniere.md) | `/etc/issue.net`, `/etc/issue` | ✅ |
| 07 | [fail2ban](docs/07-fail2ban.md) | Bannissement automatique des IP | ✅ |
| 08 | [Compte de déploiement (Docker)](docs/08-utilisateur-deploy.md) | Utilisateur `deploy`, accès Docker | ✅ |
| 09 | [AppArmor](docs/09-apparmor.md) | Confinement des applications | 🚧 |

L'ordre de numérotation correspond à l'**ordre d'application recommandé** sur un serveur neuf.

---

## Règles communes à tous les guides

- **Ne jamais éditer les fichiers de configuration principaux** : déposer un fichier dédié dans le répertoire d'inclusion (`sshd_config.d/`, `sysctl.d/`, `sudoers.d/`, `jail.local`…). Les mises à jour du paquet n'écrasent pas vos réglages.
- **Sauvegarder avant de modifier** et **valider la syntaxe avant de redémarrer** (`sshd -t`, `visudo`, `sysctl -p`…).
- **Garder une session SSH ouverte** pendant les tests : une erreur de configuration ne doit jamais vous enfermer hors du serveur.
- **Vérifier après chaque étape** : chaque guide se termine par une section de vérification et des points de vigilance.

---

## Structure du dépôt

```
.
├── README.md        # Ce fichier : présentation et sommaire
├── docs/            # Guides numérotés, dans l'ordre d'application
│   ├── 00-preparation.md
│   ├── 01-ssh.md
│   └── ...
└── notes/           # Brouillons et analyses en cours, pas encore intégrés aux guides
```

### Ajouter un nouveau guide

1. Créer `docs/NN-sujet.md` (numéro suivant, nom en minuscules avec tirets).
2. Suivre le plan habituel :
   1. Objectif
   2. Installation / configuration
   3. Paramètres recommandés (tableau *paramètre — valeur — explication*)
   4. Vérification
   5. Ordre d'application recommandé
   6. Points de vigilance
3. Ajouter une ligne dans le [sommaire](#sommaire) ci-dessus.

---

## Avertissement

Ces guides sont fournis à titre informatif. Testez chaque modification sur un environnement non critique avant de l'appliquer en production : certains réglages peuvent couper l'accès au serveur ou casser des services existants (Docker, VPN, automatisations).
# vps-security
