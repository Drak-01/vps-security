# Guide de durcissement PAM (Pluggable Authentication Modules)

## 1. Les composants de base de PAM

PAM (*Pluggable Authentication Modules*) est un système qui fournit des mécanismes pour intégrer de multiples technologies d'authentification dans un seul cadre cohérent. Les composants de base de PAM se répartissent en quatre catégories principales, chacune ayant un rôle spécifique dans le processus d'authentification et de gestion des sessions utilisateur.

### 1.1 Modules d'authentification (`auth`)

Les modules d'authentification sont le cœur de PAM. Ils sont responsables de la vérification de l'identité de l'utilisateur. Lorsqu'un utilisateur tente de se connecter à un système, ces modules vérifient les informations d'identification, comme les noms d'utilisateur et les mots de passe. Ils peuvent également intégrer des méthodes d'authentification plus avancées, telles que les empreintes digitales, la reconnaissance faciale, ou un second facteur (TOTP).

### 1.2 Modules de gestion de compte (`account`)

Les modules de gestion de compte interviennent après l'authentification. Leur rôle est de vérifier que l'accès de l'utilisateur est autorisé : compte expiré, droit d'accès à un service spécifique, conditions d'accès particulières (horaires, verrouillage suite à des échecs répétés, etc.).

### 1.3 Modules de gestion de mot de passe (`password`)

Ces modules interviennent lors du changement de mot de passe : application des règles de complexité, historique des mots de passe, hachage.

### 1.4 Modules de gestion de session (`session`)

Après une authentification réussie, les modules de gestion de session entrent en jeu. Ils s'occupent de la préparation et de la fin de la session utilisateur : montage de répertoires personnels, journalisation des activités, application de limites de ressources (`ulimit`), variables d'environnement.

---

## 2. Structure des fichiers de configuration

Chaque ligne d'un fichier `/etc/pam.d/<service>` suit la syntaxe :

```
type    contrôle    module_path    arguments
```

| Champ | Description |
|---|---|
| **type** | Catégorie du module : `auth`, `account`, `password`, `session`. |
| **contrôle** | Réaction de PAM à l'issue du module : `required`, `requisite`, `sufficient`, `optional` (ou syntaxe avancée `[value=action ...]`). |
| **module_path** | Chemin vers le fichier du module (ex. `pam_unix.so`, souvent dans `/lib/x86_64-linux-gnu/security/` ou `/usr/lib64/security/`). |
| **arguments** | Options spécifiques au module. |

### Rappel des valeurs de contrôle

| Contrôle | Comportement |
|---|---|
| `required` | Doit réussir pour que l'authentification globale réussisse, mais l'évaluation continue même en cas d'échec (l'échec n'est renvoyé qu'à la fin de la pile). |
| `requisite` | Comme `required`, mais stoppe immédiatement la pile en cas d'échec. |
| `sufficient` | Si le module réussit et qu'aucun `required` précédent n'a échoué, l'authentification est acceptée immédiatement. |
| `optional` | N'influence le résultat global que s'il est le seul module de la pile. |

---

## 3. Politique de mots de passe (`pam_pwquality`)

Fichier : `/etc/security/pwquality.conf`

```ini
# Longueur minimale
minlen = 12

# Nombre minimum de classes de caractères (maj, min, chiffres, spéciaux)
minclass = 3

# Nombre max de caractères identiques consécutifs
maxrepeat = 3

# Interdit la réutilisation de séquences (ex: "abcd", "1234")
maxsequence = 3

# Refuse si le mot de passe contient le login
usercheck = 1

# Nombre de caractères modifiés minimum par rapport à l'ancien mot de passe
difok = 5

# Root doit aussi respecter la politique
enforce_for_root

# Nombre de tentatives avant échec définitif du changement
retry = 3
```

Dans `/etc/pam.d/common-password` (Debian/Ubuntu) ou `/etc/pam.d/system-auth` (RHEL) :

```
password    requisite     pam_pwquality.so retry=3
password    [success=1 default=ignore]    pam_unix.so obscure use_authtok try_first_pass sha512 rounds=65536
```

### Historique des mots de passe (anti-réutilisation)

```
password    required    pam_pwhistory.so remember=5 use_authtok
```

---

## 4. Verrouillage de compte après échecs (`pam_faillock`)

### 4.1 Fichier `/etc/security/faillock.conf`

