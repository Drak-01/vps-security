# Guide de durcissement du noyau (sysctl)

## 1. Objectif

Le noyau Linux expose une grande partie de sa configuration (réseau, mémoire, sécurité, limites système...) sous forme de fichiers virtuels dans `/proc/sys/`. `sysctl` est l'interface pratique pour lire ou écrire ces valeurs, sans avoir à faire `cat`/`echo` directement dans ces fichiers.

L'objectif du durcissement `sysctl` est de réduire la surface d'attaque au niveau du noyau, en particulier :

- **Réseau** : empêcher le serveur de router du trafic qu'il ne devrait pas router, de faire confiance à des paquets falsifiés (spoofing) ou manipulés (redirections ICMP), et limiter l'exposition à des attaques réseau classiques (SYN flood, source routing).
- **Noyau** : limiter les informations exposées aux processus non privilégiés (adresses mémoire, messages `dmesg`), réduire les vecteurs d'escalade de privilèges (`ptrace`, eBPF non privilégié, rechargement de noyau via `kexec`).
- **Système de fichiers** : empêcher des attaques classiques par liens symboliques/durs ou par manipulation de fichiers spéciaux (FIFO), et éviter les fuites de mémoire via les core dumps.

Ce guide ne bloque **pas de service applicatif** : il agit uniquement sur le comportement du noyau, ce qui le rend généralement sans impact sur les applications, à l'exception des machines faisant explicitement office de routeur/VPN (voir section 6).

---

## 2. Configuration persistante

Les valeurs définies via `sysctl -w` ne survivent pas au redémarrage. Pour les rendre permanentes, on les écrit dans :

- `/etc/sysctl.conf` (fichier historique) ;
- ou, comme pour SSH/PAM, un fichier dédié dans `/etc/sysctl.d/` (ex. `/etc/sysctl.d/99-hardening.conf`) — c'est la méthode recommandée pour ne pas éditer le fichier principal.

```bash
sudo nano /etc/sysctl.d/99-hardening.conf
```

Application sans redémarrage :

```bash
sudo sysctl -p /etc/sysctl.d/99-hardening.conf
# ou, pour recharger tous les fichiers de /etc/sysctl.d/ :
sudo sysctl --system
```

---

## 3. Avant le durcissement : sauvegarder l'existant et diagnostiquer

Certaines valeurs varient selon le système, la distribution et la version du noyau (rôle de routeur, présence d'IPv6, etc.). **Il faut sauvegarder les valeurs actuelles avant toute modification**, pour pouvoir revenir en arrière si une valeur casse un service (ex. `ip_forward` sur un routeur ou un hôte de conteneurs).

```bash
# Sauvegarde de l'état actuel de tous les paramètres
sudo sysctl -a > ~/sysctl-backup-$(date +%F).txt

# Pour restaurer une valeur précise si besoin
sudo sysctl -w net.ipv4.ip_forward=1
```

### 3.1 Diagnostic rapide avant application

Pour comparer l'état actuel aux valeurs cibles sur les paramètres les plus sensibles :

```bash
for k in net.ipv4.conf.all.accept_redirects net.ipv4.conf.all.send_redirects \
         net.ipv4.conf.all.rp_filter net.ipv4.conf.all.log_martians \
         kernel.kptr_restrict kernel.yama.ptrace_scope; do
  printf "%-42s = %s\n" "$k" "$(sysctl -n $k)"
done
```

### 3.2 Déterminer si l'IPv6 est en SLAAC (dynamique) ou statique

Ce point conditionne uniquement `accept_ra` (voir section 4.2) — il n'a **aucun impact** sur `accept_redirects`, qui reste sans risque dans tous les cas (redirections ICMPv6 de routes, mécanisme indépendant de l'adressage).

```bash
ip -6 addr show
```

- Adresse avec le flag **`dynamic`** (ex. `scope global dynamic mngtmpaddr`) → IPv6 attribuée par **SLAAC/Router Advertisement**. **Ne pas désactiver `accept_ra`**, au risque de perdre l'IPv6 au renouvellement du bail (`valid_lft`).
- Adresse **sans** le flag `dynamic` → IPv6 configurée en statique. `accept_ra = 0` peut être appliqué sans risque.

