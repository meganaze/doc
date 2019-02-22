Sauvegarder son serveur et ses apps
===================================

Dans le contexte de l'auto-hébergement, les sauvegardes (backup) sont un élément important pour palier à des événements inattendus (incendies, corruption de base de données, perte d'accès au serveur, serveur compromis, ...). La politique de sauvegardes à mettre en place dépend de l'importance des services et des données que vous gérez. Par exemple, sauvegarder un serveur de test aura peu d'intérêt, tandis que vous voudrez être très prudent si vous gérez des données critiques pour une association ou une entreprise - et dans ce genre de cas, vous souhaiterez stocker les sauvegardes *dans un endroit différent*.

Les sauvegardes avec YunoHost
-----------------------------

YunoHost contient un système de sauvegarde, qui permet de sauvegarder (et restaurer) les configurations du système, les données "système" (comme les mails) et les applications si elles le supportent.

Vous pouvez gérer vos sauvegardes via la ligne de commande (`yunohost backup --help`) ou la webadmin (dans la section Sauvegardes) bien que certaines fonctionnalités ne soient pas disponibles via celle-ci.

Actuellement, la méthode de sauvegarde actuelle consiste à créer des archives `.tar.gz` qui contiennent les fichiers pertinents. Pour le futur, YunoHost envisage de supporter nativement [Borg](https://www.borgbackup.org/) qui est une solution plus flexible, performante et puissante pour gérer des sauvegardes.

Créer des sauvegardes
---------------------

#### Depuis la webadmin

Vous pouvez facilement créer des archives depuis la webadmin en allant dans Sauvegardes > Archives locales et en cliquant sur "Nouvelle sauvegarde". Vous pourrez ensuite sélectionner les éléments à sauvegarder (configuration, données "système", applications).

![](/images/backup.png)

#### Depuis la ligne de commande

Vous pouvez créer de nouvelles archives depuis la ligne de commande. Voici quelques exemples de commandes et leur comportement correspondant:

- Tout sauvegarder (système et application)
```bash
yunohost backup create
```

- Sauvegarder seulement les apps
```bash
yunohost backup create --apps
```

- Sauvegarder seulement deux apps (wordpress et shaarli)
```bash
yunohost backup create --apps wordpress shaarli
```

- Sauvegarder seulement les mails
```bash
yunohost backup create --system data_mail
```

- Sauvegarder les mails et wordpress
```bash
yunohost backup create --system data_mail --apps wordpress
```

Pour plus d'informations et d'options sur la création d'archives, consultez `yunohost backup create --help`. Vous pouvez également lister les parties de système qui sont sauvegardables avec `yunohost hook list backup`.

#### Configuration spécifique à certaines apps

Certaines apps comme Nextcloud sont potentiellement rattachées à des quantités importantes de données. Il est possible de ne pas les sauvegarder par défaut. Dans ce cas, on dit que l'app "sauvegarde uniquement le core" (de l'app). 

Pour activer la sauvegarde de toutes les données de cette application vous pouvez utiliser (dans le cas de Nextcloud) `yunohost app setting nextcloud backup_core_only -v ''`. Cette commande ajoute `backup_core_only:` dans `etc/yunohost/apps/nextcloud/settings.yml` Soyez prudent : en fonction des données stockées dans Nextcloud, il se peut que l'archive que vous obtenez ensuite devienne énorme...

Pour désactiver la sauvegarde de toutes les données de cette application vous pouvez utiliser (dans le cas de Nextcloud) `yunohost app setting nextcloud backup_core_only -v '1'`. Cette commande ajoute `backup_core_only: '1'` dans `etc/yunohost/apps/nextcloud/settings.yml` Soyez prudent : il vous faudra alors sauvegarder vous même les données des utilisateurs de nextcloud. Choisir ce type de sauvegarde vous permettra de mettre en place manuellement des sauvegardes incrémentielles ou différentielles (que yunohost ne permet pas encore de faire automatiquement).

Notes : pour exclure tous le /home du backup utiliser l'argument  : ```bash--system conf_ssh conf_ynh_firewall data_mail conf_cron conf_ynh_certs conf_ynh_mysql conf_xmpp conf_ynh_currenthost conf_nginx conf_ssowat conf_ldap``` (liste tous les hook du système a l'exeption de celui des données de /home)

