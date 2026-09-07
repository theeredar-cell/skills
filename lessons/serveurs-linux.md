# Comprendre les serveurs Linux, sans jargon

Cette leçon part de zéro. Vous n'avez jamais administré de serveur, et beaucoup de mots
techniques ne vous disent rien : c'est exactement le point de départ prévu.

**Trois promesses.** Chaque mot technique est expliqué la première fois qu'il apparaît, en
français courant. Une seule image sert de fil rouge du début à la fin, celle du restaurant.
Et rien ne vous est demandé d'avance : on construit le vocabulaire au fur et à mesure.

Une version condensée pour ceux qui connaissent déjà le sujet existe dans
[serveurs-linux-reference.md](./serveurs-linux-reference.md). Ne la lisez pas maintenant.

---

## Sommaire

**Partie 1 : les fondations**
1. [Le mot "serveur" veut dire trois choses](#1-le-mot-serveur-veut-dire-trois-choses)
2. [Les 15 mots à connaître avant de commencer](#2-les-15-mots-à-connaître-avant-de-commencer)
3. [L'image du restaurant](#3-limage-du-restaurant)
4. [Ce qu'il y a dans la machine : les couches](#4-ce-quil-y-a-dans-la-machine--les-couches)
5. [Pourquoi Linux, et c'est quoi une distribution](#5-pourquoi-linux-et-cest-quoi-une-distribution)

**Partie 2 : comment le système fonctionne**

6. [Le démarrage, étape par étape](#6-le-démarrage-étape-par-étape)
7. [Les services : les employés qui travaillent en permanence](#7-les-services--les-employés-qui-travaillent-en-permanence)
8. [Les fichiers et les dossiers](#8-les-fichiers-et-les-dossiers)
9. [Les utilisateurs et les droits](#9-les-utilisateurs-et-les-droits)
10. [Les programmes en train de tourner](#10-les-programmes-en-train-de-tourner)
11. [Le réseau : adresses, noms et ports](#11-le-réseau--adresses-noms-et-ports)

**Partie 3 : les composants d'un serveur**

12. [Qui fait quoi dans la pile](#12-qui-fait-quoi-dans-la-pile)
13. [Le serveur web : Apache et Nginx](#13-le-serveur-web--apache-et-nginx)
14. [Les bases de données : MariaDB et PostgreSQL](#14-les-bases-de-données--mariadb-et-postgresql)
15. [Le cache, les files d'attente et la recherche](#15-le-cache-les-files-dattente-et-la-recherche)
16. [Les conteneurs](#16-les-conteneurs)

**Partie 4 : mettre tout ensemble**

17. [Le voyage d'une page web, du clic à l'écran](#17-le-voyage-dune-page-web-du-clic-à-lécran)
18. [Sécurité : les sept réflexes](#18-sécurité--les-sept-réflexes)
19. [Savoir si tout va bien](#19-savoir-si-tout-va-bien)
20. [Vos premiers pas, en pratique](#20-vos-premiers-pas-en-pratique)
21. [Mémo, quiz corrigé et glossaire](#21-mémo-quiz-corrigé-et-glossaire)

---

# Partie 1 : les fondations

## 1. Le mot "serveur" veut dire trois choses

C'est la première source de confusion, et elle bloque beaucoup de débutants. Selon la phrase,
"serveur" désigne l'une de ces trois choses :

1. **Une machine.** Un ordinateur allumé en permanence, quelque part, qui rend service à
   d'autres ordinateurs. "J'ai loué un serveur à 5 euros par mois."
2. **Un logiciel.** Un programme qui attend des demandes et y répond. Apache est un "serveur
   web" : c'est un programme, pas une machine. "J'ai installé un serveur web."
3. **Un rôle.** L'idée générale de répondre aux autres. "Cette machine sert de serveur de
   fichiers."

Dans cette leçon, quand le mot est ambigu, je précise : *la machine serveur* ou *le logiciel
serveur*.

### Client et serveur

C'est le couple de base de tout ce qui suit.

- Le **client**, c'est celui qui demande. Votre navigateur web (Chrome, Firefox, Safari) est un
  client. Votre application mobile aussi.
- Le **serveur**, c'est celui qui répond.

Quand vous tapez une adresse dans votre navigateur, votre navigateur (client) envoie une
**requête** (une demande) à une machine serveur, qui renvoie une **réponse** (la page).

C'est exactement la relation entre un client de restaurant et le personnel du restaurant. D'où
l'image qu'on va garder tout du long.

---

## 2. Les 15 mots à connaître avant de commencer

Lisez ce tableau une fois, sans chercher à le retenir. Vous y reviendrez naturellement. Chaque
mot est réexpliqué en contexte plus loin.

| Le mot | Ce que ça veut dire, simplement | L'image |
|---|---|---|
| **Système d'exploitation** (OS) | Le logiciel de base qui fait marcher l'ordinateur et sur lequel tous les autres programmes s'installent. Windows, macOS et Linux en sont. | Le bâtiment et son règlement intérieur |
| **Noyau** (*kernel*) | Le cœur du système d'exploitation. C'est lui, et lui seul, qui commande le matériel. | Le directeur, seul à avoir les clés du bâtiment |
| **Matériel** | Les composants physiques : processeur, mémoire, disque, carte réseau. | Les murs, les machines, les camions |
| **Processeur** (CPU) | Ce qui calcule. Sa puissance se compte en "cœurs" : un cœur, une tâche à la fois. | Les paires de mains |
| **Mémoire vive** (RAM) | L'espace de travail immédiat, très rapide, mais qui s'efface quand la machine s'éteint. | Le plan de travail |
| **Disque** | Le stockage durable, plus lent, qui survit à l'extinction. | L'entrepôt |
| **Programme** | Un fichier contenant des instructions, au repos sur le disque. | Une recette écrite |
| **Processus** | Un programme en train de s'exécuter, chargé en mémoire. | La recette en train d'être cuisinée |
| **Service** ou **démon** | Un programme qui tourne en permanence en arrière-plan, sans écran, et attend des demandes. | L'employé toujours à son poste |
| **Terminal** | La fenêtre noire où l'on tape des commandes au clavier. C'est l'outil principal sur un serveur. | Le talkie-walkie pour donner des ordres |
| **Commande** | Un ordre tapé dans le terminal. Exemple : `ls` affiche la liste des fichiers. | Une phrase d'ordre |
| **Réseau** | Ce qui relie les machines entre elles, y compris Internet. | Les routes |
| **Adresse IP** | Le numéro d'une machine sur le réseau, comme `192.168.1.10`. | L'adresse postale |
| **Port** | Un numéro qui désigne *quel service* on veut sur cette machine. Le 443 pour les sites sécurisés. | Le numéro du guichet à l'accueil |
| **Requête / réponse** | La demande envoyée par le client, et ce que le serveur renvoie. | La commande passée, le plat servi |

Trois mots supplémentaires qui reviendront souvent :

- **Configuration** : les réglages d'un logiciel, écrits dans des fichiers texte. Sur Linux, on
  ne clique pas dans des menus, on modifie des fichiers.
- **Paquet** : un logiciel prêt à installer, avec tout ce qu'il lui faut. On les récupère depuis
  une sorte de magasin en ligne, avec une commande.
- **Cache** : une copie temporaire d'un résultat, gardée sous la main pour ne pas refaire le
  travail. Comme garder la sauce déjà préparée à côté du feu.

---

## 3. L'image du restaurant

Gardez ce tableau en tête. Chaque composant technique de cette leçon a sa place dedans.

| Dans le restaurant | Sur le serveur | Ce que ça fait |
|---|---|---|
| Le client à table | Le navigateur | Il demande quelque chose |
| La porte et le videur | Le pare-feu | Il laisse entrer ou refuse |
| Le serveur de salle | Le serveur web (Apache, Nginx) | Il prend la commande et apporte l'assiette |
| Les plats déjà préparés en vitrine | Les fichiers statiques (images, feuilles de style) | Servis tels quels, immédiatement |
| La cuisine | L'application (votre code) | Elle prépare ce qui est demandé sur mesure |
| L'entrepôt et ses rayonnages | La base de données (MariaDB, PostgreSQL) | Elle range tout, durablement, et retrouve vite |
| Le frigo à portée de main | Le cache (Redis) | Il garde les choses les plus demandées, très près |
| Le carnet de commandes en attente | La file de messages (RabbitMQ) | Il note ce qui sera fait plus tard |
| Le bâtiment et le règlement | Le système d'exploitation Linux | Il fait tenir l'ensemble |
| Le directeur, seul à avoir les clés | Le noyau | Il arbitre l'accès aux ressources |
| Le cahier de bord | Les journaux (*logs*) | Il note tout ce qui s'est passé |

Quand une notion vous échappe plus loin, revenez à cette ligne-là du tableau.

---

## 4. Ce qu'il y a dans la machine : les couches

Un serveur Linux est organisé en couches empilées. Chaque couche ne parle qu'à ses voisines
immédiates. Du bas vers le haut :

```
   4. VOS PROGRAMMES
      Apache, MariaDB, votre site
              ▲
   3. LES BIBLIOTHÈQUES
      Des morceaux de code tout faits, partagés par tous les programmes
              ▲
      ---- LE GUICHET (les "appels système") ----
              ▲
   2. LE NOYAU LINUX
      Le seul à commander le matériel
              ▲
   1. LE MATÉRIEL
      Processeur, mémoire, disque, carte réseau
```

### Pourquoi cette séparation existe

Le noyau est le seul autorisé à toucher au matériel. Aucun programme ne lit directement le
disque : il **demande** au noyau de le faire pour lui. Cette demande porte un nom technique,
l'**appel système** (*system call*), mais l'idée est simple : c'est un guichet.

Reprenons le restaurant. Un cuisinier ne va pas chercher lui-même la marchandise dans le camion.
Il passe une demande au responsable, qui vérifie qu'il en a le droit et va chercher la
marchandise. C'est plus lent qu'en libre-service, mais cela apporte trois choses décisives :

1. **La sécurité.** Le noyau vérifie chaque demande. Un programme ne peut pas lire un fichier
   qu'il n'a pas le droit de lire.
2. **La stabilité.** Un programme qui plante n'emporte que lui-même. Le reste de la machine
   continue. C'est pour cela qu'un site web peut tomber sans que le serveur s'éteigne.
3. **Le partage.** Cent programmes croient chacun avoir la machine pour eux seuls. C'est le
   noyau qui distribue le temps de calcul et la mémoire entre eux, en alternant très vite.

### Deux territoires

On parle souvent de deux zones :

- L'**espace utilisateur**, au-dessus du guichet : là où vivent tous vos programmes. Un accident
  y reste local.
- L'**espace noyau**, en dessous : le territoire du noyau. Un accident ici fait tomber toute la
  machine. C'est rare, et ça porte un nom, le *kernel panic*.

Vous n'écrirez jamais de code dans l'espace noyau. Mais savoir que cette frontière existe
explique beaucoup de messages d'erreur.

---

## 5. Pourquoi Linux, et c'est quoi une distribution

### Linux, ce n'est que le noyau

Techniquement, "Linux" désigne uniquement le noyau, la couche 2 du schéma. Tout seul, il ne fait
rien d'utile pour un humain : pas de terminal, pas de commandes, pas d'installateur.

Une **distribution** (souvent abrégée "distro") est un assemblage complet et cohérent : le noyau
Linux, plus les outils de base, plus un moyen d'installer des logiciels, plus des réglages par
défaut. C'est une recette d'assemblage, préparée par une équipe ou une entreprise.

| Famille | Distributions | Commande d'installation | Notes |
|---|---|---|---|
| Debian | Debian, Ubuntu Server | `apt install nginx` | La plus répandue, la plus documentée en ligne |
| Red Hat | RHEL, Rocky, AlmaLinux, Fedora | `dnf install nginx` | Très présente en entreprise |
| Alpine | Alpine Linux | `apk add nginx` | Minuscule, surtout utilisée dans les conteneurs |

**Cette leçon utilise les commandes Debian/Ubuntu**, et signale l'équivalent Red Hat quand il
change. Si vous débutez, prenez Ubuntu Server : vous trouverez plus de réponses sur Internet.

### Le gestionnaire de paquets

Sur Windows, on télécharge un fichier `.exe` sur un site. Sur Linux, ce serait une mauvaise
pratique. On utilise un **gestionnaire de paquets** : une commande qui va chercher le logiciel
dans un dépôt officiel, vérifie sa signature, installe aussi tout ce dont il dépend, et sait le
mettre à jour plus tard.

```bash
sudo apt update              # rafraîchit la liste des logiciels disponibles
sudo apt install nginx       # installe le serveur web Nginx
sudo apt upgrade             # met à jour tout ce qui est installé
```

`sudo` signifie "fais ceci en tant qu'administrateur". On y revient au chapitre 9.

### Pourquoi Linux domine sur les serveurs

- **On n'installe que le nécessaire.** Pas d'interface graphique, pas de logiciels inutiles :
  moins de choses à maintenir et moins de failles possibles.
- **Tout se pilote au clavier**, donc tout peut être écrit dans un script et rejoué à
  l'identique sur mille machines.
- **C'est libre et gratuit**, donc dupliquer un serveur ne coûte rien en licences.

---

# Partie 2 : comment le système fonctionne

## 6. Le démarrage, étape par étape

Quand vous allumez la machine, cinq choses se passent dans l'ordre. Savoir cet ordre, c'est
savoir où chercher quand un serveur ne redémarre pas.

**1. Le firmware fait l'appel.**
Un petit programme gravé dans la carte mère (l'UEFI, autrefois le BIOS) vérifie que le matériel
répond, puis cherche sur les disques de quoi démarrer.

**2. Le chargeur d'amorçage choisit le noyau.**
C'est GRUB, le petit menu noir qui apparaît parfois une seconde. Il charge le noyau Linux en
mémoire et lui passe la main.

**3. La trousse à outils de secours (l'initramfs) entre en jeu.**
Problème d'œuf et de poule : pour lire le disque, le noyau a parfois besoin de pilotes qui sont
*sur* ce disque. La solution est un mini-système temporaire chargé en mémoire, contenant juste
les outils nécessaires pour atteindre le vrai disque. Il s'appelle l'**initramfs**. Une fois le
disque monté, il s'efface.

**4. Le premier programme démarre.**
Le noyau lance un programme et un seul, qui porte le numéro 1 et devient l'ancêtre de tous les
autres. Aujourd'hui c'est **systemd**. S'il meurt, toute la machine s'arrête.

**5. systemd ouvre le restaurant.**
Il monte les disques, configure le réseau, puis lance les services les uns après les autres,
en parallèle quand il le peut, jusqu'à ce que la machine soit prête à répondre.

### Si le démarrage est lent

```bash
systemd-analyze              # combien de temps a pris chaque grande phase
systemd-analyze blame        # la liste des services, du plus lent au plus rapide
journalctl -b -p err         # les erreurs du démarrage en cours
```

---

## 7. Les services : les employés qui travaillent en permanence

### Ce qu'est un service

Un **service** (ou **démon**, en anglais *daemon*) est un programme qui tourne en permanence en
arrière-plan et attend qu'on lui demande quelque chose. Il n'a pas de fenêtre, pas de bouton.
Apache est un service. MariaDB est un service. Votre application aussi, une fois installée.

C'est l'employé qui reste à son poste toute la journée, même quand il n'y a personne.

### systemd, le chef du personnel

**systemd** est le programme qui gère tous les services : il les démarre au bon moment, les
redémarre s'ils tombent, et note ce qu'ils racontent.

Les commandes suivent toutes le même moule : `systemctl` + l'action + le nom du service.

```bash
systemctl status nginx     # comment va ce service ? Actif ? Depuis quand ?
systemctl start nginx      # démarre-le maintenant
systemctl stop nginx       # arrête-le
systemctl restart nginx    # arrête puis redémarre
systemctl reload nginx     # relis tes réglages sans t'arrêter
systemctl enable nginx     # démarre-le automatiquement à chaque allumage
systemctl disable nginx    # ne le démarre plus automatiquement
```

Deux pièges de débutant :

- `start` démarre le service **maintenant**, mais ne le fera pas au prochain redémarrage de la
  machine. C'est `enable` qui s'en charge. Pour les deux d'un coup :
  `systemctl enable --now nginx`.
- `restart` coupe le service : les visiteurs en cours sont interrompus. `reload` lui fait relire
  sa configuration sans s'arrêter. **Préférez toujours `reload` quand c'est possible.**

### Le cahier de bord

Chaque service raconte ce qu'il fait. Tout est centralisé et se consulte avec `journalctl` :

```bash
journalctl -u nginx              # tout ce qu'a dit le service nginx
journalctl -u nginx -f           # et continue à me le montrer en direct
journalctl -u nginx --since "1 hour ago"
systemctl list-units --failed    # LA commande à taper quand quelque chose ne marche pas
```

Cette dernière ligne liste tous les services en échec. C'est le premier réflexe de diagnostic.

---

## 8. Les fichiers et les dossiers

### Une seule racine, pas de lettres de lecteur

Sur Windows, chaque disque a sa lettre : `C:`, `D:`. Sur Linux, il n'y en a pas. Tout part d'un
unique point de départ, la **racine**, notée `/`, et tout le reste est un dossier à l'intérieur.

Quand on ajoute un deuxième disque, on ne lui donne pas de lettre : on le **monte**, c'est-à-dire
qu'on le greffe à un dossier existant. Après avoir monté un disque sur `/donnees`, écrire dans
`/donnees` écrit sur ce disque. L'utilisateur ne voit qu'une arborescence continue.

### Les dossiers à connaître

Vous n'avez besoin d'en retenir que six pour commencer.

| Dossier | Ce qu'il contient | Pourquoi il compte pour vous |
|---|---|---|
| `/etc` | Tous les fichiers de réglages | C'est là que vous travaillerez le plus. À sauvegarder en priorité |
| `/var` | Ce qui grossit avec le temps | Contient les journaux et les bases de données |
| `/var/log` | Les journaux de tous les logiciels | Le premier endroit où regarder quand ça casse |
| `/home` | Les dossiers personnels des utilisateurs | Peu utilisé sur un serveur |
| `/srv` | Les données servies au public | Un bon endroit pour mettre votre site |
| `/tmp` | Le temporaire, effacé au redémarrage | N'y rangez jamais rien d'important |

Trois autres que vous croiserez sans avoir à y toucher : `/usr` (les programmes installés),
`/dev` (les périphériques, vus comme des fichiers) et `/proc` (une fenêtre sur ce que fait le
noyau, générée à la volée, ce ne sont pas de vrais fichiers).

### Se déplacer et regarder

```bash
pwd                  # où suis-je ?
ls -la /etc          # liste le contenu d'un dossier, tout compris
cd /var/log          # se déplacer dans un dossier
cat /etc/hostname    # afficher le contenu d'un fichier court
less /var/log/syslog # lire un gros fichier page par page (q pour sortir)
tail -f /var/log/nginx/access.log   # voir les nouvelles lignes en direct
```

### L'espace disque

Un disque plein arrête tout : les bases de données refusent d'écrire, les sites tombent. Deux
commandes suffisent :

```bash
df -h            # combien reste-t-il de place, disque par disque
du -sh /var/log/*   # qu'est-ce qui prend de la place dans ce dossier
```

Dans `df -h`, regardez la colonne d'utilisation. Au-delà de 80 %, il faut agir : c'est le moment
de vérifier `/var/log`, qui grossit sans arrêt si personne ne surveille.

---

## 9. Les utilisateurs et les droits

### root, l'administrateur

Sur Linux, un compte peut absolument tout : il s'appelle **root**. Il n'y a aucun garde-fou, pas
de fenêtre "êtes-vous sûr ?". C'est le passe-partout du bâtiment.

D'où la règle universelle : **on ne travaille pas en root**. On se connecte avec son propre
compte, et on demande ponctuellement les pouvoirs d'administrateur avec `sudo` :

```bash
sudo apt install nginx        # exécute cette commande en tant qu'administrateur
```

Deux avantages : vous ne détruisez pas le système par une faute de frappe, et chaque usage de
`sudo` est noté dans le cahier de bord, donc on sait qui a fait quoi.

### Lire les permissions

Chaque fichier appartient à un **propriétaire** et à un **groupe**, et porte des droits pour
trois catégories de personnes. Quand vous tapez `ls -l`, vous voyez ceci :

```
-rw-r--r--  1 www-data www-data  1256  index.html
```

Décodons de gauche à droite :

- Le premier caractère dit le type : `-` un fichier, `d` un dossier, `l` un raccourci.
- Ensuite trois blocs de trois lettres : les droits du **propriétaire**, ceux du **groupe**,
  ceux de **tous les autres**.
- `r` = lire (*read*), `w` = écrire (*write*), `x` = exécuter, et `-` = pas ce droit.

Donc `rw- r-- r--` se lit : le propriétaire peut lire et modifier, le groupe peut seulement
lire, tout le monde peut seulement lire.

Sur un **dossier**, `x` ne veut pas dire "exécuter" mais "entrer dedans". Un dossier sans `x` est
inaccessible même si vous avez le droit de lecture. C'est une cause d'erreur très fréquente.

### Modifier les droits

```bash
sudo chown -R www-data:www-data /srv/site   # change le propriétaire et le groupe
sudo chmod 750 /srv/site                    # change les droits
```

Les chiffres viennent d'une addition : lire vaut 4, écrire vaut 2, exécuter vaut 1. Un chiffre
par catégorie.

| Chiffre | Calcul | Signifie |
|---|---|---|
| 7 | 4+2+1 | lire, écrire, exécuter |
| 6 | 4+2 | lire, écrire |
| 5 | 4+1 | lire, exécuter |
| 4 | 4 | lire seulement |
| 0 | 0 | rien |

Donc `750` = le propriétaire a tout, le groupe peut lire et entrer, les autres n'ont rien.
Pour un site web, `644` sur les fichiers et `755` sur les dossiers est le réglage habituel.

### Pourquoi chaque service a son propre compte

Vous verrez des comptes bizarres comme `www-data`, `mysql`, `postgres`. Ce ne sont pas des
humains : ce sont des comptes créés pour un service et un seul, sans mot de passe et sans
possibilité de se connecter.

L'intérêt est simple. Si quelqu'un exploite une faille de votre site web, il se retrouve avec les
droits de `www-data`, qui ne peut presque rien faire, et pas avec ceux de root. Dans le
restaurant : le badge du plongeur n'ouvre pas le coffre.

**Aucun service ne doit tourner en root.** C'est la règle de sécurité la plus rentable de toute
cette leçon.

---

## 10. Les programmes en train de tourner

### Processus, mémoire, processeur

Un **processus** est un programme en cours d'exécution. Il porte un numéro unique, le **PID**.
Il occupe de la **mémoire vive** (le plan de travail) et consomme du temps de **processeur**
(les mains qui travaillent).

Un détail utile : un processus peut se diviser en plusieurs **fils d'exécution** (*threads*),
qui travaillent en parallèle en partageant le même plan de travail. Un processus, c'est un
cuisinier avec son poste ; les threads, ce sont ses deux mains qui font deux choses à la fois.

### Regarder ce qui tourne

```bash
top                       # tableau de bord en direct (q pour quitter)
htop                      # la même chose, en plus lisible : à installer
ps aux --sort=-%mem | head   # les 10 programmes qui prennent le plus de mémoire
uptime                    # depuis quand la machine tourne, et sa charge
```

### Comprendre la "charge"

`uptime` affiche trois nombres, par exemple `0.52, 1.10, 0.98`. C'est la **charge moyenne** sur
1, 5 et 15 minutes : le nombre moyen de tâches qui attendaient leur tour.

Pour l'interpréter, comparez-la au nombre de cœurs de votre processeur (`nproc` vous le donne).
Sur une machine à 4 cœurs, une charge de 4 signifie plein régime, sans file d'attente. La même
charge de 4 sur 1 cœur signifie que ça bouchonne sérieusement.

### La mémoire

```bash
free -h
```

**Ne paniquez pas si "free" est presque à zéro.** Linux se sert de toute la mémoire inutilisée
comme cache : il y garde des morceaux de fichiers récemment lus, au cas où. Cette mémoire est
rendue instantanément dès qu'un programme en a besoin. La seule colonne à regarder s'appelle
**available**.

Si en revanche la mémoire manque vraiment, le noyau tue le programme le plus gourmand pour
sauver la machine. C'est souvent la base de données. Pour vérifier après un arrêt inexpliqué :

```bash
dmesg -T | grep -i 'killed process'
```

Si vous y trouvez une ligne, votre machine manque de mémoire : il faut en ajouter ou réduire les
réglages du programme concerné.

### Arrêter un programme

```bash
kill 1234        # demande poliment au processus 1234 de s'arrêter et de ranger
kill -9 1234     # le supprime brutalement, sans lui laisser ranger : dernier recours
```

Utilisez toujours la première forme d'abord. La seconde peut laisser des fichiers dans un état
incohérent, ce qui est particulièrement risqué pour une base de données.

---

## 11. Le réseau : adresses, noms et ports

### Trois notions, dans l'ordre

**L'adresse IP** est le numéro d'une machine sur le réseau, par exemple `93.184.216.34`. C'est
l'adresse postale : précise, mais impossible à retenir.

**Le nom de domaine** est le nom lisible, par exemple `example.com`. C'est le nom dans
l'annuaire.

**Le DNS** est l'annuaire lui-même. Quand vous tapez un nom, votre machine interroge le DNS pour
obtenir l'adresse IP correspondante. Si le DNS est mal configuré, le site est injoignable alors
que le serveur fonctionne parfaitement : c'est une panne très courante et très déroutante.

```bash
dig +short example.com     # à quelle adresse IP correspond ce nom ?
ping example.com           # cette machine répond-elle ?
```

### Les ports

Une machine héberge souvent plusieurs services : un site web, une base de données, un accès à
distance. Comment savoir à qui on s'adresse ? Par le **port**, un numéro entre 1 et 65535. C'est
le numéro du guichet à l'accueil.

| Port | Service | À quoi ça sert |
|---|---|---|
| 22 | SSH | Se connecter au serveur à distance en ligne de commande |
| 80 | HTTP | Le web non chiffré (on redirige vers le 443) |
| 443 | HTTPS | Le web chiffré, celui du cadenas dans le navigateur |
| 3306 | MySQL / MariaDB | Une base de données |
| 5432 | PostgreSQL | Une autre base de données |
| 6379 | Redis | Un cache |

Une adresse particulière revient tout le temps : `127.0.0.1`, aussi appelée `localhost`. Elle
signifie "moi-même, cette machine". Un service qui écoute uniquement sur `127.0.0.1` n'est
joignable que depuis la machine elle-même, jamais depuis Internet. **C'est le bon réglage pour
une base de données.**

```bash
ss -lntp     # quels services écoutent, sur quels ports, et lesquels sont exposés
```

Dans le résultat, regardez la colonne d'adresse locale : `127.0.0.1:5432` est sûr,
`0.0.0.0:5432` signifie "ouvert à tout le monde" et mérite une vérification immédiate.

### Le pare-feu

Le **pare-feu** décide quels ports acceptent des connexions venant de l'extérieur. C'est le
videur à la porte. La bonne pratique est de tout fermer, puis d'ouvrir uniquement le nécessaire.

```bash
sudo ufw default deny incoming    # par défaut, on refuse tout ce qui entre
sudo ufw default allow outgoing   # on autorise ce qui sort
sudo ufw allow 22/tcp             # on ouvre SSH : À FAIRE EN PREMIER
sudo ufw allow 80,443/tcp         # on ouvre le web
sudo ufw enable                   # on active le pare-feu
sudo ufw status verbose           # on vérifie
```

**Attention, erreur classique et douloureuse** : si vous activez le pare-feu sans avoir autorisé
le port 22 d'abord, votre propre connexion est coupée et vous ne pouvez plus entrer. Sur une
machine distante, cela veut dire tout réinstaller.

---

# Partie 3 : les composants d'un serveur

## 12. Qui fait quoi dans la pile

On appelle **pile** (*stack* en anglais) l'ensemble des logiciels empilés pour faire fonctionner
un site ou une application. Voici la pile complète, dans l'ordre où une demande la traverse.

```
   Le visiteur, dans son navigateur
                │
                ▼
   1. LE PARE-FEU          le videur : il laisse entrer ou non
                │
                ▼
   2. LE SERVEUR WEB       Apache ou Nginx
      le serveur de salle : il prend la commande.
      Si le plat est déjà prêt (une image, un fichier), il le donne tout de suite.
                │
                ▼
   3. L'APPLICATION        PHP, Node.js, Python, Java...
      la cuisine : elle prépare la réponse sur mesure
             │        │
             ▼        ▼
   4. LE CACHE      5. LA BASE DE DONNÉES
      Redis            MariaDB ou PostgreSQL
      le frigo         l'entrepôt : tout y est rangé durablement
                │
                ▼
   6. LA FILE DE MESSAGES   RabbitMQ
      le carnet : les tâches longues, faites plus tard
```

Quelques sigles que vous verrez partout, et qui ne désignent que des combinaisons courantes de
cette pile :

- **LAMP** : Linux + Apache + MySQL/MariaDB + PHP. La pile historique du web, celle de WordPress.
- **LEMP** : la même, avec Nginx à la place d'Apache (le E se prononce "eun-jinn-x").
- **MEAN** ou **MERN** : la pile où tout est en JavaScript, avec MongoDB et Node.js.

Ce ne sont pas des règles. On mélange librement selon les besoins.

---

## 13. Le serveur web : Apache et Nginx

### Ce que fait un serveur web

Un **serveur web** est le logiciel qui écoute sur les ports 80 et 443, reçoit les demandes des
navigateurs et renvoie les réponses. C'est le serveur de salle.

Il fait la différence entre deux sortes de contenus :

- Le **contenu statique** : un fichier qui existe déjà sur le disque et qu'on renvoie tel quel.
  Une image, une feuille de style, un fichier JavaScript. C'est le plat déjà en vitrine : on le
  donne immédiatement, ça ne coûte presque rien.
- Le **contenu dynamique** : une page qui doit être fabriquée maintenant, parce qu'elle dépend
  du visiteur ou des données du moment. Votre panier, votre profil, la liste des produits en
  stock. C'est le plat à cuisiner : le serveur web passe la commande à la cuisine.

Il s'occupe aussi du **HTTPS** (le chiffrement, le cadenas dans le navigateur), de la
compression, et il note chaque visite dans son journal.

Les deux logiciels dominants sont Apache et Nginx. Ils font le même travail avec deux
philosophies différentes.

### Apache

Né en 1995, c'est le plus ancien et le plus documenté. Sur Internet, la quasi-totalité des vieux
tutoriels parlent de lui.

**Sa façon de travailler.** Historiquement, Apache attribue un employé à chaque client : un
processus par connexion. C'est simple et robuste, mais chaque employé occupe de la place, et
mille visiteurs simultanés font mille employés. Apache a depuis appris des modes plus économes
(le mode `event`, qui est le réglage moderne recommandé et fait travailler quelques employés très
efficaces), mais sa réputation de gourmandise en mémoire vient de là.

**Ses deux forces.**

1. Les **modules** : des fonctions qu'on ajoute à la demande (réécriture d'adresses, chiffrement,
   pare-feu applicatif). Il y en a pour tout.
2. Les fichiers **`.htaccess`** : un petit fichier de réglages qu'on dépose dans n'importe quel
   dossier, sans être administrateur et sans redémarrer le service. C'est ce qui a fait le succès
   d'Apache chez les hébergeurs bon marché, où chaque client règle son coin sans toucher au
   reste.

**Son coût.** Ces `.htaccess` obligent Apache à vérifier, à chaque demande et dans chaque dossier
du chemin, si un tel fichier existe. C'est du travail répété inutilement. Si vous êtes
administrateur de votre serveur, désactivez-les et mettez vos règles dans la configuration
principale.

**Où sont les fichiers** (sur Debian et Ubuntu) :

```
/etc/apache2/apache2.conf         le réglage principal
/etc/apache2/sites-available/     un fichier par site défini
/etc/apache2/sites-enabled/       les sites réellement activés
/var/log/apache2/access.log       le journal des visites
/var/log/apache2/error.log        le journal des erreurs
```

Sur Red Hat, tout est dans `/etc/httpd/` et le service s'appelle `httpd` au lieu d'`apache2`.

```bash
sudo apachectl configtest    # vérifie que la configuration ne contient pas d'erreur
sudo systemctl reload apache2
```

Prenez l'habitude de toujours vérifier avant de recharger. Une erreur de syntaxe recharge un
service cassé, et le site tombe.

### Nginx

Créé en 2004 pour répondre à un problème précis : tenir dix mille visiteurs en même temps sur une
machine modeste.

**Sa façon de travailler.** Au lieu d'un employé par client, Nginx emploie quelques serveurs de
salle (un par cœur du processeur) qui ne restent jamais plantés à attendre. Dès qu'une table
n'a pas besoin d'eux, ils passent à la suivante, et reviennent quand quelque chose se passe.
C'est le principe de la **boucle d'événements** : on ne bloque jamais, on réagit.

Résultat concret : là où l'ancien Apache réservait plusieurs mégaoctets de mémoire par visiteur,
Nginx en utilise quelques kilooctets. Sur des connexions lentes ou nombreuses, l'écart est
énorme.

**Ses limites.** Nginx n'exécute jamais lui-même le code de votre application : il le transmet
toujours à un programme séparé. Et il n'existe pas de `.htaccess` : toute la configuration est
centralisée, ce qui est plus rapide et plus sûr, mais moins souple si plusieurs personnes se
partagent la machine.

```bash
sudo nginx -t                # vérifie la configuration
sudo systemctl reload nginx
```

### Un exemple de configuration, ligne par ligne

```nginx
server {
    listen 443 ssl;                      # écoute sur le port du web sécurisé
    server_name site.example.com;        # ne répond que pour ce nom de domaine
    root /srv/site/public;               # les fichiers du site sont ici

    ssl_certificate     /etc/letsencrypt/live/site.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/site.example.com/privkey.pem;
    # les deux lignes ci-dessus : le certificat qui prouve l'identité du site

    location ~* \.(jpg|png|css|js)$ {    # pour les images et fichiers de style
        expires 30d;                     # dis au navigateur de les garder 30 jours
    }

    location / {                         # pour tout le reste
        proxy_pass http://127.0.0.1:3000;   # transmets à l'application, sur le port 3000
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;   # dis-lui qui est le vrai visiteur
    }
}

server {
    listen 80;                           # sur le port non sécurisé
    server_name site.example.com;
    return 301 https://$host$request_uri;   # renvoie tout le monde vers la version sécurisée
}
```

Deux mots utiles rencontrés ici :

- **Proxy inverse** (*reverse proxy*) : un serveur placé devant d'autres, qui reçoit les demandes
  et les transmet à celui qui convient. C'est le rôle de `proxy_pass`. Le maître d'hôtel qui
  répartit les clients entre les salles.
- **Certificat** : un fichier signé par une autorité reconnue, qui prouve au navigateur qu'il
  parle bien au vrai site. Let's Encrypt en délivre gratuitement, renouvelés automatiquement par
  l'outil `certbot`.

### Lequel choisir

| Votre situation | Le choix raisonnable |
|---|---|
| Un nouveau projet, une API, une application moderne | Nginx |
| Un site WordPress ou PHP hérité, avec des `.htaccess` partout | Apache |
| Beaucoup de visiteurs simultanés, beaucoup d'images | Nginx |
| Vous suivez un tutoriel qui parle d'Apache et vous débutez | Apache, ne compliquez pas |

Et sachez qu'un montage très répandu utilise **les deux** : Nginx à l'entrée, qui gère le
chiffrement et les images, et Apache derrière pour l'application. Ils ne sont pas rivaux.

---

## 14. Les bases de données : MariaDB et PostgreSQL

### Pourquoi pas simplement des fichiers ?

On pourrait ranger ses données dans des fichiers. Mais dès que plusieurs personnes écrivent en
même temps, tout se casse : deux modifications simultanées s'écrasent, une coupure de courant en
plein milieu laisse un fichier à moitié écrit, et retrouver une ligne parmi un million demande de
tout relire.

Une **base de données** résout ces trois problèmes. C'est l'entrepôt avec un magasinier : lui
seul touche aux rayonnages, il sert plusieurs personnes à la fois sans confusion, et il sait
exactement où se trouve chaque chose.

### Le vocabulaire minimum

- Une **table** est un tableau : une ligne par élément, une colonne par information. La table
  `clients` a une ligne par client, avec les colonnes `nom`, `email`, `date_inscription`.
- **SQL** est la langue dans laquelle on parle à la base. Elle se lit presque comme de l'anglais :
  `SELECT nom FROM clients WHERE ville = 'Lyon'` veut dire "donne-moi le nom des clients dont la
  ville est Lyon".
- Un **index** est la clé de la performance. C'est exactement l'index d'un livre : sans lui, pour
  trouver un mot, il faut lire toutes les pages ; avec lui, on va directement à la bonne. Sur un
  million de lignes, une recherche sans index prend des secondes, avec index quelques
  millisecondes. **C'est de très loin la cause numéro un des sites lents.**
- Une **transaction** est un groupe d'opérations qui doivent réussir ensemble ou échouer
  ensemble. Un virement bancaire retire d'un compte et ajoute à l'autre : si la machine s'éteint
  entre les deux, la base annule tout. Elle ne laisse jamais l'argent disparaître à mi-chemin.

Ces garanties portent un sigle que vous verrez souvent, **ACID**. Retenez surtout la dernière
lettre : une fois que la base vous a dit "c'est enregistré", c'est vrai même si la machine
s'éteint dans la seconde.

### MySQL et MariaDB

MySQL est né en 1995 et a accompagné toute la première génération du web. En 2010, il a été
racheté par Oracle ; ses créateurs d'origine sont partis fonder **MariaDB**, une copie libre qui
a gardé les mêmes commandes et le même fonctionnement.

En pratique, pour un débutant, **MariaDB et MySQL s'utilisent de la même façon**. Sur Debian et
Ubuntu, installer "mysql" installe d'ailleurs souvent MariaDB. Les deux ont divergé depuis, mais
pas sur ce que vous ferez au début.

**Son point fort** : la simplicité et l'omniprésence. WordPress, Drupal, PrestaShop et la
majorité des logiciels PHP sont conçus pour elle. Si vous installez un CMS, ce sera elle.

**Le réglage qui compte vraiment.** MariaDB garde en mémoire une copie des données les plus
utilisées, pour éviter d'aller sur le disque. Ce cache s'appelle `innodb_buffer_pool_size`. S'il
est trop petit, chaque demande va lire le disque et tout rame. Sur une machine dédiée à la base,
donnez-lui environ 60 % de la mémoire totale.

```bash
sudo mysql_secure_installation     # à lancer juste après l'installation
sudo mysql -u root                 # ouvrir une session
mysqldump -u root -p mabase > sauvegarde.sql    # sauvegarder une base
```

### PostgreSQL

Plus ancien encore dans ses racines universitaires, PostgreSQL a une réputation de rigueur. Là où
MySQL a longtemps accepté des approximations pour aller vite, PostgreSQL refuse les données
incohérentes et prévient tout de suite.

**Ses points forts.**

- Il vérifie réellement les règles que vous lui donnez. Si vous déclarez qu'une commande doit
  appartenir à un client existant, il est impossible de créer une commande orpheline.
- Il gère nativement des données modernes : le format JSON avec recherche rapide dedans, les
  coordonnées géographiques (avec l'extension PostGIS), les séries de mesures dans le temps.
- Il s'étend par **extensions**, des modules officiels qui ajoutent des capacités entières.

**Ses deux particularités à connaître.**

1. **Une connexion coûte cher.** PostgreSQL crée un processus séparé par client connecté. Passé
   quelques centaines, la machine souffre. La solution standard s'appelle un *pool* de
   connexions (PgBouncer) : un intermédiaire qui fait patienter tout le monde sur un petit nombre
   de connexions réelles. Vous n'en aurez pas besoin au début, mais retenez le mot.
2. **Le ménage, ou VACUUM.** Quand on modifie une ligne, PostgreSQL n'écrase pas l'ancienne : il
   en écrit une nouvelle version et marque l'ancienne comme périmée. Cela permet à ceux qui
   lisent de ne jamais bloquer ceux qui écrivent. En contrepartie, il faut passer le balai. Un
   processus automatique, l'*autovacuum*, s'en charge. S'il est mal réglé, la base gonfle jusqu'à
   occuper dix fois la place utile. C'est le point de surveillance propre à PostgreSQL.

**Le fichier qui cause 90 % des problèmes de connexion débutants** : `pg_hba.conf`. Il définit qui
a le droit de se connecter, depuis où, et comment. Si votre application dit "connexion refusée"
alors que la base tourne, c'est presque toujours là qu'il faut regarder.

```bash
sudo -u postgres psql              # ouvrir une session
pg_dump -Fc mabase > mabase.dump   # sauvegarder
```

### Comment choisir

| Votre situation | Le choix raisonnable |
|---|---|
| WordPress, Drupal, un CMS PHP | MariaDB, vous n'avez pas le choix et c'est très bien |
| Une nouvelle application métier | PostgreSQL |
| Des données géographiques, du JSON, des calculs complexes | PostgreSQL, sans hésiter |
| Votre équipe connaît déjà l'un des deux | Celui qu'elle connaît |

Un mot d'honnêteté pour finir : entre ces deux moteurs, la différence de vitesse est aujourd'hui
minime comparée à la différence entre une requête indexée et une requête qui ne l'est pas.
Occupez-vous de vos index avant de choisir votre camp.

### Et SQLite ?

SQLite n'est pas un serveur : c'est une bibliothèque qui range tout dans un simple fichier, sans
service à démarrer ni port à ouvrir. Parfait pour apprendre, pour une petite application, pour
des tests. Sa limite : une seule écriture à la fois, et pas de partage entre plusieurs machines.

---

## 15. Le cache, les files d'attente et la recherche

### Redis, le frigo du comptoir

**Redis** garde des données en mémoire vive plutôt que sur le disque. Conséquence : c'est
extrêmement rapide (une fraction de milliseconde), mais la mémoire est chère et limitée. On n'y
met donc pas tout : on y met ce qui est demandé sans arrêt.

Trois usages qui couvrent presque tous les cas :

1. **Le cache.** Une requête à la base coûte 50 millisecondes ? On garde le résultat dans Redis
   pendant 5 minutes. Les milliers de visiteurs suivants sont servis instantanément, sans
   déranger la base.
2. **Les sessions.** Ce qui vous garde connecté d'une page à l'autre. C'est petit, ça change
   souvent, ça n'a pas besoin de survivre éternellement : le profil parfait.
3. **Les files de tâches.** Une liste de choses à faire, dans laquelle des programmes viennent
   piocher.

À savoir : par défaut, Redis ne demande pas de mot de passe. Exposé sur Internet sans réglage,
il est immédiatement pillé. Faites-le écouter sur `127.0.0.1` et donnez-lui un mot de passe.

**Memcached** est un cousin plus ancien et plus simple. Redis l'a largement remplacé.

### Les files de messages

Certaines tâches sont longues : envoyer 5 000 e-mails, générer un PDF, encoder une vidéo. Si le
serveur web les fait pendant la visite, le visiteur attend devant une page bloquée.

La solution est la **file de messages**. Le site dépose un ticket ("il faut envoyer cet e-mail")
et répond immédiatement au visiteur. Un autre programme, appelé **travailleur** (*worker*), prend
les tickets un par un et les traite tranquillement. C'est exactement le carnet de commandes
accroché en cuisine.

Les outils courants : **RabbitMQ** (le classique pour distribuer des tâches), **Kafka** (pour de
très gros volumes d'événements, avec conservation de l'historique), et Redis lui-même pour les
besoins simples.

### La recherche

Une base de données classique cherche mal dans du texte libre : elle ne gère ni les fautes de
frappe, ni les mots proches, ni le classement par pertinence. **Elasticsearch** (et son jumeau
libre **OpenSearch**) fait ce travail.

Un principe à retenir : ce n'est pas là qu'on range la vérité. On y recopie une image des données
pour pouvoir chercher dedans ; l'original reste dans la base de données. Si l'index se perd, on
le reconstruit.

### MongoDB

**MongoDB** range des fiches libres plutôt que des tableaux à colonnes fixes : chaque
enregistrement peut avoir sa propre forme. C'est pratique quand les données sont irrégulières ou
quand on prototype vite.

Le revers : comme la base ne vérifie rien, toute la cohérence repose sur votre code. Beaucoup
d'équipes reviennent aujourd'hui à PostgreSQL, qui sait stocker du JSON souple tout en gardant
les garanties d'un moteur relationnel.

---

## 16. Les conteneurs

### L'idée

Le problème que résolvent les conteneurs est vieux comme l'informatique : "ça marche sur ma
machine, mais pas sur le serveur". Les versions diffèrent, une bibliothèque manque, un réglage
n'est pas le même.

Un **conteneur** est une boîte qui contient l'application **et** tout ce dont elle a besoin pour
tourner. On expédie la boîte entière : elle se comporte partout de la même façon. C'est le plateau
repas préparé en cuisine centrale, qui arrive identique dans chaque train.

### Ce n'est pas une machine virtuelle

Confusion fréquente. Une **machine virtuelle** simule un ordinateur complet avec son propre
système d'exploitation : c'est lourd, plusieurs gigaoctets, et long à démarrer.

Un conteneur, lui, n'emporte pas de système d'exploitation. C'est un programme ordinaire qui
tourne sur le noyau Linux de la machine hôte, mais à qui le noyau ment sur ce qu'il voit : il
croit être seul, avec ses propres dossiers et son propre réseau. D'où sa légèreté.

| | Machine virtuelle | Conteneur |
|---|---|---|
| Contenu | Un système complet | Juste l'application et ses dépendances |
| Taille | Plusieurs gigaoctets | Quelques dizaines de mégaoctets |
| Démarrage | Une minute | Une fraction de seconde |
| Isolation | Très forte | Bonne, mais le noyau est partagé |

### Docker en pratique

**Docker** est l'outil qui a popularisé tout cela. **Podman** fait la même chose sans avoir
besoin des droits d'administrateur.

```bash
docker ps                            # quels conteneurs tournent ?
docker logs -f mon-conteneur         # que raconte celui-ci ?
docker exec -it mon-conteneur bash   # entrer dedans pour regarder
docker compose up -d                 # démarrer toute une pile décrite dans un fichier
```

**Le piège numéro un du débutant** : un conteneur est jetable, et tout ce qu'il contient
disparaît avec lui. Si vous mettez une base de données dans un conteneur sans configurer de
**volume** (un dossier de la machine hôte relié au conteneur), vous perdez toutes vos données au
premier redémarrage. Cette erreur se fait une seule fois dans une vie.

### Kubernetes

Quand une application ne tient plus sur une seule machine, **Kubernetes** répartit les conteneurs
sur un groupe de machines, redémarre ce qui tombe, ajoute des copies quand la charge monte.
C'est puissant, et c'est complexe.

Conseil sincère pour un débutant : vous n'en avez pas besoin. Un serveur, systemd, et
éventuellement Docker Compose suffisent très longtemps.

---

# Partie 4 : mettre tout ensemble

## 17. Le voyage d'une page web, du clic à l'écran

Voici le récit complet. Chaque étape utilise une notion des chapitres précédents : c'est le
moment où tout se relie. Le visiteur ouvre `https://site.example.com/produits/42`.

**1. Trouver l'adresse.** Le navigateur ne connaît que le nom du site. Il interroge le DNS,
l'annuaire, qui lui répond avec l'adresse IP de la machine. *(chapitre 11)*

**2. Frapper à la porte.** Le navigateur établit une connexion vers le port 443 de cette adresse.
C'est le noyau du serveur qui décroche, pas encore l'application. *(chapitres 4 et 11)*

**3. Vérifier l'identité.** Le serveur envoie son certificat. Le navigateur vérifie qu'il est
authentique, puis les deux se mettent d'accord sur une clé de chiffrement. À partir de là,
personne ne peut lire ce qui passe. C'est le cadenas. *(chapitre 13)*

**4. Passer commande.** Le navigateur envoie sa demande : "donne-moi la page /produits/42".

**5. Le serveur de salle reçoit.** Nginx regarde le nom demandé, trouve la configuration du site
concerné, et décide. Si la demande visait une image, il l'envoie directement et **tout s'arrête
ici**, en une fraction de milliseconde. *(chapitre 13)*

**6. Passer en cuisine.** Ici, la page doit être fabriquée. Nginx transmet la demande à
l'application, sur le port 3000 de la même machine. *(chapitre 13)*

**7. Regarder dans le frigo.** L'application demande d'abord à Redis : "as-tu déjà le produit
42 ?" Si oui, on saute l'étape suivante et on gagne 50 millisecondes. *(chapitre 15)*

**8. Aller à l'entrepôt.** Sinon, l'application interroge la base de données. Celle-ci lit sa
requête, choisit son chemin (avec un index, c'est instantané ; sans, elle relit toute la table),
renvoie les lignes. Le résultat est déposé dans Redis pour la prochaine fois. *(chapitre 14)*

**9. Dresser l'assiette.** L'application assemble la page HTML avec les données reçues.

**10. Servir.** La réponse repart vers Nginx, qui la compresse pour qu'elle voyage plus vite,
ajoute quelques en-têtes, et l'envoie au navigateur par la connexion chiffrée.

**11. Noter dans le cahier.** Une ligne s'ajoute au journal des visites : qui, quand, quelle
page, quel temps de réponse. *(chapitre 19)*

**12. La suite, plus tard.** Si la page devait déclencher un e-mail de confirmation, un ticket a
été déposé dans la file. Un travailleur l'enverra dans quelques secondes, sans faire attendre le
visiteur. *(chapitre 15)*

### Quand c'est lent, cherchez dans cet ordre

C'est le réflexe de diagnostic le plus utile de toute la leçon. Les causes, de la plus fréquente
à la plus rare :

1. **Une requête sans index** à l'étape 8. Vérifiez ceci en premier, toujours.
2. **Le problème dit "N+1"** : l'application fait une requête, puis une requête par ligne
   obtenue. 100 produits affichés, 101 allers-retours à la base.
3. **Un appel à un service extérieur** en plein milieu (une API de paiement, un service de
   météo), qui met deux secondes à répondre.
4. **L'absence de cache** à l'étape 7 : on refait à chaque visite un travail identique.
5. **Le code lui-même**, enfin. C'est rarement le coupable, contrairement à l'intuition.

---

## 18. Sécurité : les sept réflexes

Vous n'avez pas besoin d'être expert. Sept habitudes couvrent l'immense majorité des incidents
réels.

**1. Se connecter par clé, pas par mot de passe.**
Une **clé SSH** est une paire de fichiers : une partie privée qui reste sur votre ordinateur et
que vous ne donnez jamais, une partie publique que vous déposez sur le serveur. Un mot de passe
peut se deviner, pas une clé. Interdisez ensuite les mots de passe et la connexion directe en
root dans `/etc/ssh/sshd_config` :

```
PermitRootLogin no
PasswordAuthentication no
```

**Testez toujours une nouvelle connexion dans une deuxième fenêtre avant de fermer la première.**
Si vous vous êtes trompé, la première fenêtre encore ouverte est votre seule porte de secours.

**2. Fermer tout ce qui ne sert pas.** Un pare-feu qui refuse par défaut, et les bases de données
qui écoutent uniquement sur `127.0.0.1`. *(chapitre 11)*

**3. Ne jamais faire tourner un service en root.** Chaque service a son compte dédié.
*(chapitre 9)*

**4. Mettre à jour.** La plupart des serveurs compromis le sont par une faille connue et corrigée
depuis des mois. Activez les mises à jour de sécurité automatiques :

```bash
sudo apt install unattended-upgrades
```

**5. Chiffrer les échanges.** HTTPS partout, avec un certificat gratuit Let's Encrypt renouvelé
automatiquement par `certbot`. Il n'y a plus aucune raison de servir un site en clair.

**6. Ajouter fail2ban.** Ce petit outil surveille les journaux et bannit temporairement les
adresses qui multiplient les tentatives de connexion ratées. Cela élimine le bruit de fond
permanent des robots.

**7. Sauvegarder, et surtout restaurer.**
La règle **3-2-1** : trois copies, sur deux supports différents, dont une ailleurs
géographiquement.

Et la vérité que tout le monde apprend trop tard : **une sauvegarde jamais restaurée n'est pas
une sauvegarde.** Testez la restauration sur une machine séparée, chronomètre en main. Vous
saurez alors combien de temps il vous faudrait vraiment pour repartir. Beaucoup découvrent à ce
moment-là que leurs sauvegardes étaient vides depuis des mois.

---

## 19. Savoir si tout va bien

### Les journaux

Chaque logiciel raconte ce qu'il fait. Quand quelque chose ne marche pas, la réponse est presque
toujours écrite quelque part.

```bash
systemctl list-units --failed          # y a-t-il un service en échec ?
journalctl -u nginx --since today      # que dit ce service aujourd'hui ?
journalctl -p err -b                   # toutes les erreurs depuis le démarrage
tail -f /var/log/nginx/error.log       # voir arriver les erreurs en direct
```

Un conseil qui fait gagner des heures : **lisez le message d'erreur en entier, jusqu'au bout,
avant de chercher sur Internet.** Il contient très souvent le nom du fichier fautif et le numéro
de la ligne.

Attention aussi à un piège : les journaux grossissent sans fin et peuvent remplir le disque, ce
qui arrête tout. L'outil `logrotate` s'en occupe automatiquement, mais vérifiez qu'il est bien
configuré pour tout nouveau service que vous installez.

### Les cinq questions à se poser régulièrement

| Question | La commande | Le seuil d'inquiétude |
|---|---|---|
| Reste-t-il de la place sur le disque ? | `df -h` | Au-delà de 80 % d'occupation |
| La mémoire suffit-elle ? | `free -h` | Colonne *available* qui s'effondre |
| La machine est-elle surchargée ? | `uptime` | Charge durablement supérieure au nombre de cœurs |
| Un service est-il tombé ? | `systemctl list-units --failed` | La liste doit être vide |
| Mon certificat expire-t-il bientôt ? | `certbot certificates` | Moins de 21 jours restants |

Pour aller plus loin, on installe un système de surveillance qui pose ces questions
automatiquement et prévient par e-mail : **Prometheus** collecte les mesures, **Grafana** dessine
les courbes. Mais commencez par prendre l'habitude des cinq commandes ci-dessus.

---

## 20. Vos premiers pas, en pratique

Faites tout ceci sur une machine jetable : une machine virtuelle sur votre ordinateur, ou un
petit serveur loué à quelques euros que vous pourrez détruire ensuite. **Jamais sur une machine
qui sert à quelque chose.**

### Exercice 1 : se connecter et regarder (30 minutes)

Connectez-vous en SSH, puis répondez à ces questions avec une commande chacune. L'objectif n'est
pas de tout comprendre, mais de reconnaître les réponses.

```bash
cat /etc/os-release       # quelle distribution, quelle version ?
uname -r                  # quelle version du noyau ?
nproc                     # combien de cœurs de processeur ?
free -h                   # combien de mémoire ?
df -h                     # combien de place sur le disque ?
uptime                    # depuis combien de temps la machine tourne ?
ss -lntp                  # quels services écoutent sur le réseau ?
systemctl list-units --failed    # quelque chose est-il cassé ?
```

### Exercice 2 : installer un serveur web (45 minutes)

```bash
sudo apt update
sudo apt install nginx
sudo systemctl status nginx        # doit afficher "active (running)" en vert
```

Ouvrez ensuite l'adresse IP de la machine dans votre navigateur. Vous devez voir la page
d'accueil par défaut de Nginx. **Si vous voyez cette page, vous venez de faire fonctionner un
serveur web.**

Puis modifiez-la pour comprendre le lien entre fichier et page affichée :

```bash
echo "<h1>Bonjour depuis mon serveur</h1>" | sudo tee /var/www/html/index.html
```

Rechargez la page dans le navigateur. Si rien ne s'affiche depuis l'extérieur, c'est presque
toujours le pare-feu : vérifiez avec `sudo ufw status` que le port 80 est autorisé.

### Exercice 3 : une base de données (1 heure)

```bash
sudo apt install postgresql
sudo -u postgres psql
```

Vous êtes maintenant dans la base. Tapez :

```sql
CREATE TABLE clients (id serial PRIMARY KEY, nom text, ville text);
INSERT INTO clients (nom, ville) VALUES ('Dupont', 'Lyon'), ('Martin', 'Paris');
SELECT * FROM clients;
SELECT nom FROM clients WHERE ville = 'Lyon';
\q
```

Vous venez de créer une table, d'y mettre deux lignes et de les interroger. C'est tout le
principe des bases de données relationnelles.

### Exercice 4 : casser et réparer (1 heure)

Le plus formateur des quatre. Avant tout, faites une sauvegarde.

1. Arrêtez Nginx (`sudo systemctl stop nginx`) et rechargez la page. Constatez l'erreur du
   navigateur, puis regardez ce que dit `systemctl status nginx`.
2. Introduisez volontairement une faute dans la configuration de Nginx, lancez `sudo nginx -t`,
   et lisez le message : il vous donne le fichier et la ligne. Réparez.
3. Supprimez la table `clients` créée à l'exercice 3, puis restaurez-la depuis votre sauvegarde.
   Chronométrez.

Savoir lire un message d'erreur et restaurer une sauvegarde vaut plus que connaître cent
commandes par cœur.

---

## 21. Mémo, quiz corrigé et glossaire

### Les vingt commandes du quotidien

| Commande | Ce qu'elle fait |
|---|---|
| `ls -la` | Lister le contenu d'un dossier |
| `cd /chemin` | Se déplacer |
| `pwd` | Où suis-je ? |
| `cat fichier` | Afficher un fichier court |
| `less fichier` | Lire un gros fichier (q pour sortir) |
| `tail -f fichier` | Voir les nouvelles lignes en direct |
| `grep motif fichier` | Chercher un texte dans un fichier |
| `nano fichier` | Éditer un fichier simplement |
| `cp`, `mv`, `rm` | Copier, déplacer, supprimer |
| `sudo commande` | Exécuter en administrateur |
| `systemctl status service` | Comment va ce service ? |
| `systemctl restart service` | Le redémarrer |
| `journalctl -u service -f` | Suivre ses messages en direct |
| `df -h` | Place disque restante |
| `free -h` | Mémoire disponible |
| `top` ou `htop` | Ce qui consomme les ressources |
| `ss -lntp` | Qui écoute sur quel port |
| `ps aux` | Tous les processus |
| `apt install paquet` | Installer un logiciel |
| `chown` et `chmod` | Changer propriétaire et droits |

### Quiz corrigé

Répondez d'abord, vérifiez ensuite.

**1. Quelle est la différence entre `systemctl start` et `systemctl enable` ?**
`start` démarre le service maintenant, mais il ne repartira pas après un redémarrage de la
machine. `enable` le programme au démarrage sans le lancer tout de suite. Pour les deux :
`enable --now`.

**2. Pourquoi `free -h` montre-t-il si peu de mémoire libre sur un serveur en bonne santé ?**
Parce que Linux utilise la mémoire inutilisée comme cache de fichiers. Elle est rendue
instantanément si un programme en a besoin. Regardez la colonne *available*.

**3. Que veut dire `x` sur un dossier ?**
Le droit d'entrer dans le dossier, pas de l'exécuter. Sans lui, le dossier est inaccessible même
avec le droit de lecture.

**4. Pourquoi ne faut-il jamais faire tourner un service en root ?**
Parce qu'une faille dans ce service donnerait alors tous les pouvoirs sur la machine. Avec un
compte dédié, l'attaquant reste enfermé dans un périmètre minuscule.

**5. Un service écoute sur `0.0.0.0:5432`. Pourquoi est-ce inquiétant ?**
`0.0.0.0` signifie "accessible depuis n'importe où". Le port 5432 est celui de PostgreSQL : la
base de données est exposée à Internet. Elle devrait écouter sur `127.0.0.1`.

**6. Qu'est-ce qu'un index dans une base de données, et pourquoi est-ce important ?**
C'est l'équivalent de l'index d'un livre : il évite de tout relire pour trouver une ligne. C'est
la première cause de lenteur d'un site quand il manque.

**7. Quelle est la différence entre contenu statique et contenu dynamique ?**
Le statique est un fichier existant renvoyé tel quel (image, CSS) : très rapide. Le dynamique
doit être fabriqué à la demande par l'application, souvent en interrogeant la base.

**8. Pourquoi Nginx tient-il plus de visiteurs simultanés qu'un Apache en configuration
classique ?**
Apache attribuait historiquement un processus par visiteur, chacun coûtant plusieurs mégaoctets.
Nginx emploie quelques travailleurs qui ne restent jamais bloqués à attendre et gèrent des
milliers de connexions chacun.

**9. Qu'est-ce qu'un conteneur, et en quoi diffère-t-il d'une machine virtuelle ?**
C'est une application empaquetée avec tout ce dont elle a besoin, qui tourne sur le noyau de la
machine hôte. Une machine virtuelle simule un ordinateur entier avec son propre système : bien
plus lourde et bien plus lente à démarrer.

**10. Pourquoi faut-il autoriser le port 22 avant d'activer le pare-feu ?**
Parce que le port 22 est celui de SSH, votre connexion. En l'oubliant, vous coupez votre propre
accès et vous ne pouvez plus entrer sur la machine.

**11. Une page met 4 secondes à s'afficher. Par quoi commencez-vous ?**
Par les requêtes à la base de données, et en particulier les index manquants, puis le problème
N+1. Le code applicatif vient en dernier.

**12. Que signifie la règle 3-2-1, et qu'oublie-t-on presque toujours ?**
Trois copies, deux supports, une hors site. On oublie de tester la restauration, qui est
pourtant la seule preuve que la sauvegarde existe vraiment.

### Glossaire

| Terme | Définition simple |
|---|---|
| **Cache** | Copie temporaire d'un résultat, gardée près pour éviter de refaire le travail |
| **Certificat** | Fichier qui prouve l'identité d'un site et permet le chiffrement |
| **Client** | Celui qui demande : votre navigateur |
| **Conteneur** | Application empaquetée avec tout ce qu'il lui faut pour tourner |
| **Démon / service** | Programme qui tourne en permanence en arrière-plan |
| **Distribution** | Assemblage complet : le noyau Linux plus les outils autour |
| **DNS** | L'annuaire qui traduit un nom de domaine en adresse IP |
| **Index** | Table des matières d'une base de données, indispensable à la vitesse |
| **Journal (log)** | Le cahier de bord d'un logiciel |
| **Monter** | Rattacher un disque à un dossier de l'arborescence |
| **Noyau (kernel)** | Le cœur du système, seul à commander le matériel |
| **Paquet** | Un logiciel prêt à installer depuis un dépôt officiel |
| **Pare-feu** | Le filtre qui décide quels ports acceptent des connexions |
| **Port** | Numéro désignant quel service on veut sur une machine |
| **Processus** | Un programme en cours d'exécution |
| **Proxy inverse** | Serveur placé devant d'autres, qui répartit les demandes |
| **Requête** | Une demande envoyée par un client |
| **root** | Le compte administrateur, qui peut tout |
| **SQL** | La langue pour interroger une base de données |
| **SSH** | Le moyen de se connecter à distance en ligne de commande |
| **Transaction** | Groupe d'opérations qui réussissent ou échouent ensemble |
| **Volume** | Dossier persistant relié à un conteneur, pour ne pas perdre les données |

### Pour continuer

- Refaites les quatre exercices sur une machine neuve, sans regarder la leçon.
- Installez un vrai logiciel de bout en bout : WordPress avec MariaDB, ou une petite application
  avec PostgreSQL. C'est en butant sur les vrais problèmes qu'on apprend.
- La commande `man` contient le manuel de tout : `man ls`, `man systemctl`. C'est aride, mais
  c'est toujours exact, contrairement à beaucoup de tutoriels trouvés en ligne.
- Quand vous serez à l'aise, la [fiche de référence](./serveurs-linux-reference.md) reprend les
  mêmes sujets en version dense.
