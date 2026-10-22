# Guide de durcissement SSH (OpenSSH)

## 1. Principe : ne jamais éditer `sshd_config` directement

Pour éviter de modifier `/etc/ssh/sshd_config`, on dépose un fichier de configuration dédié dans le répertoire d'inclusion, qui sera lu automatiquement par OpenSSH et surchargera les valeurs par défaut.

```bash
sudo nano /etc/ssh/sshd_config.d/hardening.conf
```

Une fois le fichier rempli (sections 2 et 3 ci-dessous), on **valide toujours la syntaxe avant de redémarrer** :

```bash
# Vérification de syntaxe
sudo sshd -t

# Redémarrage du service
sudo systemctl restart ssh
```

>  Ne jamais fermer la session SSH en cours avant d'avoir validé qu'une nouvelle connexion fonctionne. Gardez toujours une session active en parallèle lors des tests.

---

## 2. Paramètres de durcissement général

| Paramètre | Valeur recommandée | Explication |
|---|---|---|
| `PermitRootLogin` | `no` | Interdit la connexion directe en root (défaut OpenSSH = `prohibit-password`, déjà mieux que `yes`). |
| `PubkeyAuthentication` | `yes` | Active l'authentification par clé, le socle du durcissement. |
| `PasswordAuthentication` | `no` | Interdit le mot de passe au profit des clés (à n'activer qu'après avoir déployé et testé une clé). |
| `KbdInteractiveAuthentication` | `no` | Ferme l'auth interactive (clavier/PAM). Nom moderne de l'ancien `ChallengeResponseAuthentication` (alias déprécié). |
| `PermitEmptyPasswords` | `no` | Refuse tout compte sans mot de passe (défaut déjà `no`, on le fige explicitement). |
| `AllowUsers` / `AllowGroups` | `alice bob` | Liste blanche des comptes autorisés à se connecter : le réglage le plus efficace. Ces directives s'additionnent. |
| `MaxAuthTries` | `3` | Limite les tentatives par connexion (défaut 6). |
| `MaxSessions` | `2` | Limite les sessions multiplexées sur une même connexion. |
| `MaxStartups` | `10:30:60` | Limite les connexions non authentifiées simultanées (anti-flood, réduit la fenêtre d'exploitation type regreSSHion). |
| `LoginGraceTime` | `30` | Délai d'authentification court : moins de connexions pendantes. Le 300 (5 min) souvent recommandé va à contre-sens du durcissement. |
| `LogLevel` | `VERBOSE` | Logs détaillés (empreintes de clés incluses) dans `/var/log/auth.log` (Debian) ou `/var/log/secure` (RHEL). |
| `AllowTcpForwarding` | `no` | Bloque le rebond TCP via SSH si le serveur n'en a pas besoin (défaut `yes`). |
| `AllowAgentForwarding` | `no` | Bloque l'agent forwarding (vecteur de rebond). |
| `X11Forwarding` | `no` | Désactive le transfert graphique X11, inutile sur un serveur. |
| `PermitTunnel` | `no` | Pas de tunnel réseau L2/L3 via SSH. |
| `ClientAliveInterval` / `ClientAliveCountMax` | `300` / `2` | Ferme les sessions inactives (ici après ~10 min sans réponse). |
| `UsePAM` | `yes` | Active PAM (politiques de mot de passe, verrouillage de comptes). |

### Exemple de bloc à insérer dans `hardening.conf`

```
PermitRootLogin no
PubkeyAuthentication yes
PasswordAuthentication no
KbdInteractiveAuthentication no
PermitEmptyPasswords no
AllowUsers alice bob
MaxAuthTries 3
MaxSessions 2
MaxStartups 10:30:60
LoginGraceTime 30
LogLevel VERBOSE
AllowTcpForwarding no
AllowAgentForwarding no
X11Forwarding no
PermitTunnel no
ClientAliveInterval 300
ClientAliveCountMax 2
UsePAM yes
```

---

## 3. Algorithmes de sécurité (crypto)

```
KexAlgorithms sntrup761x25519-sha512@openssh.com,curve25519-sha256,curve25519-sha256@libssh.org,diffie-hellman-group16-sha512,diffie-hellman-group18-sha512
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com,aes128-gcm@openssh.com
MACs hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com,umac-128-etm@openssh.com
HostKeyAlgorithms ssh-ed25519,ssh-ed25519-cert-v01@openssh.com,rsa-sha2-512,rsa-sha2-256
RequiredRSASize 3072
```

- **KexAlgorithms** : échange de clés, avec priorité à l'algorithme post-quantique hybride `sntrup761x25519-sha512`.
- **Ciphers** : uniquement des modes authentifiés (AEAD), pas de CBC.
- **MACs** : uniquement les variantes `-etm` (Encrypt-then-MAC), plus robustes.
- **HostKeyAlgorithms** : priorité à Ed25519, RSA en repli avec SHA-2.
- **RequiredRSASize** : rejette les clés RSA trop courtes.

---

## 4. Ordre d'application recommandé

1. Déployer et tester une **clé publique** pour chaque compte concerné.
2. Vérifier qu'une connexion par clé fonctionne **avant** de désactiver `PasswordAuthentication`.
3. Appliquer les paramètres de la section 2.
4. Appliquer les algorithmes de la section 3.
5. À chaque modification : `sudo sshd -t` puis `sudo systemctl restart ssh`.
6. Garder une session SSH active en parallèle pendant les tests.

---

## 5. Audit avec `ssh-audit`

```bash
sudo apt install pipx
pipx install ssh-audit
```

Pour analyser sa propre machine :

```bash
ssh-audit localhost
```

Nettoyage une fois l'audit terminé :

```bash
pipx uninstall ssh-audit
```

---

## 6. Points de vigilance

- **`AllowUsers`/`AllowGroups`** : à adapter impérativement aux comptes réels du serveur, sinon risque de verrouillage complet.
- **`PasswordAuthentication no`** : ne jamais l'activer avant d'avoir confirmé l'accès par clé, sous peine de perdre l'accès distant.
- **Fichiers `.conf` dans `sshd_config.d/`** : l'ordre de lecture dépend de l'ordre alphabétique et de la directive `Include` dans `sshd_config` principal — vérifier qu'aucune autre directive contradictoire n'existe déjà.
- **Compatibilité clients** : les algorithmes très récents (ex. `sntrup761x25519-sha512`) nécessitent des clients OpenSSH récents ; prévoir un test depuis les postes clients réels avant généralisation.