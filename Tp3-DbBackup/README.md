# Module 3 : Sauvegarde de base de données

## I. Script dump

➜ **Créer un utilisateur DANS LA BASE DE DONNEES**

```sql
MariaDB [(none)]> CREATE USER 'db_dumps'@'localhost' IDENTIFIED BY 'root';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON nextcloud.* TO 'db_dumps'@'localhost';
MariaDB [(none)]> FLUSH PRIVILEGES;
MariaDB [(none)]> exit
```
```bash
sudo systemctl restart mariadb
sudo mysql -u db_dumps -p
sudo dnf install tar -y
```

➜ **Ecrire le script `bash`**

- il s'appellera `tp3_db_dump.sh`
- il devra être stocké dans le dossier `/srv` sur la machine `db.tp2.linux`
- le script doit commencer par un *shebang* qui indique le chemin du programme qui exécutera le contenu du script

- le script doit contenir une commande `mysqldump`
  - qui récupère le contenu de la base de données `nextcloud`
  - en utilisant l'utilisateur précédemment créé
- le fichier `.sql` produit doit avoir **un nom précis** :
  - il doit comporter le nom de la base de données dumpée
  - il doit comporter la date, l'heure la minute et la seconde où a été effectué le dump
  - par exemple : `db_nextcloud_2211162108.sql`
- enfin, le fichier `sql` doit être compressé
  - au format `.zip` ou `.tar.gz`
  - le fichier produit sera stocké dans le dossier `/srv/db_dumps/`
  - il doit comporter la date, l'heure la minute et la seconde où a été effectué le dump

```bash
#!/bin/bash

user='restore'
password='toto'
dbname='nextcloud'
ip_server='localhost'
datesave=$(date '+%y%m%d%H%M%S')
name="${dbname}_${datesave}"
outputpath="/srv/dbname_dumps/${name}.sql"

# Dump

echo "Backup started for database - ${dbname}."
mysqldump -h ${ip_server} -u ${user} -p${password} --skip-lock-tables --databases ${dbname} > $outputpath
if [[ $? == 0 ]]
then
        gzip -c $outputpath > "${outputpath}.gz"
        rm -f $outputpath
        echo "Backup successfully completed."
else
        echo "Backup failed."
        rm -f outputpath
        exit 1
fi
```

**Bien sur on change les permissions du fichier pour qu'il puisse s'exécuter :**
```bash
sudo chmod 744 tp3_db_dump.sh
ls -al
total 4
drwxr-xr-x.  3 root root  44 Nov 23 11:23 .
dr-xr-xr-x. 18 root root 235 Sep 30 15:28 ..
drwxr-xr-x.  2 root root   6 Nov 23 11:20 db_dumps
-rwxr--r--.  1 root root 654 Nov 23 11:23 tp3_db_dump.sh
```

## II. Clean it

On va rendre le script un peu plus propre vous voulez bien ?

➜ **Utiliser des variables** déclarées en début de script pour stocker les valeurs suivantes :

- utilisateur de la base de données utiliser pour dump
- son password
- le nom de la base
- l'IP à laquelle la commande `mysqldump` se connecte
- le nom du fichier `.tar.gz` ou `.zip` produit par le script

**On a déja fait cette partie dans la partie d'avant dans notre premier script**

---

➜ **Commentez le script**

- au minimum un en-tête sous le shebang
  - date d'écriture du script
  - nom/pseudo de celui qui l'a écrit
  - un résumé TRES BREF de ce que fait le script

**Ajout du créateur du script :**
```bash
sudo cat tp3_db_dump.sh
#!/bin/bash
# Last Update : 23/11/2022
# This script will dump the database and save it to a file
# Creator : Mehdi SENHAJ
```

---

➜ **Environnement d'exécution du script**

- créez un utilisateur sur la machine `db.tp2.linux`
  - il s'appellera `db_dumps`
  - son homedir sera `/srv/db_dumps/`
  - son shell sera `/usr/bin/nologin`