> ⚠️ Vérifiez aussi si la machine fait du routage, du NAT, ou héberge des conteneurs/VMs (Docker, LXC, KVM) **avant** de désactiver `net.ipv4.ip_forward` ou `net.ipv6.conf.*.forwarding`.

---

## 4. Paramètres recommandés

### 4.1 Réseau : routage et redirections

| Paramètre | Valeur | Ce que ça corrige |
|---|---|---|
| `net.ipv4.ip_forward` | `0` | Empêche la machine de router du trafic entre interfaces si elle n'est pas censée être un routeur/NAT. |
| `net.ipv4.conf.all.accept_redirects` / `.default.` | `0` | Empêche le noyau de modifier sa table de routage sur simple réception d'un message ICMP redirect (souvent falsifiable). |
| `net.ipv6.conf.all.accept_redirects` / `.default.` | `0` | Équivalent IPv6 du point précédent. |
| `net.ipv4.conf.all.secure_redirects` / `.default.` | `0` | Ferme aussi les redirects dits "sécurisés" (venant d'une gateway connue), qui restent un vecteur de manipulation de route. |
| `net.ipv4.conf.all.send_redirects` / `.default.` | `0` | Empêche la machine d'émettre elle-même des ICMP redirects (pertinent si elle n'est pas un routeur). |
| `net.ipv4.conf.all.accept_source_route` / `.default.` | `0` | Bloque le *source routing* IPv4, qui permet à l'émetteur d'un paquet d'imposer sa propre route (contournement de filtrage). |
| `net.ipv6.conf.all.accept_source_route` | `0` | Équivalent IPv6. |

### 4.2 Réseau : anti-spoofing et journalisation

