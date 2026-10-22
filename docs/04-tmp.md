# Guide de durcissement de `/tmp`

## 1. Objectif

`/tmp` est un répertoire **world-writable** (tout utilisateur peut y écrire), ce qui en fait une cible classique : un attaquant peut y déposer un fichier, un lien, ou un binaire pour ensuite l'exploiter. Le durcissement se fait à deux niveaux complémentaires :

- les **paramètres sysctl** (`fs.protected_*`, voir le guide [03-sysctl.md](03-sysctl.md)), qui protègent contre la manipulation de liens et de fichiers déjà présents ;
- les **options de montage**, qui protègent contre l'exécution et l'élévation de privilèges via des fichiers déposés dans `/tmp`.

Ce guide couvre le second niveau.

---

## 2. Options de montage recommandées

| Option | Effet | Attaque bloquée |
|---|---|---|
| `noexec` | Interdit d'exécuter un binaire ou un script depuis ce montage | Charge utile déposée dans `/tmp` puis exécutée directement. |
| `nosuid` | Neutralise les bits `setuid`/`setgid` sur les fichiers du montage | Élévation de privilèges via un binaire `setuid` déposé sur cette partition. |
| `nodev` | Ignore les fichiers de périphériques (`/dev/*`) présents sur le montage | Fichier de périphérique piégé créé sur une partition de données pour accéder au matériel/mémoire. |

Ces trois options s'appliquent au niveau du **montage**, indépendamment des permissions Unix classiques (propriétaire, `chmod`) : même root ne peut pas contourner `noexec` en changeant les droits d'un fichier, il faut remonter la partition autrement.

---

## 3. Vérifier l'état actuel

```bash
mount | grep /tmp
```

- Si aucune ligne ne s'affiche → `/tmp` est un simple sous-répertoire de `/`, sans aucune de ces protections. C'est le cas par défaut sur la plupart des VPS.
- Si une ligne s'affiche avec `tmpfs` ou une partition dédiée → vérifier si `noexec,nosuid,nodev` sont déjà présents dans les options listées.

---

## 4. Appliquer la protection

### 4.1 Méthode `tmpfs` (recommandée sur un VPS)

`tmpfs` monte `/tmp` en RAM, ce qui est à la fois plus rapide et plus simple à sécuriser qu'une partition disque dédiée.

Ajouter dans `/etc/fstab` :

```
tmpfs   /tmp   tmpfs   rw,nosuid,nodev,noexec,size=512M   0 0
```

> Équivalent à `defaults,noexec,nosuid,nodev,size=512M` vu en section 4.1 : `rw` remplace ici `defaults` en listant explicitement le droit d'écriture, le reste est identique. `size=512M` est un exemple de dimensionnement plus conservateur pour un VPS à mémoire limitée — à ajuster selon la RAM disponible.

Puis appliquer :

- si `/tmp` n'est pas encore monté séparément (première mise en place), utiliser directement l'entrée `fstab` :
  ```bash
  sudo mount /tmp
  ```
- si `/tmp` est déjà monté avec d'anciennes options et qu'on veut juste changer les options sans démonter (ex. session distante active) :
  ```bash
  sudo mount -o remount /tmp
  ```

Vérifier que les options sont bien actives :

```bash
findmnt -no OPTIONS /tmp
```

- `findmnt` lit l'état réel du montage (contrairement à `cat /etc/fstab`, qui ne montre que la configuration voulue, pas ce qui est effectivement monté).
- `-n` : supprime l'en-tête de colonne, pour un résultat directement exploitable en script.
- `-o OPTIONS` : n'affiche que la colonne des options de montage, plutôt que le tableau complet (source, cible, type, options).

Sortie attendue :

```
rw,nosuid,nodev,noexec,relatime,size=524288k,inode65536
```

Si `noexec`, `nosuid` ou `nodev` sont absents de cette sortie, le montage n'est pas durci malgré la ligne dans `/etc/fstab` — il faut alors vérifier qu'aucun montage bind ou service n'écrase les options (`systemctl status tmp.mount` sur les systèmes utilisant l'unit `tmp.mount` de systemd plutôt que `fstab`).

> `size=1G` : à ajuster selon la RAM disponible du VPS, puisque `tmpfs` consomme de la mémoire vive et non du disque.

### 4.2 Méthode partition disque existante

Si `/tmp` est déjà sur sa propre partition (moins courant sur un VPS), il suffit d'ajouter les options dans la ligne existante de `/etc/fstab` :

```
/dev/sdXn   /tmp   ext4   defaults,noexec,nosuid,nodev   0   2
```

---

## 5. Protection complémentaire par service (`systemd`)

Indépendamment du montage global, chaque service systemd peut recevoir son propre `/tmp` isolé, invisible des autres processus :

```ini
[Service]
PrivateTmp=true
```

À ajouter dans le fichier unit du service (`/etc/systemd/system/<service>.d/override.conf` via `systemctl edit <service>`), puis :

```bash
sudo systemctl daemon-reload
sudo systemctl restart <service>
```

C'est un complément, pas un remplacement des options de montage : `PrivateTmp` isole un service des autres, mais ne bloque pas l'exécution de binaires dans son propre `/tmp` privé.

---

## 6. Points de vigilance

- **`noexec` peut casser des installeurs ou outils** qui compilent/exécutent temporairement depuis `/tmp` (certains scripts d'installation, `pip` avec compilation native, certains agents CI/CD). À tester avant généralisation en production.
- **`/var/tmp`** suit les mêmes règles et est souvent oublié — pensez à l'inclure si des applications y écrivent des fichiers temporaires persistants.
- Après passage en `tmpfs`, tout contenu de `/tmp` est perdu au redémarrage (comportement normalement attendu, mais à vérifier si un service s'appuie dessus pour autre chose que du temporaire).
- Vérifier après application qu'aucun service critique ne dépend d'une exécution depuis `/tmp` (logs d'erreurs `Permission denied` juste après le remount).