```ini
# Nombre d'échecs avant verrouillage
deny = 5

# Durée du verrouillage (secondes) ; 0 = verrouillage permanent (déblocage manuel requis)
unlock_time = 900

# Fenêtre de temps prise en compte pour compter les échecs
fail_interval = 900

# Journalisation détaillée
audit
silent
```

### 4.2 Intégration dans `/etc/pam.d/common-auth` (Debian/Ubuntu) ou `/etc/pam.d/system-auth` (RHEL/Fedora)

```
auth    required                      pam_faillock.so preauth
auth    [success=1 default=ignore]    pam_unix.so try_first_pass
auth    [default=die]                 pam_faillock.so authfail
auth    required                      pam_faillock.so authsucc
```

> Sur RHEL/CentOS, `authselect` peut régénérer ces fichiers : passer par `authselect enable-feature with-faillock` plutôt que d'éditer directement si `authselect` est utilisé.

### 4.3 Consulter et débloquer un compte

```bash
# État des échecs
sudo faillock --user alice

# Débloquer manuellement
sudo faillock --user alice --reset
```

---

## 5. Limitation des ressources par session (`pam_limits`)

Fichier : `/etc/security/limits.conf` (ou fragment dans `/etc/security/limits.d/hardening.conf`)

```
# Limite le nombre de processus par utilisateur (protection fork-bomb)
*        hard    nproc     100

# Limite le nombre de fichiers ouverts
*        hard    nofile    4096

# Désactive les core dumps (évite la fuite de données sensibles en mémoire)
*        hard    core      0
```

Activation dans `/etc/pam.d/common-session` (et `common-session-noninteractive`) :

```
session    required    pam_limits.so
```

---

## 6. Restriction d'accès par utilisateur/groupe/horaire (`pam_access`)

Fichier : `/etc/security/access.conf`

```
# Autorise uniquement root et le groupe "admins" depuis la console locale
+ : root admins : LOCAL

# Autorise alice et bob uniquement depuis un sous-réseau donné
+ : alice bob : 10.0.0.0/24

# Refuse tout le reste
- : ALL : ALL
```

Activation, en général en tout début de pile `auth` dans `/etc/pam.d/login` ou `/etc/pam.d/sshd` :

```
account    required    pam_access.so
```

---

## 7. Verrouillage automatique après inactivité (session)

Dans `/etc/pam.d/common-session`, combiné à `TMOUT` shell :

```
session    optional    pam_exec.so /usr/local/bin/log-session-start.sh
```

Et dans `/etc/profile.d/timeout.sh` (complémentaire, pas PAM à proprement parler) :

```bash
TMOUT=600
readonly TMOUT
export TMOUT
```

---

## 8. Ordre d'application recommandé sur un serveur

1. Sauvegarder les fichiers `/etc/pam.d/*` avant toute modification.
2. Mettre en place `pam_pwquality` + `pam_pwhistory` (section 3).
3. Mettre en place `pam_faillock` (section 4) — **tester dans une session parallèle** avant de fermer la session en cours, sous peine de verrouillage.
4. Ajouter `pam_limits` (section 5) si le serveur est multi-utilisateur.
5. Ajouter `pam_access` (section 6) si un contrôle fin par IP/groupe/horaire est nécessaire.
6. Vérifier chaque changement avec une connexion de test **avant** de fermer la session administrative en cours.
7. Consulter les logs (`/var/log/auth.log` ou `/var/log/secure`) après chaque modification pour confirmer le comportement attendu.

---

## 9. Points de vigilance

- **Ne jamais éditer `pam.d` sans session de secours ouverte** : une erreur de syntaxe ou une pile mal ordonnée peut bloquer tous les logins, y compris root.
- **`authselect` (RHEL/Fedora)** : sur ces distributions, les fichiers `pam.d` sont souvent gérés par `authselect` ; une édition manuelle peut être écrasée à la prochaine régénération de profil.
- **`enforce_for_root`** : à activer avec prudence, mais recommandé en environnement durci.
- **Verrouillage permanent (`unlock_time = 0`)** : nécessite une procédure de déblocage manuel documentée (`faillock --reset`), sinon risque de blocage total d'un compte critique.
- **Cohérence avec SSH** : `UsePAM yes` dans `sshd_config` est indispensable pour que ces politiques PAM (verrouillage, complexité) s'appliquent réellement aux connexions SSH.