```bash
sudo useradd db_dumps -m -d /srv/db_dumps/ -s /usr/bin/nologin
useradd: Warning: missing or non-executable shell '/usr/bin/nologin'
useradd: warning: the home directory /srv/db_dumps/ already exists.
useradd: Not copying any file from skel directory into it.
```

- cet utilisateur sera celui qui lancera le script
- le dossier `/srv/db_dumps/` doit appartenir au user `db_dumps`
```bash
sudo chown db_dumps:db_dumps tp3_db_dump.sh
ls -al
-rwxr--r--.  1 db_dumps db_dumps 683 Nov 23 11:27 tp3_db_dump.sh
```

- pour tester l'exécution du script en tant que l'utilisateur `db_dumps`, utilisez la commande suivante :

```bash
sudo -u db_dumps /srv/tp3_db_dump.sh
Backup started for database - nextcloud.
Backup successfully completed.

ls -al
total 52
drwxr-xr-x. 2 db_dumps db_dumps    43 Nov 23 11:49 .
drwxr-xr-x. 3 root     root        44 Nov 23 11:46 ..
-rw-r--r--. 1 db_dumps db_dumps 52285 Nov 23 11:49 nextcloud_221115091918.sql.gz
```

**On essaye de Unzip la backup qu'on vient de faire pour voir si ca marche bien :**
```bash
sudo gzip -d nextcloud_221115091918.sql.gz
cat nextcloud_221115091918.sql
-- MariaDB dump 10.19  Distrib 10.5.16-MariaDB, for Linux (x86_64)
--
-- Host: localhost    Database: nextcloud
-- ------------------------------------------------------
-- Server version       10.5.16-MariaDB-log

/*!40101 SET @OLD_CHARACTER_SET_CLIENT=@@CHARACTER_SET_CLIENT */;
/*!40101 SET @OLD_CHARACTER_SET_RESULTS=@@CHARACTER_SET_RESULTS */;
/*!40101 SET @OLD_COLLATION_CONNECTION=@@COLLATION_CONNECTION */;
/*!40101 SET NAMES utf8mb4 */;
/*!40103 SET @OLD_TIME_ZONE=@@TIME_ZONE */;
/*!40103 SET TIME_ZONE='+00:00' */;
/*!40014 SET @OLD_UNIQUE_CHECKS=@@UNIQUE_CHECKS, UNIQUE_CHECKS=0 */;
/*!40014 SET @OLD_FOREIGN_KEY_CHECKS=@@FOREIGN_KEY_CHECKS, FOREIGN_KEY_CHECKS=0 */;
/*!40101 SET @OLD_SQL_MODE=@@SQL_MODE, SQL_MODE='NO_AUTO_VALUE_ON_ZERO' */;
/*!40111 SET @OLD_SQL_NOTES=@@SQL_NOTES, SQL_NOTES=0 */;
```


## III. Service et timer

➜ **Créez un *service*** système qui lance le script

- inspirez-vous du *service* créé à la fin du TP1
- la seule différence est que vous devez rajouter `Type=oneshot` dans la section `[Service]` pour indiquer au système que ce service ne tournera pas à l'infini (comme le fait un serveur web par exemple) mais se terminera au bout d'un moment
- vous appelerez le service `db-dump.service`
- assurez-vous qu'il fonctionne en utilisant des commandes `systemctl`

```bash
cd /etc/systemd/system
cat db_dump.service
[Unit]
Description=Dump the nextcloud database

[Service]
ExecStart=/srv/tp3_db_dump.sh
Type=oneshot
User=db_dumps
WorkingDirectory=/srv/db_dumps

[Install]
WantedBy=multi-user.target
```

**Ajout des permissions pour le script**
```bash
sudo chmod 754 tp3_db_dump.sh
ls -al
total 4
drwxr-xr-x.  3 root     root      44 Nov 23 11:46 .
dr-xr-xr-x. 18 root     root     235 Sep 30 15:28 ..
drwxr-xr-x.  2 db_dumps db_dumps  40 Nov 23 11:50 db_dumps
-rwxr-xr--.  1 db_dumps db_dumps 683 Nov 23 11:48 tp3_db_dump.sh
```