| Paramètre | Valeur | Ce que ça corrige |
|---|---|---|
| `net.ipv4.conf.all.rp_filter` / `.default.` | `1` | Active le *reverse path filtering* : rejette les paquets dont l'adresse source ne correspond pas à une route valide (protection anti-spoofing IP). |
| `net.ipv4.conf.all.log_martians` / `.default.` | `1` | Journalise les paquets aux adresses source/destination impossibles ("martiens"), utile pour détecter des tentatives de spoofing. |
| `net.ipv4.icmp_echo_ignore_broadcasts` | `1` | Empêche de répondre aux pings envoyés en broadcast (protection contre l'attaque Smurf). |
| `net.ipv4.icmp_ignore_bogus_error_responses` | `1` | Ignore les réponses ICMP d'erreur invalides, qui polluent les logs sans intérêt réel. |
| `net.ipv4.tcp_syncookies` | `1` | Active les SYN cookies : protège contre le SYN flood (épuisement de la table des connexions en attente) sans bloquer les connexions légitimes. |
| `net.ipv4.tcp_rfc1337` | `1` | Corrige le comportement des paquets en état `TIME-WAIT`, empêchant une classe d'attaques par réinitialisation de connexion. |
| `net.ipv6.conf.all.accept_ra` / `.default.` | `0` **uniquement si IPv6 statique** — sinon **ne pas définir cette clé** (voir section 3.2) | Empêche la machine d'accepter des annonces de routeur IPv6 non maîtrisées (utile sur un serveur avec adressage IPv6 statique). Sur un VPS en SLAAC, la couper coupe l'IPv6. |

### 4.3 Noyau : exposition d'informations et eBPF

| Paramètre | Valeur | Ce que ça corrige |
|---|---|---|
| `kernel.kptr_restrict` | `2` | Masque les adresses mémoire du noyau dans `/proc`, qui pourraient aider à contourner l'ASLR lors d'une exploitation de vulnérabilité noyau. |
| `kernel.dmesg_restrict` | `1` | Réserve la lecture des messages du noyau (`dmesg`) aux utilisateurs privilégiés, évitant une fuite d'informations sur le matériel/les modules chargés. |
| `kernel.randomize_va_space` | `2` | Active l'ASLR complet (données, pile, tas, bibliothèques), rendant l'exploitation de failles mémoire plus difficile. |
| `kernel.yama.ptrace_scope` | `1` | Limite `ptrace` aux processus parents/enfants directs, empêchant un processus utilisateur d'inspecter/manipuler la mémoire d'un autre processus arbitraire. |
| `kernel.unprivileged_bpf_disabled` | `2` | Interdit aux utilisateurs non privilégiés de charger des programmes eBPF (vecteur d'escalade de privilèges régulièrement exploité). |
| `net.core.bpf_jit_harden` | `2` | Durcit le JIT eBPF pour tous les utilisateurs, réduisant le risque d'exploitation de bugs du compilateur JIT. |
| `kernel.kexec_load_disabled` | `1` | Interdit le chargement d'un nouveau noyau à chaud via `kexec`, ce qui empêche un attaquant root de remplacer le noyau en mémoire sans redémarrage physique/contrôlé. |

### 4.4 Système de fichiers

| Paramètre | Valeur | Ce que ça corrige |
|---|---|---|
| `fs.protected_hardlinks` | `1` | Empêche de créer un lien dur vers un fichier qu'on ne possède pas (protection contre des attaques de type TOCTOU/privilege escalation via `/tmp`). |
| `fs.protected_symlinks` | `1` | Empêche de suivre un lien symbolique dans un répertoire partagé en écriture (`/tmp`) si on n'en est pas propriétaire. |
| `fs.protected_fifos` | `2` | Empêche l'écriture dans une FIFO créée par un autre utilisateur dans un répertoire world-writable. |
| `fs.protected_regular` | `2` | Même protection que ci-dessus pour les fichiers réguliers ouverts dans un répertoire world-writable. |
| `fs.suid_dumpable` | `0` | Désactive les core dumps pour les binaires `setuid`, évitant qu'un crash n'expose en clair de la mémoire potentiellement sensible. |

### 4.5 Fichier complet — profil « serveur avec IPv6 statique »

À utiliser si `ip -6 addr show` ne montre **pas** le flag `dynamic` sur l'adresse globale.

```ini
# --- Réseau : routage et redirections (paires all + default) ---
net.ipv4.ip_forward = 0
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv6.conf.all.accept_redirects = 0
net.ipv6.conf.default.accept_redirects = 0
net.ipv4.conf.all.secure_redirects = 0
net.ipv4.conf.default.secure_redirects = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.send_redirects = 0
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.default.accept_source_route = 0
net.ipv6.conf.all.accept_source_route = 0

# --- Réseau : anti-spoofing et journalisation ---
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1
net.ipv4.conf.all.log_martians = 1
net.ipv4.conf.default.log_martians = 1
net.ipv4.icmp_echo_ignore_broadcasts = 1
net.ipv4.icmp_ignore_bogus_error_responses = 1
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_rfc1337 = 1
net.ipv6.conf.all.accept_ra = 0
net.ipv6.conf.default.accept_ra = 0

# --- Noyau : exposition d'informations et eBPF ---
kernel.kptr_restrict = 2
kernel.dmesg_restrict = 1
kernel.randomize_va_space = 2
kernel.yama.ptrace_scope = 1
kernel.unprivileged_bpf_disabled = 2
net.core.bpf_jit_harden = 2
kernel.kexec_load_disabled = 1

# --- Système de fichiers ---
fs.protected_hardlinks = 1
fs.protected_symlinks = 1
fs.protected_fifos = 2
fs.protected_regular = 2
fs.suid_dumpable = 0
```

### 4.6 Fichier complet — profil « VPS avec IPv6 en SLAAC » (dynamique)

À utiliser si `ip -6 addr show` montre le flag `dynamic` (cas fréquent chez OVH, Hetzner, Scaleway, DigitalOcean...). Identique au profil précédent, **sans les deux lignes `accept_ra`** : les couper couperait l'IPv6 au renouvellement du bail RA.

```ini
# --- Réseau : routage et redirections (paires all + default) ---
net.ipv4.ip_forward = 0
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv6.conf.all.accept_redirects = 0
net.ipv6.conf.default.accept_redirects = 0
net.ipv4.conf.all.secure_redirects = 0
net.ipv4.conf.default.secure_redirects = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.send_redirects = 0
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.default.accept_source_route = 0
net.ipv6.conf.all.accept_source_route = 0

# --- Réseau : anti-spoofing et journalisation ---
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1
net.ipv4.conf.all.log_martians = 1
net.ipv4.conf.default.log_martians = 1
net.ipv4.icmp_echo_ignore_broadcasts = 1
net.ipv4.icmp_ignore_bogus_error_responses = 1
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_rfc1337 = 1
# accept_ra volontairement absent : IPv6 obtenue en SLAAC sur ce type d'hôte

# --- Noyau : exposition d'informations et eBPF ---
kernel.kptr_restrict = 2
kernel.dmesg_restrict = 1
kernel.randomize_va_space = 2
kernel.yama.ptrace_scope = 1
kernel.unprivileged_bpf_disabled = 2
net.core.bpf_jit_harden = 2
kernel.kexec_load_disabled = 1

# --- Système de fichiers ---
fs.protected_hardlinks = 1
fs.protected_symlinks = 1
fs.protected_fifos = 2
fs.protected_regular = 2
fs.suid_dumpable = 0
```

> `accept_redirects = 0` reste valable et sans risque dans les deux profils, y compris en SLAAC : il bloque uniquement les redirections ICMPv6 de type *Redirect* (changement de route), un mécanisme indépendant des Router Advertisements qui gèrent l'adressage.

---

## 5. Application

```bash
sudo sysctl --system
```

Vérifier qu'une valeur a bien été prise en compte :

```bash
sysctl net.ipv4.tcp_syncookies
```

---

## 6. Ordre d'application recommandé

1. Sauvegarder l'état actuel (`sysctl -a > backup.txt`).
2. Lancer le diagnostic rapide (section 3.1) pour voir l'écart avec les valeurs cibles.
3. Identifier le rôle de la machine (routeur, NAT, hôte de conteneurs/VMs) et le mode IPv6 (SLAAC ou statique, section 3.2) pour choisir le bon profil (4.5 ou 4.6).
4. Déposer le fichier correspondant dans `/etc/sysctl.d/99-hardening.conf`.
5. Appliquer avec `sudo sysctl --system`.
6. Vérifier le fonctionnement réseau normal (connectivité IPv4 **et** IPv6, DNS, services dépendants) juste après application.
7. Consulter les logs (`dmesg`, `journalctl -k`) pour repérer d'éventuels rejets inattendus (`log_martians`, `rp_filter`).

---

## 7. Points de vigilance

- **`net.ipv4.ip_forward` / `net.ipv6.conf.*.forwarding`** : à laisser à `1` si la machine fait du routage, du NAT, héberge du Docker/LXC/KVM, ou est un serveur VPN.
- **`rp_filter`** : `1` (strict) convient à un VPS standard (une seule interface/passerelle) ; réserver le mode `2` (loose) aux topologies à routage asymétrique (plusieurs uplinks).
- **`accept_ra`** : à ne définir que si l'IPv6 est en statique (section 3.2). Sur un VPS en SLAAC (flag `dynamic` sur l'adresse), omettre cette clé plutôt que de la mettre à `0`, sous peine de perdre l'IPv6 au renouvellement du bail RA.
- **`accept_redirects` en IPv6** : reste sans risque même en SLAAC — ce n'est pas le même mécanisme que les Router Advertisements, donc pas de conflit avec `accept_ra`.
- **`unprivileged_bpf_disabled` / `bpf_jit_harden`** : peuvent impacter certains outils de supervision réseau ou de sécurité (ex. certains agents eBPF) — à tester avant généralisation.
- Toujours tester sur un environnement non critique avant application en production, notamment pour les paramètres réseau qui peuvent couper l'accès distant.

---

## 8. Exemple validé — VPS avec IPv6 SLAAC

État constaté avant durcissement sur un VPS type (une interface `ens16`, IPv6 avec flag `dynamic mngtmpaddr`) :

```
net.ipv4.conf.all.accept_redirects         = 1
net.ipv4.conf.all.send_redirects           = 1
net.ipv4.conf.all.rp_filter                = 2
net.ipv4.conf.all.log_martians             = 0
kernel.kptr_restrict                       = 1
kernel.yama.ptrace_scope                   = 1
```

Sur ce type d'hôte : `accept_redirects`, `send_redirects`, `log_martians` et `kptr_restrict` sont à durcir ; `rp_filter` peut passer de `2` à `1` (une seule passerelle) ; `ptrace_scope` était déjà conforme. Le profil à appliquer est celui de la section 4.6 (sans `accept_ra`).