#### Exemple de script de sauvegardes incrémentielles (et cryptés)

Le script suivant permet de sauvegarder `/home/*` de facon incrémentielle sur un support de votre choix, ainsi que de faire un backup du système et des apps de yunohost (non incrémentielle). Montez votre suport de sauvegarde dans /mnt pour être tranquille. 

Pour les sauvegardes distantes il est possible d'utiliser un contener crypté. Par exemple un contener luks.

```bash
#!/bin/sh

## VARS
source=/home/ # must contain the trailing slash
target=/path/to/backup/folder  # must contain the trailing slash
date=`date '+%Y-%m-%d_%Hh%Mm'`
demonte=/path/to/backup/folder/demonte

# Mount device
mount /dev/sdXX /path/to/backup/folder 

# OR Mount luks container
cryptsetup --key-file /home/qlfacab/backup/lukkey luksOpen /path/to/container containername
mount /dev/mapper/containername /path/to/backup/folder  

# Check disk is mounted : if folder "demonte" exit in the mount path that mean there is nothing mounted in this path -> the script stop
if [ -d "$demonte" ]
  then echo "Connecter le disque de sauvegarde !" 
  exit
fi

# Check if subfolders exists
if [ ! -d "$target"history ]
  then mkdir "$target"history
fi
if [ ! -d "$target"latest ]
  then mkdir "$target"latest
fi

# Delete backups older than 60 days !!be sure about the path - test before lauching script!!
#find /path/to/backup/folder/yunohost -maxdepth 1 -type d -mtime +60 -exec rm -r {} \;
#find /path/to/backup/folder/history -maxdepth 1 -type d -mtime +60 -exec rm -r {} \;


## Ceate yunohost backup (systeme and apps only, /home excluded)
mkdir /path/to/backup/folder/yunohost/"${date}"
yunohost backup create --apps --system conf_ssh conf_ynh_firewall data_mail conf_cron conf_ynh_certs conf_ynh_mysql conf_xmpp conf_ynh_currenthost conf_nginx conf_ssowat conf_ldap --debug --output-directory /path/to/backup/folder/yunohost/"${date}"

## Créé backup  de /home
rsync \
  --exclude-from=/path/to/file/with/list/of/folder/to/be/excluded.txt \
  --progress \
  --verbose \
  --archive \
  --human-readable \
  --delete \
  --delete-excluded \
  -HAX \
  "$source" \
  "${target}"latest

## Backup the backup 
cp -al "${target}"latest "${target}"history/"${date}"

### Umount if necessary 
```
Pour rendre le script exécutable : `chmod - x chemin/vers/le/script`
Ensuite pour lancer le script régulièrement, il suffit d'éditer cron avec`crontab -e`. Par exemple, si vous ajoutez la ligne `* 1 * * * bash /chemin/du/script` la sauvegarde de yunohost et des fichiers utilisateurs aura lieux tous les jours à 1h du matin.
 

Télécharger et téléverser des sauvegardes
-----------------------------------------

Après avoir créé des sauvegardes, il est possible de les lister et de les inspecter grâce aux vues correspondantes dans la webadmin, ou via `yunohost backup list` et `yunohost backup info <nom_d'archive>` depuis la ligne de commande. Par défaut, les sauvegardes sont stockées dans `/home/yunohost.backup/archives/`.

À l'heure actuelle, la solution la plus accessible pour récupérer les sauvegardes est d'utiliser le programme FileZilla comme expliqué dans [cette page](/filezilla_fr).

Une autre solution alternative consiste à installer une application comme Nextcloud et à la configurer pour être en mesure d'accéder aux fichiers dans `/home/yunohost.backup/archives/` depuis un navigateur web.

Enfin, il est possible d'utiliser `scp` (un programme basé sur [`ssh`](/ssh)) pour copier des fichiers entre deux machines grâce à la ligne de commande. Ainsi, depuis une machine sous Linux, vous pouvez utiliser la commande suivante pour télécharger une archive :

```bash
scp admin@your.domain.tld:/home/yunohost.backup/archives/<nom_d'archive>.tar.gz ./
```

De façon similaire, vous pouvez téléverser une sauvegarde depuis une machine vers votre serveur avec :