**On lance le service :**
```bash
sudo systemctl start db_dump
sudo systemctl status db_dump
○ db_dump.service - Dump the nextcloud database
     Loaded: loaded (/etc/systemd/system/db_dump.service; disabled; vendor preset: disabled)
     Active: inactive (dead)

Nov 23 11:56:07 db.tp2.linux systemd[1]: Starting Dump the nextcloud database...
Nov 23 11:56:07 db.tp2.linux tp3_db_dump.sh[1307]: Backup started for database - nextcloud.
Nov 23 11:56:07 db.tp2.linux tp3_db_dump.sh[1307]: Backup successfully completed.
Nov 23 11:56:07 db.tp2.linux systemd[1]: db_dump.service: Deactivated successfully.
Nov 23 11:56:07 db.tp2.linux systemd[1]: Finished Dump the nextcloud database.
```

➜ **Créez un *timer*** système qui lance le *service* à intervalles réguliers

- le fichier doit être créé dans le même dossier
- le fichier doit porter le même nom
- l'extension doit être `.timer` au lieu de `.service`
- ainsi votre fichier s'appellera `db-dump.timer`

```bash
sudo cat db_dump.timer
[Unit]
Description=Run service X

[Timer]
OnCalendar=*-*-* 4:00:00

[Install]
WantedBy=timers.target
```

- une fois le fichier créé :

```bash
sudo systemctl daemon-reload
sudo systemctl start db_dump.timer
sudo systemctl enable db_dump.timer
Created symlink /etc/systemd/system/timers.target.wants/db_dump.timer → /etc/systemd/system/db_dump.timer.
sudo systemctl status db_dump.timer
● db_dump.timer - Run service X
     Loaded: loaded (/etc/systemd/system/db_dump.timer; enabled; vendor preset: disabled)
     Active: active (waiting) since Wed 2022-11-23 16:40:28 CET; 10s ago
      Until: Wed 2022-11-23 16:40:28 CET; 12s ago
    Trigger: Thu 2022-11-24 04:00:00 CET; 16h left
   Triggers: ● db_dump.service

Nov 23 16:40:28 db.tp2.linux systemd[1]: Started Run service X.

sudo systemctl list-timers
NEXT                        LEFT          LAST                        PASSED       UNIT                         ACTIVATES
Wed 2022-11-23 16:43:18 CET 2min 31s left Wed 2022-11-23 15:01:28 CET 1h 39min ago    dnf-makecache.timer          dnf-makecache.service
Thu 2022-11-24 00:00:00 CET 7h left      Wed 2022-11-23 12:35:25 CET 4h 5min ago  logrotate.timer              logrotate.service
Thu 2022-11-24 04:00:00 CET 11h left      n/a                         n/a          db_dump.timer                db_dump.service
Thu 2022-11-24 10:10:47 CET 18h left      Wed 2022-11-23 10:10:47 CET 6h 30min ago systemd-tmpfiles-clean.timer systemd-tmpfiles-clean.service

4 timers listed.
```

➜ **Tester la restauration des données** sinon ça sert à rien :)

**Pour récupérer les données comment faire ?**
- Première étape récupérer la dernière sauvegarde qu'on a fait avec le service :
```bash
ls -lt /srv/db_dumps/ | head -2
total 304
-rw-r--r--. 1 db_dumps db_dumps  52285 Nov 23 11:56 nextcloud_221123115607.sql.gz
```

- Deuxième étape on unzip le fichier :
```bash
sudo gzip -d /srv/db_dumps/nextcloud_221123115607.sql.gz
```

- Ensuite on se connecte à la base de données en local :
```bash
mysql -u root -p nextcloud < /srv/db_dumps/db_nextcloud_221123115607.sql
```

- Enfin on fait un dump sur notre base de donnée avec le fichier de la backup qu'on a unzip :
```bash
mysql -u dump -p nextcloud < /srv/db_dumps/db_nextcloud_221123115607.sql
```

**Et voila on a maintenant restaurer les données sur notre base de données !**
