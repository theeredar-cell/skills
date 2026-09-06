# Les serveurs Linux : architecture, fonctionnement et composants

Une leçon complète pour comprendre ce qui tourne réellement sur une machine qui sert des sites,
des API et des bases de données. On part du matériel, on monte jusqu'à la requête HTTP,
et on démonte au passage les briques les plus courantes : Apache, Nginx, MariaDB, PostgreSQL,
Redis, Docker.

**Public visé** : développeur, étudiant ou administrateur débutant qui sait ouvrir un terminal.
**Durée** : environ 3 heures de lecture, plus les travaux pratiques.

---

## Sommaire

1. [Qu'est-ce qu'un serveur Linux ?](#1-quest-ce-quun-serveur-linux-)
2. [L'architecture en couches](#2-larchitecture-en-couches)
3. [Le noyau Linux](#3-le-noyau-linux)
4. [Le démarrage, du bouton d'alimentation au service prêt](#4-le-démarrage-du-bouton-dalimentation-au-service-prêt)
5. [systemd et la gestion des services](#5-systemd-et-la-gestion-des-services)
6. [Le système de fichiers](#6-le-système-de-fichiers)
7. [Utilisateurs, groupes et permissions](#7-utilisateurs-groupes-et-permissions)
8. [Processus, mémoire et ordonnancement](#8-processus-mémoire-et-ordonnancement)
9. [Le réseau](#9-le-réseau)
10. [La pile logicielle d'un serveur : la carte des composants](#10-la-pile-logicielle-dun-serveur--la-carte-des-composants)
11. [Les serveurs web : Apache et Nginx](#11-les-serveurs-web--apache-et-nginx)
12. [Les bases de données relationnelles : MariaDB/MySQL et PostgreSQL](#12-les-bases-de-données-relationnelles--mariadbmysql-et-postgresql)
13. [Caches, NoSQL, files de messages et moteurs de recherche](#13-caches-nosql-files-de-messages-et-moteurs-de-recherche)
14. [Conteneurs et orchestration](#14-conteneurs-et-orchestration)
15. [Le trajet complet d'une requête HTTP](#15-le-trajet-complet-dune-requête-http)
16. [Sécurité](#16-sécurité)
17. [Observabilité : logs, métriques, sauvegardes](#17-observabilité--logs-métriques-sauvegardes)
18. [Travaux pratiques](#18-travaux-pratiques)
19. [Quiz de révision](#19-quiz-de-révision)
20. [Glossaire](#20-glossaire)

---

## 1. Qu'est-ce qu'un serveur Linux ?

Un serveur n'est pas une catégorie de matériel, c'est un **rôle**. La même machine devient un
serveur dès qu'elle exécute en permanence des programmes qui attendent des requêtes venant du
réseau et y répondent. Un vieux portable, un Raspberry Pi, une instance cloud et une lame de
centre de données peuvent tous jouer ce rôle.

Ce qui distingue une installation serveur d'un poste de travail :

| Aspect | Poste de travail | Serveur |
|---|---|---|
| Interface | Environnement graphique (GNOME, KDE) | Ligne de commande, accès distant SSH |
| Durée de vie | Éteint chaque soir | Fonctionne en continu, souvent des années |
| Charge | Un utilisateur interactif | Des milliers de connexions simultanées |
| Priorité | Latence perçue par l'humain | Débit, disponibilité, prévisibilité |
| Mises à jour | Quand ça arrange | Fenêtres planifiées, souvent sans redémarrage |

**Pourquoi Linux domine ce rôle ?** Trois raisons pratiques. Le système est modulaire, donc on
n'installe que ce dont on a besoin (une image serveur minimale tient en quelques centaines de
mégaoctets). Il est pilotable intégralement en ligne de commande, donc scriptable et
reproductible. Et il est libre, donc dupliquer un serveur en mille exemplaires ne coûte rien en
licences.

**Distributions serveur courantes** : Debian et Ubuntu Server (famille APT, paquets `.deb`),
Red Hat Enterprise Linux, Rocky Linux, AlmaLinux et Fedora Server (famille DNF/RPM), et
Alpine Linux (très légère, omniprésente dans les conteneurs). Les commandes de cette leçon
utilisent la syntaxe Debian/Ubuntu quand elle diffère, avec l'équivalent RHEL indiqué.

---

## 2. L'architecture en couches

Tout le système s'organise en couches empilées. Chaque couche ne parle qu'à ses voisines
immédiates, ce qui permet de remplacer une brique sans toucher aux autres.

```
┌─────────────────────────────────────────────────────────────┐
│  APPLICATIONS                                               │
│  Apache, Nginx, MariaDB, PostgreSQL, Redis, votre code      │
├─────────────────────────────────────────────────────────────┤
│  BIBLIOTHÈQUES SYSTÈME (glibc, OpenSSL, libpcre...)         │
│  Traduisent les appels du langage en appels système         │
├─────────────────────────────────────────────────────────────┤
│  APPELS SYSTÈME (open, read, write, socket, fork, mmap...)  │  <- frontière
├─────────────────────────────────────────────────────────────┤     de sécurité
│  NOYAU LINUX                                                │
│  Processus | Mémoire | VFS | Réseau | Pilotes               │
├─────────────────────────────────────────────────────────────┤
│  MATÉRIEL : CPU, RAM, disques, cartes réseau                │
└─────────────────────────────────────────────────────────────┘
```

La ligne des appels système est la frontière la plus importante du système. Au-dessus, on est en
**espace utilisateur** (*user space*) : un programme qui plante n'emporte que lui-même. En
dessous, on est en **espace noyau** (*kernel space*) : un bug y fait tomber la machine entière.
Le processeur applique physiquement cette séparation via ses niveaux de privilège (anneaux 0 et 3
sur x86).

Concrètement, quand votre code PHP écrit `file_get_contents('/etc/hosts')`, la chaîne est :
PHP appelle une fonction de la glibc, la glibc exécute l'instruction `syscall` avec le numéro de
`openat`, le processeur bascule en mode noyau, le noyau vérifie les permissions, lit les blocs via
le pilote de disque, recopie les données dans la mémoire du processus, puis rend la main.

**Observer cette frontière en direct** :

```bash
strace -c cat /etc/hostname      # compte les appels système d'une commande
strace -e trace=openat,read ls   # ne trace que certains appels
```

---

## 3. Le noyau Linux

Le noyau est le programme qui possède réellement la machine. Tout le reste lui demande la
permission. Il est **monolithique modulaire** : le code cœur est un seul gros binaire
(`/boot/vmlinuz-*`), mais les pilotes peuvent être chargés et déchargés à chaud sous forme de
modules (`.ko`).

Ses cinq grands chantiers :

### 3.1 La gestion des processus
Le noyau crée les processus (`fork`, `clone`, `execve`), leur attribue du temps de calcul via
l'**ordonnanceur** (CFS, remplacé par EEVDF depuis Linux 6.6), et les fait communiquer (signaux,
tubes, sockets, mémoire partagée). Il donne l'illusion que des centaines de programmes tournent
en même temps sur quelques cœurs, en les alternant toutes les quelques millisecondes.

### 3.2 La gestion de la mémoire
Chaque processus voit un espace d'adressage virtuel privé et continu. La **MMU** du processeur,
pilotée par les tables de pages du noyau, traduit ces adresses virtuelles en adresses physiques.
Cela permet trois choses essentielles : l'isolation (un processus ne peut pas lire la mémoire d'un
autre), le partage (une bibliothèque comme la glibc n'est chargée qu'une fois en RAM pour tous),
et le *swap* (déplacer des pages inactives sur disque).

Le noyau utilise aussi toute la RAM libre comme **cache disque** (*page cache*). C'est pourquoi
`free -h` montre souvent très peu de mémoire "libre" : ce n'est pas un problème, la colonne
`available` est la seule qui compte.

### 3.3 Le système de fichiers virtuel (VFS)
Le VFS est une couche d'abstraction qui présente ext4, XFS, Btrfs, NFS, tmpfs et une clé USB
derrière la même interface `open`/`read`/`write`/`close`. C'est ce qui permet le principe
"tout est fichier" : un disque, un terminal, une socket réseau et un capteur de température
s'utilisent tous avec les mêmes appels.

### 3.4 La pile réseau
Le noyau implémente TCP/IP intégralement : découpage en paquets, retransmission, contrôle de
congestion, routage, filtrage (Netfilter, base de `iptables` et `nftables`). Les applications ne
voient que des **sockets**.

### 3.5 Les pilotes de périphériques
Le code qui parle au matériel réel. Il représente la majorité des lignes du noyau.

**Commandes utiles** :

```bash
uname -a                 # version du noyau et architecture
lsmod                    # modules chargés
dmesg -T | tail -30      # messages du noyau, horodatés
sysctl -a | grep tcp     # paramètres réglables du noyau
cat /proc/cpuinfo        # /proc est une fenêtre sur les structures du noyau
```

`/proc` et `/sys` ne sont pas de vrais fichiers sur disque : ce sont des systèmes de fichiers
virtuels générés à la volée par le noyau. Lire `/proc/meminfo` exécute du code noyau.

---

## 4. Le démarrage, du bouton d'alimentation au service prêt

Comprendre cette séquence, c'est savoir où chercher quand un serveur ne remonte pas.

**Étape 1 : le firmware (UEFI, ou BIOS sur les machines anciennes).**
Il teste le matériel, puis cherche un chargeur d'amorçage. En UEFI, il lit la partition EFI
(`/boot/efi`, formatée en FAT32) et exécute un fichier `.efi`.

**Étape 2 : le chargeur d'amorçage (GRUB2 le plus souvent).**
Il affiche le menu des noyaux disponibles, charge en RAM le noyau choisi et l'**initramfs**, puis
passe la main au noyau avec une ligne de paramètres (`root=UUID=...`, `quiet`).

**Étape 3 : l'initramfs.**
C'est un mini système de fichiers en RAM contenant juste les pilotes nécessaires pour atteindre
le vrai disque racine : contrôleur RAID, LVM, déchiffrement LUKS, pilote réseau pour un démarrage
iSCSI. Sans lui, un noyau générique ne saurait pas monter une racine chiffrée sur RAID.

**Étape 4 : le montage de la racine et le PID 1.**
Le noyau monte `/` puis exécute `/sbin/init`, qui est aujourd'hui un lien vers `systemd`.
Ce processus porte le PID 1 et devient l'ancêtre de tous les autres. S'il meurt, le noyau panique.

**Étape 5 : systemd déroule les cibles.**
Il monte les systèmes de fichiers de `/etc/fstab`, configure le réseau, lance les services, et
atteint la cible par défaut (`multi-user.target` sur un serveur, `graphical.target` sur un poste).

**Diagnostiquer un démarrage lent** :

```bash
systemd-analyze                 # temps total, réparti firmware / noyau / userspace
systemd-analyze blame           # les services les plus lents, classés
systemd-analyze critical-chain  # le chemin critique des dépendances
journalctl -b -p err            # erreurs du démarrage courant
journalctl -b -1                # journal du démarrage précédent (utile après un crash)
```

---

## 5. systemd et la gestion des services

`systemd` est le gestionnaire de services de la quasi-totalité des distributions serveur
modernes. Il remplace l'ancien System V init et ses scripts shell séquentiels par un modèle
déclaratif de **unités** avec dépendances, ce qui autorise le démarrage en parallèle.

### 5.1 Les types d'unités

| Suffixe | Rôle |
|---|---|
| `.service` | Un démon (Apache, PostgreSQL, votre application) |
| `.socket` | Une socket en écoute qui démarre le service à la première connexion |
| `.timer` | Un déclencheur périodique, remplaçant moderne de cron |
| `.mount` | Un point de montage |
| `.target` | Un regroupement, équivalent des anciens niveaux d'exécution |

### 5.2 Les commandes du quotidien

```bash
systemctl status nginx           # état, PID, mémoire, dernières lignes de log
systemctl start|stop|restart nginx
systemctl reload nginx           # recharge la conf sans couper les connexions
systemctl enable --now nginx     # active au démarrage ET démarre maintenant
systemctl list-units --failed    # tout ce qui est en échec : le premier réflexe
journalctl -u nginx -f           # suivre les logs du service en direct
journalctl -u nginx --since "1 hour ago"
```

Retenez la différence entre `restart` (le processus meurt et renaît, les connexions en cours
tombent) et `reload` (le processus relit sa configuration à chaud). En production, on privilégie
`reload` chaque fois que le service le supporte.

### 5.3 Écrire une unité de service

Fichier `/etc/systemd/system/mon-api.service` :

```ini
[Unit]
Description=Mon API Node.js
After=network.target postgresql.service
Wants=postgresql.service

[Service]
Type=simple
User=api
Group=api
WorkingDirectory=/srv/mon-api
ExecStart=/usr/bin/node server.js
Restart=on-failure
RestartSec=5
Environment=NODE_ENV=production
EnvironmentFile=/etc/mon-api/env

# Durcissement : le service ne voit qu'un système minimal
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
ProtectHome=true
ReadWritePaths=/srv/mon-api/uploads

[Install]
WantedBy=multi-user.target
```

Puis `systemctl daemon-reload && systemctl enable --now mon-api`.

Les directives de durcissement de la fin méritent l'attention : elles utilisent les *namespaces*
du noyau pour donner au service une vue restreinte du système. `ProtectSystem=strict` rend tout le
système de fichiers en lecture seule sauf les chemins listés dans `ReadWritePaths`. Une faille
dans votre application ne permet alors plus d'écrire ailleurs. Vérifiez le résultat avec
`systemd-analyze security mon-api.service`.

---

## 6. Le système de fichiers

### 6.1 L'arborescence unique

Linux n'a pas de lettres de lecteur. Tout part d'une racine `/` et les périphériques
supplémentaires sont **montés** dans des sous-répertoires. La norme FHS
(*Filesystem Hierarchy Standard*) fixe le rôle de chaque dossier :

| Chemin | Contenu | À connaître pour un serveur |
|---|---|---|
| `/etc` | Fichiers de configuration | C'est ici qu'on travaille et qu'on sauvegarde |
| `/var` | Données variables : logs, bases, files | `/var/log`, `/var/lib/mysql`, `/var/lib/postgresql` |
| `/srv` | Données servies par la machine | Emplacement recommandé pour les sites |
| `/home` | Répertoires personnels | Peu utilisé côté serveur |
| `/usr` | Programmes et bibliothèques installés | Géré par le gestionnaire de paquets |
| `/opt` | Logiciels tiers autonomes | Installations manuelles |
| `/tmp` | Temporaire, vidé au redémarrage | Souvent en RAM (tmpfs) |
| `/proc`, `/sys` | Fenêtres virtuelles sur le noyau | Lecture seule d'information |
| `/dev` | Fichiers de périphériques | `/dev/sda`, `/dev/null` |
| `/boot` | Noyau, initramfs, GRUB | Partition souvent petite, à surveiller |

### 6.2 Les systèmes de fichiers

**ext4** est le choix par défaut historique : robuste, journalisé, sans surprise.
**XFS** gère mieux les très gros fichiers et le parallélisme, c'est le défaut sur RHEL.
**Btrfs** et **ZFS** apportent les instantanés (*snapshots*), les sommes de contrôle par bloc et
la compression transparente, au prix d'une complexité supérieure.

**LVM** (*Logical Volume Manager*) s'intercale entre les disques physiques et les systèmes de
fichiers. Il permet d'agrandir un volume à chaud, de le répartir sur plusieurs disques et de
prendre des instantanés avant une migration risquée. Sur un serveur, c'est presque toujours un
bon investissement.

### 6.3 Montage et espace disque

```bash
lsblk                    # arbre des disques et partitions
df -hT                   # espace libre par système de fichiers, avec le type
df -i                    # inodes libres : un disque peut être plein d'inodes sans être plein !
du -sh /var/log/*        # ce qui pèse dans un dossier
mount | column -t        # ce qui est monté et comment
cat /etc/fstab           # montages permanents appliqués au démarrage
```

Le piège classique : `df -h` dit qu'il reste de la place, mais l'écriture échoue. Deux causes
fréquentes, les inodes épuisés (`df -i`, typique d'un dossier de sessions PHP avec des millions
de petits fichiers), ou un fichier supprimé mais toujours ouvert par un processus, dont l'espace
n'est libéré qu'à la fermeture (`lsof +L1`).

---

## 7. Utilisateurs, groupes et permissions

### 7.1 Le modèle de base

Chaque fichier a un propriétaire, un groupe, et neuf bits de permission : lecture, écriture,
exécution, pour le propriétaire, le groupe et les autres.

```
-rw-r--r--  1 www-data www-data  1256 sept.  6 10:12 index.html
│└┬┘└┬┘└┬┘     └───┬──┘ └───┬──┘
│ │  │  │          │        └── groupe
│ │  │  └── autres │
│ │  └───── groupe └─────────── propriétaire
│ └──────── propriétaire
└────────── type : - fichier, d dossier, l lien
```

En notation octale, `r=4`, `w=2`, `x=1`. Donc `chmod 644` donne `rw-r--r--` et `chmod 750` donne
`rwxr-x---`. Sur un dossier, `x` signifie "traverser", pas "exécuter" : sans lui, on ne peut pas
entrer dans le dossier même en ayant `r`.

```bash
chown -R www-data:www-data /srv/site
chmod 750 /srv/site
find /srv/site -type f -exec chmod 640 {} +
find /srv/site -type d -exec chmod 750 {} +
```

### 7.2 Les comptes de service

Règle d'or : **un service ne tourne jamais en `root`**. Chaque démon a son compte système dédié,
sans mot de passe et sans shell de connexion (`www-data` pour Apache/Nginx sur Debian, `mysql`,
`postgres`, `redis`). Si le service est compromis, l'attaquant n'obtient que les droits de ce
compte.

```bash
useradd --system --no-create-home --shell /usr/sbin/nologin monapp
id www-data
```

### 7.3 sudo et l'escalade contrôlée

On se connecte avec un compte nominatif, et on élève ses droits ponctuellement avec `sudo`, ce qui
laisse une trace dans les journaux. Les règles vivent dans `/etc/sudoers.d/`, à éditer avec
`visudo` (qui valide la syntaxe avant d'enregistrer : une erreur ici peut vous verrouiller dehors).

### 7.4 Au-delà des neuf bits

Les **ACL** (`setfacl`, `getfacl`) permettent des droits par utilisateur supplémentaires.
Les **attributs étendus** (`chattr +i` rend un fichier immuable, même pour root).
Les **capabilities** découpent les pouvoirs de root en une quarantaine de privilèges séparés :
c'est ainsi qu'un serveur web peut écouter sur le port 80 sans être root
(`setcap 'cap_net_bind_service=+ep' /usr/bin/monserveur`).

---

## 8. Processus, mémoire et ordonnancement

### 8.1 Anatomie d'un processus

Un processus possède un PID, un parent (PPID), un utilisateur effectif, un espace mémoire virtuel,
une table de descripteurs de fichiers ouverts, et un ou plusieurs fils d'exécution (*threads*).
Il naît par `fork` (duplication du parent) suivi d'`execve` (remplacement du code).

**États** : R (en cours ou prêt), S (sommeil interruptible, l'état normal d'un démon en attente),
D (sommeil non interruptible, typiquement bloqué sur une entrée/sortie disque), Z (zombie,
terminé mais non récupéré par son parent), T (arrêté).

Un pic de processus en état `D` signale presque toujours un problème de disque ou de stockage
réseau, pas un manque de CPU.

### 8.2 Observer

```bash
ps aux --sort=-%mem | head          # les plus gros consommateurs de mémoire
top    # ou mieux : htop
pidstat 1                           # consommation par processus, seconde par seconde
lsof -p 1234                        # tout ce qu'ouvre le processus 1234
ss -lntp                            # qui écoute sur quels ports, avec le processus
uptime                              # charge moyenne sur 1, 5 et 15 minutes
vmstat 1                            # mémoire, swap, entrées/sorties, CPU
iostat -xz 1                        # saturation des disques
```

La **charge moyenne** (*load average*) compte les processus prêts à tourner **plus** ceux bloqués
en entrée/sortie. Une charge de 4,00 sur une machine à 4 cœurs est un plein régime sain. La même
valeur sur 1 cœur signale une file d'attente.

### 8.3 Signaux

```bash
kill -TERM 1234    # demande polie d'arrêt, le programme peut nettoyer (signal 15, défaut)
kill -HUP 1234     # convention : relire la configuration
kill -KILL 1234    # exécution immédiate par le noyau, aucun nettoyage (signal 9, dernier recours)
pkill -f 'node server.js'
```

### 8.4 cgroups et namespaces

Ces deux mécanismes du noyau sont la fondation de tout le monde des conteneurs, et ils servent
déjà sans Docker.

Les **cgroups** (*control groups*) limitent et comptabilisent les ressources d'un groupe de
processus : mémoire maximale, part de CPU, débit disque. systemd place chaque service dans son
propre cgroup, donc `MemoryMax=512M` dans une unité suffit à plafonner un service.

Les **namespaces** isolent la *vue* qu'un processus a du système : ses propres PID, son propre
réseau, ses propres points de montage, ses propres utilisateurs. Un processus dans un namespace
PID se croit seul avec le PID 1.

Un conteneur, ce n'est rien d'autre que : des namespaces pour l'isolation, des cgroups pour les
quotas, et une image de système de fichiers en couches.

### 8.5 Le tueur de mémoire

Quand la RAM et le swap sont épuisés, le noyau déclenche l'**OOM killer** et tue le processus au
score le plus élevé, souvent la base de données, qui est le plus gros consommateur. À vérifier
systématiquement après un arrêt inexpliqué :

```bash
dmesg -T | grep -i 'killed process'
journalctl -k | grep -i oom
```

---

## 9. Le réseau

### 9.1 Les couches, en pratique

| Couche | Élément | Outils |
|---|---|---|
| Lien | Carte réseau, adresse MAC | `ip link`, `ethtool` |
| Réseau | IP, routage | `ip addr`, `ip route`, `ping`, `traceroute` |
| Transport | TCP (fiable, ordonné), UDP (rapide, sans garantie) | `ss`, `nc` |
| Application | HTTP, SSH, SMTP, DNS, PostgreSQL | `curl`, `dig`, `psql` |

### 9.2 Ports et sockets

Un service écoute sur un couple adresse IP + port. Les ports sous 1024 sont privilégiés (root ou
capability requise). Les repères à connaître : 22 SSH, 25 SMTP, 53 DNS, 80 HTTP, 443 HTTPS,
3306 MySQL/MariaDB, 5432 PostgreSQL, 6379 Redis, 27017 MongoDB, 9200 Elasticsearch.

```bash
ss -lntup                          # tout ce qui écoute, TCP et UDP, avec le processus
ss -s                              # statistiques de connexions
curl -I https://example.com        # en-têtes HTTP seulement
dig +short example.com             # résolution DNS
traceroute example.com
```

**Règle de sécurité fondamentale** : une base de données ne doit écouter que sur `127.0.0.1` ou
sur un réseau privé, jamais sur `0.0.0.0`, sauf nécessité explicite et pare-feu en place.
Vérifiez la colonne d'adresse locale dans `ss -lntp`.

### 9.3 Pare-feu

Le filtrage est dans le noyau (Netfilter). Les outils d'administration sont des interfaces vers
lui : `nftables` (moderne), `iptables` (historique), `ufw` (simple, Debian/Ubuntu),
`firewalld` (RHEL).

```bash
ufw default deny incoming
ufw default allow outgoing
ufw allow 22/tcp
ufw allow 80,443/tcp
ufw enable
ufw status verbose
```

Le principe : tout fermer par défaut, ouvrir uniquement ce qui est nécessaire. Et **toujours
autoriser SSH avant d'activer le pare-feu**, sinon la session en cours est coupée et le serveur
devient inaccessible.

---

## 10. La pile logicielle d'un serveur : la carte des composants

Un serveur applicatif typique assemble sept familles de composants. Voici la carte complète, avec
les représentants les plus courants.

```
   Internet
      │
      ▼
┌───────────────┐   1. Répartiteur / proxy inverse
│ HAProxy       │      Nginx, HAProxy, Traefik, Caddy
│ Nginx         │      Répartit la charge, termine le TLS, met en cache
└───────┬───────┘
        ▼
┌───────────────┐   2. Serveur web
│ Apache        │      Apache httpd, Nginx, Caddy, LiteSpeed
│ Nginx         │      Sert les fichiers statiques, parle HTTP
└───────┬───────┘
        ▼
┌───────────────┐   3. Serveur d'application / runtime
│ PHP-FPM       │      PHP-FPM, Node.js, Gunicorn/uWSGI (Python),
│ Node, Tomcat  │      Puma (Ruby), Tomcat (Java), .NET Kestrel
└───┬───────┬───┘      Exécute votre code métier
    │       │
    ▼       ▼
┌────────┐ ┌────────┐  4. Cache mémoire        5. Base de données
│ Redis  │ │MariaDB │     Redis, Memcached        MariaDB/MySQL, PostgreSQL,
│Memcach.│ │Postgres│                             SQLite, MongoDB
└────────┘ └────────┘
    │
    ▼
┌───────────────┐   6. File de messages / tâches asynchrones
│ RabbitMQ      │      RabbitMQ, Kafka, Redis Streams, NATS
│ Kafka         │      Découple les traitements longs
└───────────────┘
        +
┌───────────────┐   7. Services transverses
│ Postfix (SMTP)│      Messagerie, DNS (BIND, Unbound), stockage objet (MinIO),
│ Prometheus    │      recherche (Elasticsearch, OpenSearch), supervision
└───────────────┘
```

### Les acronymes de pile

**LAMP** : Linux + Apache + MySQL/MariaDB + PHP. La pile historique du web, celle de WordPress.
**LEMP** : le E se prononce "engine-x", donc Linux + Nginx + MySQL + PHP. Même chose avec Nginx.
**LAPP** : la variante PostgreSQL.
**MEAN/MERN** : MongoDB + Express + Angular ou React + Node.js, la pile tout-JavaScript.

Ces sigles décrivent des choix, pas des lois. On mélange librement : Nginx en frontal qui délègue
à Apache, PostgreSQL pour les données métier et Redis pour les sessions, c'est une combinaison
parfaitement courante.

---

## 11. Les serveurs web : Apache et Nginx

### 11.1 Le rôle d'un serveur web

Il écoute sur 80 et 443, interprète le protocole HTTP, décide si la requête correspond à un
fichier sur disque (contenu statique) ou doit être transmise à un programme (contenu dynamique),
gère le TLS, la compression, les en-têtes de cache et les journaux d'accès.

### 11.2 Apache HTTP Server

Né en 1995, c'est le serveur web le plus documenté au monde. Son architecture repose sur des
**MPM** (*Multi-Processing Modules*), interchangeables, qui déterminent comment il traite la
concurrence :

| MPM | Modèle | Usage |
|---|---|---|
| `prefork` | Un processus par connexion, sans threads | Obligatoire avec `mod_php` (bibliothèques non thread-safe). Gourmand en RAM |
| `worker` | Processus multiples, plusieurs threads chacun | Moins de mémoire par connexion |
| `event` | Comme worker, mais un thread dédié gère les connexions en attente | Le défaut moderne, recommandé |

Le MPM `event` a comblé l'essentiel de l'écart de performance avec Nginx sur les connexions
persistantes, qui était le reproche historique fait à Apache.

**Sa force réelle** : le système de modules dynamiques et la configuration décentralisée.
`mod_rewrite` pour la réécriture d'URL, `mod_ssl`, `mod_proxy`, `mod_security` (pare-feu
applicatif). Et surtout les fichiers `.htaccess` : chaque dossier peut porter sa propre
configuration, sans droits d'administration ni redémarrage. C'est ce qui a fait le succès
d'Apache chez les hébergeurs mutualisés, et c'est aussi son coût principal en performance,
puisqu'Apache doit chercher un `.htaccess` dans chaque dossier du chemin, à chaque requête.
Si vous contrôlez le serveur, désactivez-les avec `AllowOverride None` et mettez les règles
dans la configuration principale.

**Organisation des fichiers (Debian/Ubuntu)** :

```
/etc/apache2/apache2.conf          configuration principale
/etc/apache2/sites-available/      tous les hôtes virtuels définis
/etc/apache2/sites-enabled/        liens symboliques vers ceux qui sont actifs
/etc/apache2/mods-available/       modules disponibles
/var/log/apache2/access.log        journal d'accès
/var/log/apache2/error.log         journal d'erreurs
```

Sur RHEL, tout est dans `/etc/httpd/` et le service s'appelle `httpd`.

**Un hôte virtuel type** :

```apache
<VirtualHost *:443>
    ServerName site.example.com
    DocumentRoot /srv/site/public

    SSLEngine on
    SSLCertificateFile /etc/letsencrypt/live/site.example.com/fullchain.pem
    SSLCertificateKeyFile /etc/letsencrypt/live/site.example.com/privkey.pem

    <Directory /srv/site/public>
        AllowOverride None
        Require all granted
    </Directory>

    # Délégation du PHP à PHP-FPM via une socket Unix
    <FilesMatch \.php$>
        SetHandler "proxy:unix:/run/php/php8.3-fpm.sock|fcgi://localhost"
    </FilesMatch>

    ErrorLog ${APACHE_LOG_DIR}/site-error.log
    CustomLog ${APACHE_LOG_DIR}/site-access.log combined
</VirtualHost>
```

```bash
a2ensite site.conf && a2enmod ssl proxy_fcgi
apachectl configtest       # TOUJOURS avant de recharger
systemctl reload apache2
```

### 11.3 Nginx

Créé en 2004 par Igor Sysoev pour résoudre le "problème des 10 000 connexions simultanées".
Son architecture est radicalement différente : un processus maître (qui lit la configuration et
possède les ports privilégiés) et un petit nombre de **processus travailleurs**, généralement un
par cœur. Chaque travailleur est **mono-thread** et gère des milliers de connexions par une
**boucle d'événements non bloquante** basée sur `epoll`.

La différence de fond : Apache en prefork alloue une pile mémoire complète par connexion (de
l'ordre de plusieurs mégaoctets), Nginx alloue une structure de quelques kilooctets. Sur dix
mille connexions lentes ou maintenues ouvertes, l'écart devient décisif.

**En contrepartie**, Nginx n'exécute jamais de code applicatif dans son processus. Il n'existe pas
d'équivalent de `mod_php` : le PHP part toujours vers PHP-FPM. Et il n'y a pas de `.htaccess` :
toute la configuration est centralisée, ce qui est plus rapide et plus sûr, mais moins souple
pour un hébergement partagé.

**Configuration type** :

```nginx
upstream app_backend {
    least_conn;
    server 10.0.0.11:3000 max_fails=3 fail_timeout=30s;
    server 10.0.0.12:3000 max_fails=3 fail_timeout=30s;
}

server {
    listen 443 ssl http2;
    server_name site.example.com;
    root /srv/site/public;

    ssl_certificate     /etc/letsencrypt/live/site.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/site.example.com/privkey.pem;

    # Les fichiers statiques : servis directement, avec un cache long
    location ~* \.(jpg|png|css|js|woff2)$ {
        expires 30d;
        access_log off;
    }

    # Le reste : transmis à l'application
    location / {
        proxy_pass http://app_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

server {
    listen 80;
    server_name site.example.com;
    return 301 https://$host$request_uri;
}
```

```bash
nginx -t                   # test de configuration, indispensable
systemctl reload nginx
```

### 11.4 Comment choisir

| Critère | Apache | Nginx |
|---|---|---|
| Fichiers statiques, forte concurrence | Bon (MPM event) | Excellent |
| Proxy inverse, répartition de charge | Correct | Excellent, c'est son terrain |
| Configuration par dossier (`.htaccess`) | Oui | Non |
| Modules dynamiques riches | Très riche | Plus limité en version libre |
| Hébergement mutualisé | Idéal | Peu adapté |
| Mémoire par connexion | Élevée en prefork | Très faible |

En pratique, une architecture très répandue met **Nginx en frontal** (TLS, statique, répartition)
devant **Apache ou un runtime applicatif** en arrière-plan. Les deux ne sont pas concurrents dans
ce montage, ils sont complémentaires. Et si le choix reste ouvert sur un projet neuf : Nginx pour
une API ou une application moderne, Apache si vous héritez d'un écosystème PHP avec des
`.htaccess` partout.

---

## 12. Les bases de données relationnelles : MariaDB/MySQL et PostgreSQL

### 12.1 Ce que fait un SGBD

Un système de gestion de base de données stocke des données durablement, garantit leur cohérence
même en cas de coupure de courant, et permet à des centaines de clients de lire et écrire
simultanément sans se marcher dessus. Il assure les propriétés **ACID** : atomicité (une
transaction passe entièrement ou pas du tout), cohérence, isolation (les transactions concurrentes
ne se voient pas mutuellement en cours de route), durabilité (une transaction validée survit à un
crash).

Le mécanisme central de la durabilité est le **journal d'écriture anticipée** (*Write-Ahead Log*) :
avant de modifier les fichiers de données, le moteur écrit l'intention dans un journal séquentiel
et le force sur disque. Après un crash, il rejoue ce journal pour retrouver un état cohérent.

### 12.2 MySQL et MariaDB

MySQL est né en 1995. Après son rachat par Oracle en 2010, ses créateurs originaux ont lancé
**MariaDB**, un fork libre resté largement compatible : mêmes commandes, même protocole réseau,
mêmes bibliothèques clientes. Sur Debian et Ubuntu, `apt install mysql-server` installe
souvent MariaDB. Les deux ont divergé depuis (MariaDB a ses propres moteurs et son propre système
de réplication), mais pour un usage applicatif standard ils restent interchangeables.

**Architecture** : un seul processus (`mysqld`) et **un thread par connexion**. Les threads sont
plus légers que les processus, donc MySQL supporte nativement un grand nombre de connexions
directes, ce qui explique en partie sa réputation de simplicité opérationnelle.

Sa particularité historique est la séparation entre l'analyseur SQL et les **moteurs de stockage**
enfichables :

- **InnoDB** : le seul choix raisonnable aujourd'hui. Transactionnel, verrouillage par ligne,
  clés étrangères, récupération après crash.
- **MyISAM** : ancien, verrouillage par table, non transactionnel. À éviter, ne subsiste que dans
  du code hérité.
- **Aria**, **ColumnStore** (analytique), **Memory** : spécifiques à MariaDB ou à des besoins
  particuliers.

**Fichiers importants** :

```
/etc/mysql/                        configuration (my.cnf, conf.d/)
/var/lib/mysql/                    données
/var/log/mysql/error.log           journal d'erreurs
/var/lib/mysql/ib_logfile*         journal InnoDB
```

**Le réglage qui compte plus que tous les autres** : `innodb_buffer_pool_size`. C'est le cache en
RAM des pages de données et d'index. Sur un serveur dédié à la base, comptez 60 à 70 % de la RAM
totale. Un buffer pool trop petit transforme chaque requête en lectures disque.

```sql
SHOW ENGINE INNODB STATUS\G
SHOW PROCESSLIST;                          -- requêtes en cours
SHOW VARIABLES LIKE 'innodb_buffer_pool%';
EXPLAIN SELECT * FROM commandes WHERE client_id = 42;
```

```bash
mysql_secure_installation                  # à exécuter après toute installation
mysqldump -u root -p --single-transaction --all-databases > sauvegarde.sql
```

### 12.3 PostgreSQL

Descendant du projet POSTGRES de Berkeley (années 1980), PostgreSQL est réputé pour sa rigueur et
sa richesse fonctionnelle. Là où MySQL a longtemps privilégié la vitesse sur les cas simples,
PostgreSQL a privilégié la correction et l'expressivité.

**Architecture** : **un processus par connexion**, forké par le processus superviseur
(*postmaster*). C'est plus coûteux qu'un thread, d'où une règle pratique importante : au-delà de
quelques centaines de connexions, on interpose un **gestionnaire de pool** comme PgBouncer, qui
multiplexe des milliers de connexions clientes sur quelques dizaines de connexions serveur.
Autour tournent des processus auxiliaires : `checkpointer`, `background writer`, `WAL writer`,
`autovacuum launcher`.

**MVCC et VACUUM** : PostgreSQL implémente le contrôle de concurrence multiversion en conservant
plusieurs versions physiques d'une même ligne. Une écriture ne modifie pas la ligne existante,
elle en crée une nouvelle version et marque l'ancienne comme morte. Conséquence : les lecteurs ne
bloquent jamais les écrivains et réciproquement, mais les versions mortes s'accumulent. Le
processus **autovacuum** les recycle en arrière-plan. Un autovacuum mal réglé ou saturé provoque
le fameux *table bloat* : une table qui occupe dix fois sa taille utile. C'est le principal point
de surveillance spécifique à PostgreSQL.

**Ce qu'il apporte de plus** : types de données riches (JSONB indexable, tableaux, plages,
géométries), index variés (B-tree, GIN, GiST, BRIN), CTE récursives, fonctions de fenêtrage,
extensions (**PostGIS** pour le géospatial, **TimescaleDB** pour les séries temporelles,
**pgvector** pour la recherche vectorielle), et des contraintes d'intégrité réellement appliquées.

**Fichiers importants** :

```
/etc/postgresql/16/main/postgresql.conf    configuration du moteur
/etc/postgresql/16/main/pg_hba.conf        authentification : QUI peut se connecter, D'OÙ, COMMENT
/var/lib/postgresql/16/main/               données
/var/lib/postgresql/16/main/pg_wal/        journaux d'écriture anticipée
```

`pg_hba.conf` est le fichier le plus souvent responsable des erreurs de connexion. Chaque ligne
associe un type de connexion, une base, un utilisateur, une adresse source et une méthode
(`scram-sha-256`, `peer`, `md5`). Les règles sont évaluées dans l'ordre, la première qui
correspond gagne.

```sql
SELECT * FROM pg_stat_activity;                 -- sessions et requêtes en cours
SELECT * FROM pg_stat_user_tables;              -- statistiques, dont le dernier vacuum
EXPLAIN (ANALYZE, BUFFERS) SELECT ...;          -- plan d'exécution réel
```

```bash
sudo -u postgres psql
pg_dump -Fc mabase > mabase.dump               # format compressé, restaurable sélectivement
pg_basebackup -D /sauvegarde -Fp -Xs -P        # copie physique complète, base d'un PITR
```

**Le réglage principal** : `shared_buffers`, environ 25 % de la RAM (PostgreSQL s'appuie
volontairement aussi sur le cache disque du noyau, contrairement à InnoDB), plus
`effective_cache_size` à environ 50 à 75 % pour informer le planificateur, et `work_mem` par
opération de tri.

### 12.4 Tableau comparatif

| | MariaDB / MySQL | PostgreSQL |
|---|---|---|
| Modèle de concurrence | Thread par connexion | Processus par connexion |
| Concurrence | MVCC via InnoDB | MVCC natif, VACUUM requis |
| Moteurs de stockage | Enfichables (InnoDB, Aria...) | Un seul, intégré |
| Conformité SQL | Bonne, avec des tolérances | Très stricte |
| JSON | Type JSON, indexation limitée | JSONB avec index GIN, très puissant |
| Extensibilité | Modérée | Extensions, types et opérateurs personnalisés |
| Réplication | Simple à mettre en place, très mature | Streaming physique, logique, PITR |
| Terrain de prédilection | Web classique, CMS, lectures massives | Métier complexe, données géo, analytique, intégrité forte |
| Écosystème | WordPress, Drupal, Magento | Django, Rails moderne, data engineering |

**Comment choisir** : PostgreSQL est le défaut raisonnable pour une nouvelle application, surtout
si le modèle de données est riche ou si l'intégrité compte. MariaDB s'impose quand l'écosystème
l'exige (WordPress et la plupart des CMS PHP), quand l'équipe le maîtrise déjà, ou pour des charges
de lecture simples et massives. La différence de performance brute entre les deux est aujourd'hui
bien plus faible que la différence entre une requête indexée et une requête qui ne l'est pas.

### 12.5 SQLite, le cas particulier

SQLite n'est pas un serveur : c'est une bibliothèque qui écrit dans un fichier unique, sans
processus ni port. Parfait pour les tests, les applications embarquées, la configuration locale.
Avec le mode WAL, il tient très bien des charges de lecture élevées, mais il ne gère qu'un
écrivain à la fois et ne convient pas à plusieurs serveurs applicatifs partageant la même base.

---

## 13. Caches, NoSQL, files de messages et moteurs de recherche

### 13.1 Redis

Base de données clés-valeurs en mémoire, mono-thread pour les commandes (donc chaque opération est
atomique sans verrou), avec des structures de données évoluées : chaînes, listes, ensembles,
ensembles ordonnés, tables de hachage, flux, HyperLogLog. Latence de l'ordre de la fraction de
milliseconde.

**Usages typiques** : cache de résultats de requêtes SQL, stockage des sessions utilisateur,
files de tâches (Sidekiq, Bull, Celery), compteurs et limitation de débit, verrous distribués,
classements en temps réel.

Persistance optionnelle en deux modes : **RDB** (instantanés périodiques, compact, rapide au
redémarrage, perte possible des dernières minutes) et **AOF** (journal de chaque écriture, plus
sûr, fichier plus gros). On peut activer les deux.

Point de vigilance : sans mot de passe et exposé sur Internet, Redis est une porte ouverte.
Configurez `bind 127.0.0.1`, `requirepass`, et une politique d'éviction `maxmemory-policy` adaptée
(`allkeys-lru` pour un cache pur).

**Memcached** est l'alternative historique : plus simple, multi-thread, uniquement des paires
clé-valeur, sans persistance. Redis l'a largement supplanté grâce à ses structures de données.

### 13.2 MongoDB

Base orientée documents : les enregistrements sont des documents de type JSON binaire (BSON),
regroupés en collections, sans schéma imposé. Elle brille quand la structure des données varie
d'un enregistrement à l'autre, ou pour un prototypage rapide. Le partitionnement horizontal
(*sharding*) est intégré.

Le revers : sans schéma imposé par la base, la cohérence devient la responsabilité entière de
l'application, et les jointures restent coûteuses. Beaucoup d'équipes reviennent au relationnel
avec JSONB dans PostgreSQL, qui offre la souplesse documentaire tout en gardant les transactions
et les jointures.

### 13.3 Files de messages

Elles découplent les traitements : le serveur web dépose un message et répond immédiatement, un
travailleur séparé traite la tâche longue (envoi d'e-mail, génération de PDF, encodage vidéo).

**RabbitMQ** implémente le protocole AMQP avec un routage riche (échanges, files, clés de
routage). Idéal pour distribuer des tâches à des travailleurs.
**Apache Kafka** est un journal distribué et persistant : les messages sont conservés et
rejouables, plusieurs consommateurs indépendants lisent le même flux à leur rythme. Conçu pour de
très gros volumes d'événements.
**Redis Streams** et **NATS** offrent des solutions plus légères.

### 13.4 Moteurs de recherche

**Elasticsearch** et son fork libre **OpenSearch** indexent du texte avec Lucene et fournissent
une recherche pleine texte pertinente, du filtrage à facettes et des agrégations. Ils servent
aussi de socle à la centralisation de logs (la pile ELK : Elasticsearch, Logstash, Kibana).
Attention, ce sont de gros consommateurs de RAM (JVM) et ils ne remplacent pas une base de
référence : on y indexe une copie des données, la vérité reste dans le SGBD.

---

## 14. Conteneurs et orchestration

### 14.1 Ce qu'est vraiment un conteneur

Comme vu en 8.4, un conteneur est un processus Linux ordinaire, isolé par des **namespaces** et
plafonné par des **cgroups**, qui voit comme racine une **image de système de fichiers en
couches**. Il n'y a pas de machine virtuelle, pas de second noyau : tous les conteneurs d'un hôte
partagent le noyau de cet hôte. C'est pourquoi un conteneur démarre en quelques dizaines de
millisecondes là où une VM met des dizaines de secondes.

| | Machine virtuelle | Conteneur |
|---|---|---|
| Isolation | Matérielle, très forte | Noyau partagé, plus fine |
| Démarrage | Dizaines de secondes | Millisecondes |
| Empreinte | Gigaoctets (OS complet) | Mégaoctets |
| Noyau | Le sien | Celui de l'hôte |

### 14.2 Docker et Podman

**Docker** a popularisé le format d'image et l'outillage. **Podman** offre la même interface en
ligne de commande sans démon central et sans root, ce qui plaît en environnement contraint.
Le format d'image est standardisé (OCI), donc les images sont interchangeables.

```bash
docker ps                          # conteneurs en cours
docker logs -f mon-conteneur
docker exec -it mon-conteneur bash # entrer dans un conteneur
docker stats                       # consommation en direct
docker compose up -d               # démarrer une pile complète décrite en YAML
```

Un `docker-compose.yml` typique reprend exactement la pile de la section 10 : un service `nginx`,
un service `app`, un service `postgres` avec un volume persistant, un service `redis`. Les données
d'une base **doivent** être dans un volume nommé, sinon elles disparaissent avec le conteneur.

### 14.3 Kubernetes

Quand une application dépasse un seul serveur, Kubernetes orchestre les conteneurs sur un ensemble
de machines : il place les charges, redémarre ce qui tombe, ajuste le nombre de répliques,
distribue le trafic et déroule les mises à jour progressivement. C'est puissant et coûteux en
complexité. Pour un site ou une API sur un ou deux serveurs, `systemd` avec Docker Compose, ou
même sans conteneurs du tout, reste souvent le choix le plus sage.

---

## 15. Le trajet complet d'une requête HTTP

Voici ce qui se passe réellement quand un visiteur ouvre `https://site.example.com/produits/42`.
Cette chronologie relie toutes les couches vues précédemment.

1. **DNS** : le navigateur résout `site.example.com` en adresse IP, en interrogeant le résolveur
   configuré, qui remonte éventuellement jusqu'aux serveurs faisant autorité.
2. **TCP** : poignée de main en trois temps (SYN, SYN-ACK, ACK) vers le port 443. Côté serveur,
   c'est le noyau qui l'accepte et la place dans la file d'écoute de la socket.
3. **TLS** : négociation de la version et des algorithmes, envoi du certificat, vérification par le
   client de la chaîne de confiance, dérivation des clés de session. À partir de là tout est
   chiffré.
4. **HTTP** : le client envoie `GET /produits/42` avec ses en-têtes.
5. **Serveur web** : Nginx (ou Apache) reçoit la requête, la fait correspondre à un bloc `server`
   grâce à l'en-tête `Host`, puis choisit un `location`. Si la cible est un fichier statique, il
   le lit sur disque (via le cache du noyau) et le renvoie : la chaîne s'arrête ici, en une
   fraction de milliseconde.
6. **Passage à l'application** : sinon, il transmet la requête au runtime via une socket Unix ou
   TCP : PHP-FPM en FastCGI, ou un processus Node/Gunicorn/Puma en HTTP.
7. **Code applicatif** : le routeur identifie le contrôleur, qui interroge d'abord **Redis**
   (`produit:42`). En cas de succès, on saute l'étape suivante.
8. **Base de données** : sinon, requête SQL vers PostgreSQL ou MariaDB. Le moteur analyse la
   requête, choisit un plan (index ou parcours complet), lit les pages depuis son cache mémoire ou
   depuis le disque, applique l'isolation transactionnelle, renvoie les lignes. Le résultat est
   mis en cache dans Redis avec une durée de vie.
9. **Rendu** : le gabarit est assemblé en HTML.
10. **Retour** : la réponse remonte au serveur web, qui la compresse (gzip ou brotli), ajoute les
    en-têtes de cache et de sécurité, l'écrit dans la socket TLS.
11. **Journalisation** : une ligne dans `access.log`, avec le code de statut et le temps de
    réponse. Les métriques sont exposées pour Prometheus.
12. **Asynchrone** : si une tâche longue a été déclenchée, un message est déposé dans RabbitMQ et
    un travailleur la traitera hors du chemin de la requête.

**Où le temps se perd, par ordre de fréquence** : une requête SQL sans index, un appel réseau
externe synchrone, le problème des N+1 requêtes, l'absence de cache, et enfin seulement le CPU
applicatif. Diagnostiquez toujours dans cet ordre.

---

## 16. Sécurité

### 16.1 Accès SSH

C'est la porte d'entrée, donc la première à durcir. Dans `/etc/ssh/sshd_config` :

```
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
AllowUsers admin deploy
```

L'authentification par clé remplace le mot de passe : la clé privée reste sur votre poste, la
publique va dans `~/.ssh/authorized_keys` sur le serveur. Testez toujours une nouvelle session
dans un second terminal **avant** de fermer la première, sinon une erreur de configuration vous
enferme dehors. Ajoutez `fail2ban`, qui bannit temporairement les adresses IP après des échecs
répétés.

### 16.2 Les principes qui comptent

**Moindre privilège** : chaque service sous son propre compte, aucun démon en root, sudo nominatif
et journalisé.
**Surface d'attaque minimale** : désinstallez ce qui ne sert pas, fermez tous les ports sauf ceux
qui sont nécessaires, ne faites écouter les bases que sur la boucle locale.
**Mises à jour** : `unattended-upgrades` sur Debian/Ubuntu ou `dnf-automatic` sur RHEL pour les
correctifs de sécurité, avec une revue régulière des paquets nécessitant un redémarrage
(`needrestart`).
**Chiffrement en transit** : TLS partout, certificats Let's Encrypt renouvelés automatiquement par
`certbot`, protocoles anciens désactivés (TLS 1.2 minimum).
**Défense en profondeur** : SELinux (RHEL) ou AppArmor (Debian/Ubuntu) confinent chaque programme
à un profil de comportement autorisé. Quand un service se comporte étrangement, vérifiez d'abord
`ausearch -m avc` ou `dmesg | grep DENIED` avant de désactiver le mécanisme, qui est là pour vous.

### 16.3 Sauvegardes

Une sauvegarde qui n'a jamais été restaurée n'est pas une sauvegarde. Appliquez la règle **3-2-1** :
trois copies, sur deux supports différents, dont une hors site. Testez la restauration à intervalle
régulier, sur une machine séparée, chronomètre en main : c'est le seul moyen de connaître votre
délai réel de reprise.

Pour une base de données, distinguez la sauvegarde **logique** (`mysqldump`, `pg_dump` : portable,
lente à restaurer sur de gros volumes) de la sauvegarde **physique** (`pg_basebackup`, snapshot
LVM : rapide, liée à la version du moteur). Pour PostgreSQL, l'archivage continu des WAL permet une
restauration à un instant précis (*Point In Time Recovery*), ce qui sauve la mise quand un
`DELETE` sans clause `WHERE` est passé en production.

---

## 17. Observabilité : logs, métriques, sauvegardes

**Les trois piliers** : les *logs* racontent ce qui s'est passé, les *métriques* mesurent des
tendances, les *traces* suivent une requête à travers les composants.

### Les journaux

```bash
journalctl -u nginx --since today
journalctl -p err -b                     # erreurs du démarrage courant
tail -f /var/log/nginx/access.log
```

`journald` centralise les logs des services systemd en format structuré et indexé. Les
applications continuent souvent d'écrire dans `/var/log/`, où **logrotate** se charge de la
rotation et de la compression pour éviter de saturer `/var`. Vérifiez sa configuration sur tout
nouveau service : un log qui remplit le disque fait tomber tout le serveur.

### Les métriques

Le trio courant : **Prometheus** collecte (en interrogeant des exportateurs : `node_exporter` pour
la machine, `nginx-exporter`, `postgres_exporter`), **Grafana** affiche, **Alertmanager** notifie.

**Les quatre signaux dorés** à surveiller sur tout service : latence, trafic, taux d'erreurs,
saturation. Complétez avec les indicateurs propres au système : espace disque restant (avec une
alerte à 80 %, pas à 95 %), mémoire disponible, charge moyenne rapportée au nombre de cœurs,
expiration des certificats TLS.

---

## 18. Travaux pratiques

À faire sur une machine virtuelle jetable ou un conteneur, jamais sur un serveur en production.

### TP 1 : reconnaissance (30 minutes)
Sur un serveur existant, répondez par des commandes : quelle distribution et quelle version de
noyau ? Combien de cœurs et de RAM ? Quels services écoutent sur le réseau, et sous quel
utilisateur tournent-ils ? Combien d'espace reste-t-il sur `/var` ? Quels services sont en échec ?
Quel a été le service le plus lent au dernier démarrage ?

### TP 2 : une pile LEMP complète (2 heures)
Installez Nginx, PostgreSQL et un runtime applicatif. Créez un compte système dédié pour
l'application, un hôte virtuel qui sert les fichiers statiques directement et délègue le reste,
une unité systemd durcie pour le runtime, et un certificat TLS. Vérifiez chaque étape avec
`nginx -t`, `systemctl status` et `curl -I`.

### TP 3 : diagnostiquer une lenteur (1 heure)
Chargez une table de cent mille lignes, écrivez une requête sur une colonne non indexée, mesurez
avec `EXPLAIN ANALYZE`. Ajoutez l'index, remesurez, et lisez la différence dans le plan
d'exécution. Puis mettez le résultat en cache dans Redis et comparez les trois temps.

### TP 4 : survivre à une panne (1 heure)
Faites une sauvegarde de votre base. Supprimez une table. Restaurez. Chronométrez. Ensuite,
provoquez volontairement une saturation du disque avec `fallocate -l 5G /var/gros-fichier` et
observez le comportement des services, puis nettoyez. Notez quels signaux vous auraient alerté à
temps.

---

## 19. Quiz de révision

1. Quelle est la différence entre l'espace utilisateur et l'espace noyau, et où passe-t-on de l'un
   à l'autre ?
2. Pourquoi `free -h` affiche-t-il si peu de mémoire libre sur un serveur sain ?
3. Que fait l'initramfs et pourquoi ne peut-on pas toujours s'en passer ?
4. Quelle est la différence entre `systemctl restart` et `systemctl reload` ?
5. Un `df -h` montre 40 % d'espace libre mais l'écriture échoue. Citez deux causes possibles.
6. Pourquoi le MPM `prefork` d'Apache consomme-t-il beaucoup plus de mémoire que Nginx à
   concurrence égale ?
7. Qu'est-ce qu'un `.htaccess`, quel est son avantage et quel est son coût ?
8. Expliquez le rôle du VACUUM dans PostgreSQL. Que se passe-t-il s'il ne tourne pas ?
9. Quelle est la différence architecturale principale entre MySQL et PostgreSQL en matière de
   gestion des connexions, et quelle conséquence pratique en découle ?
10. Quel réglage est le plus déterminant pour les performances d'InnoDB, et à quelle valeur ?
11. Qu'est-ce qu'un conteneur, en termes de mécanismes du noyau ?
12. Dans le trajet d'une requête HTTP, citez les trois endroits les plus fréquents où le temps se
    perd.
13. Pourquoi ne doit-on jamais activer un pare-feu sans avoir d'abord autorisé le port 22 ?
14. Que signifie la règle 3-2-1 pour les sauvegardes, et quelle vérification manque-t-il souvent ?

---

## 20. Glossaire

**ACID** : atomicité, cohérence, isolation, durabilité. Les garanties transactionnelles d'un SGBD.
**cgroup** : mécanisme du noyau limitant les ressources d'un groupe de processus.
**Démon** (*daemon*) : programme fonctionnant en arrière-plan, sans terminal.
**epoll** : mécanisme du noyau permettant de surveiller des milliers de sockets efficacement.
**FastCGI** : protocole entre serveur web et interpréteur applicatif, utilisé par PHP-FPM.
**FHS** : norme définissant le rôle de chaque répertoire de l'arborescence.
**Inode** : structure décrivant un fichier (droits, dates, blocs). Leur nombre est fini.
**MPM** : module de traitement de la concurrence d'Apache.
**MVCC** : contrôle de concurrence par versions multiples des lignes.
**Namespace** : isolation de la vue qu'un processus a du système.
**Proxy inverse** : serveur en frontal qui relaie les requêtes vers des serveurs d'arrière-plan.
**Socket** : point de communication réseau, couple adresse et port.
**Swap** : espace disque servant d'extension à la RAM pour les pages inactives.
**Systemd unit** : fichier déclaratif décrivant un service, une socket, un montage ou une cible.
**WAL** : journal d'écriture anticipée garantissant la durabilité après un crash.

---

## Pour aller plus loin

- La documentation officielle de votre distribution (Debian Administrator's Handbook, Red Hat
  System Administration).
- `man` reste la meilleure référence : `man 7 signal`, `man 5 proc`, `man systemd.exec`.
- Les documentations d'Apache, Nginx, PostgreSQL et MariaDB sont d'excellente qualité et vieillissent
  bien, contrairement aux tutoriels de blog.
- Certifications structurantes si vous voulez un parcours balisé : LPIC-1, RHCSA.