```bash
scp /path/to/your/<nom_d'archive>.tar.gz admin@your.domain.tld:/home/yunohost.backup/archives/
```

Restaurer des sauvegardes
-------------------------

#### Depuis la webadmin

Allez dans Sauvegardes > Sauvegardes locales et sélectionnez l'archive. Vous pouvez ensuite choisir les différents éléments que vous voulez restaurer puis cliquer sur "Restaurer".

![](/images/restore.png)

#### Depuis la ligne de commande

Depuis la ligne de commande, vous pouvez utiliser `yunohost backup restore <nom_d'archive>` (sans le `.tar.gz`) pour restaurer une archive. Tout comme `yunohost backup create`, cela restaure tout le contenu par défaut. Si vous souhaitez restaurer seulement certaines parties, vous pouvez utiliser par exemple `yunohost backup restore --apps wordpress` qui restaurera seulement l'app wordpress.

#### Contraintes

Pour restaurer une application, le domaine sur laquelle elle est installée doit déjà être configuré (ou il vous faut restaurer en même temps la configuration correspondante). Aussi, il n'est pas possible de restaurer une application déjà installée... ce qui veut dire que pour restaurer une sauvegarde d'une app, il vous faut déjà la désinstaller.

#### Restauration d'une archive à la place de la post-installation

Une fonctionnalité particulière est la possibilité de restaurer une archive entière *à la place* de faire la post-installation. Ceci est utile pour réinstaller un système entièrement à partir d'une sauvegarde existante. Pour faire cela, il vous faudra d'abord téléverser l'archive sur le serveur et la placer dans `/home/yunohost.backup/archives`.

Ensuite, **à la place de** `yunohost tools postinstall`, réalisez la restauration de l'archive téléversée par cette ligne de commande avec le nom de l'archive (sans le `.tar.gz`) :

```bash
yunohost backup restore <nom_d'archive>
```

Note: si votre archive n'est pas dans `/home/yunohost.backup/archives`, vous pouvez spécifier où elle se trouve comme ceci :

```bash
yunohost backup restore /path/to/<archivename>
``` 



Pour aller plus loin
--------------------

#### Stocker les archives sur un autre disque

Si vous le souhaitez, vous pouvez connecter un disque externe à votre serveur pour (parmi d'autres choses) stocker les archives de backup dessus. Pour cela, il faut d'abord déplacer les archives existantes vers le disque, puis créer un lien symbolique: 

```bash
PATH_TO_DRIVE="/media/mon_disque_externe" # Par exemple - Tout dépend d'où le disque est monté
mv /home/yunohost.backup/archives $PATH_TO_DRIVE/yunohost_backup_archives
ln -s $PATH_TO_DRIVE/yunohost_backup_archives /home/yunohost.backup/archives
```

#### Sauvegardes automatiques

Vous pouvez ajouter une tâche cron pour déclencher automatiquement une sauvegarde régulièrement. Par exemple pour sauvegarder l'application wordpress toutes les semaines, créez un fichier `/etc/cron.weekly/backup-wordpress` avec le contenu suivant :

```bash
#!/bin/bash
yunohost backup create --apps wordpress
```
puis rendez-le exécutable :

```bash
chown +x /etc/cron.weekly/backup-wordpress
```

Soyez prudent à propos de ce que vous sauvegardez et de la fréquence : il vaut mieux éviter de se retrouver avec un disque saturé car vous avez voulu sauvegarder 30 Go de données tous les jours...

#### Sauvegarder sur un serveur distant

Vous pouvez suivre ce tutoriel sur le forum pour mettre en place Borg entre deux serveurs : https://forum.yunohost.org/t/how-to-backup-your-yunohost-server-on-another-server/3153

Il existe aussi l'application Archivist qui permet un système similaire : https://forum.yunohost.org/t/new-app-archivist/3747

#### Backup complet avec `dd`

Si vous êtes sur une carte ARM, une autre méthode pour créer une sauvegarde complète consiste à créer une image (copie) de la carte SD. Pour cela, éteignez votre serveur, insérez la carte SD dans votre ordinateur et créez une image avec une commande comme :

```bash
dd if=/dev/mmcblk0 of=./backup.img status=progress
```

(remplacez `/dev/mmcblk0` par le vrai nom de votre carte SD)
