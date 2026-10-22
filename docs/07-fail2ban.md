# Guide de configuration de fail2ban

## 1. Objectif

`fail2ban` surveille les journaux (SSH, mais aussi web, mail, etc.) et bannit automatiquement, via le firewall, les adresses IP qui multiplient les échecs d'authentification. C'est un complément indispensable au firewall (UFW/nftables) et à `pam_faillock` :

- **UFW/nftables** filtre par port, sans notion de comportement.
- **`pam_faillock`** bloque un *compte* après des échecs, mais laisse l'attaquant retenter avec un autre login.
- **`fail2ban`** bloque la *source* (l'IP) au niveau réseau, quel que soit le compte ciblé.

---

## 2. Installation

```bash
sudo apt update
sudo apt install fail2ban
```

Activer au démarrage :

```bash
sudo systemctl enable --now fail2ban
sudo systemctl status fail2ban
```

---

## 3. Principe : ne jamais éditer les fichiers `.conf`

Comme pour `sshd_config`, les fichiers `/etc/fail2ban/jail.conf` et `fail2ban.conf` sont **écrasés lors des mises à jour du paquet**. Toute personnalisation doit aller dans un fichier `.local`, qui surcharge le `.conf` sans être remplacé :

```bash
sudo nano /etc/fail2ban/jail.local
```

---

## 4. Configuration recommandée

### 4.1 Paramètres généraux (`[DEFAULT]`)

```ini
[DEFAULT]
# Durée du bannissement (secondes) ; -1 = permanent
bantime  = 3600

# Fenêtre de temps sur laquelle les échecs sont comptés
findtime = 600

# Nombre d'échecs avant bannissement
maxretry = 3

# Adresses jamais bannies (votre IP fixe, réseau interne...)
ignoreip = 127.0.0.1/8 ::1

# Backend de lecture des logs (auto = détection automatique, recommandé)
backend = auto

# Action par défaut : ban via le firewall + notification (voir 4.3)
banaction = ufw
```

| Paramètre | Explication |
|---|---|
| `bantime` | Durée du bannissement. `3600` (1h) est un bon point de départ ; augmenter progressivement (`bantime.increment` en 4.2) pour les récidivistes. |
| `findtime` | Période sur laquelle `maxretry` est évalué. Ici, 3 échecs en 10 minutes déclenchent un ban. |
| `maxretry` | Nombre d'échecs tolérés avant bannissement. Cohérent avec `MaxAuthTries 3` déjà en place côté SSH. |
| `ignoreip` | **Important** : y ajouter votre IP fixe si vous en avez une, pour ne jamais vous auto-bannir en cas de faute de frappe répétée. |
| `banaction` | `ufw` si vous utilisez UFW (voir section précédente) ; laisser la valeur par défaut (`iptables-multiport`) si vous utilisez `iptables`/`nftables` directement. |

### 4.2 Bannissement progressif (récidive)

Pour augmenter la durée de ban à chaque récidive plutôt qu'un ban fixe :

```ini
[DEFAULT]
bantime.increment = true
bantime.factor    = 2
bantime.maxtime   = 604800
```

- Un premier ban dure `bantime` (1h), le second `2h`, le troisième `4h`, etc., jusqu'à `bantime.maxtime` (ici 7 jours).

### 4.3 Jail SSH

```ini
[sshd]
enabled  = true
port     = ssh
logpath  = %(sshd_log)s
maxretry = 3
```

> Si le port SSH a été changé lors du durcissement (`Port XXXX` dans `sshd_config`), remplacer `port = ssh` par `port = XXXX`, sinon fail2ban peut mal appliquer la règle firewall correspondante.

---

## 5. Vérification

État général :

```bash
sudo fail2ban-client status
```

État détaillé d'une jail précise :

```bash
sudo fail2ban-client status sshd
```

Cette commande liste notamment les IP actuellement bannies.

Tester la syntaxe après modification :

```bash
sudo fail2ban-client -d
```

Redémarrer pour appliquer :

```bash
sudo systemctl restart fail2ban
```

---

## 6. Gérer les bannissements manuellement

```bash
# Bannir une IP manuellement
sudo fail2ban-client set sshd banip 203.0.113.10

# Débannir une IP (si vous vous êtes bloqué vous-même, ou faux positif)
sudo fail2ban-client set sshd unbanip 203.0.113.10

# Voir les logs d'activité de fail2ban
sudo tail -f /var/log/fail2ban.log
```

---

## 7. Ordre d'application recommandé

1. Installer `fail2ban` et vérifier qu'il démarre correctement.
2. Créer `/etc/fail2ban/jail.local` avec les paramètres de la section 4.1.
3. Ajouter votre IP actuelle dans `ignoreip` **avant** d'activer quoi que ce soit d'autre.
4. Activer la jail `sshd` (section 4.3), en adaptant le port si nécessaire.
5. Redémarrer et vérifier avec `fail2ban-client status sshd`.
6. Tester volontairement quelques échecs de connexion SSH depuis une IP que vous contrôlez (pas votre IP habituelle si elle n'est pas dans `ignoreip`) pour confirmer le déclenchement du ban.
7. Débannir cette IP de test une fois la vérification faite.

---

## 8. Points de vigilance

- **Toujours ajouter sa propre IP dans `ignoreip`** avant d'activer la jail SSH, sous peine de se bannir soi-même après quelques essais de mot de passe ratés.
- **Cohérence avec `banaction`** : si vous utilisez UFW, `banaction = ufw` ; sinon fail2ban peut essayer d'utiliser `iptables` directement, ce qui peut créer des règles en double ou en conflit avec UFW.
- **IP dynamique** : si votre IP change régulièrement (connexion résidentielle sans IP fixe), `ignoreip` par IP fixe n'est pas fiable — prévoir un accès de secours (console web du VPS) en cas d'auto-bannissement.
- Un ban `fail2ban` est **temporaire par nature** (sauf `bantime = -1`) : ce n'est pas un remplacement d'une vraie liste blanche d'IP si vous avez un accès très restreint à gérer.