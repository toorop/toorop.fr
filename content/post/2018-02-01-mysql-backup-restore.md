---
title: "Gox: sauvegardes et restaurations MySQL sans peine."
date: 2018-02-01T18:27:33+01:00
draft: false
slug: "backup-mysql-gox"
tags: ["mysql", "sysadmin"]
categories: ["devlogs", "tuto"]
thumbnail: "images/2018/02/01/mysql-backup.png"
toc: true # Optional
---

Le besoin de base était d'avoir un outil **simple**, à exécuter un sur serveur distant, permettant de:

- sauvegarder à chaud un serveur MySQL (et/ou MariaDB et/ou Percona). Autrement dit avoir un backup consistant sans avoir à couper MySQL ou à verrouiller les tables.
- transférer ce backup sur un serveur de stockage.
- gérer la durée de rétention de ces backups.
- en cas de coup dur, pouvoir restaurer un backup sans stress.

J'ai commencé a chercher, et puis... j'ai fini par en coder un en [Go](https://golang.org/): [Gox](https://github.com/toorop/gox)

Je vais vous décrire dans ce billet comment mettre en place le service.

## Préparation des serveurs à sauvegarder
Gox se base sur [xtrabackup](https://www.percona.com/software/mysql-database/percona-xtrabackup) pour le gros du travail. Donc la première chose à faire va être de l'installer sur les serveurs MySQL à sauvegarder.

La procédure suivante est valable pour un serveur sous Debian (ou dérivé comme Ubuntu). Si vous utilisez une autre distribution consultez la [documentation officielle xtrabackup](https://www.percona.com/doc/percona-xtrabackup/LATEST/installation.html)

```
$  wget https://repo.percona.com/apt/percona-release_0.1-4.$(lsb_release -sc)_all.deb
$ sudo dpkg -i percona-release_0.1-4.$(lsb_release -sc)_all.deb
$ sudo apt-get update
$ sudo apt-get install percona-xtrabackup-24 qpress
```


## Préparation du serveur de sauvegarde

Téléchargez le binaire Gox correspondant à votre architecture depuis [la section Releases](https://github.com/toorop/gox/releases) du repo Github. Par exemple si vous êtes sous un systême 64 bits:

```
$ cd /usr/local/bin
$ sudo wget https://github.com/toorop/gox/releases/download/0.1.0/gox_0.1.0_linux-amd64
```

On va renommer le binaire pour ne garder que **gox**:
```
$sudo mv gox_0.1.0_linux-amd64 gox
```

Et on va le rendre exécutable:
```
$ sudo chmod +x gox
```

## Configuration d'une tache de sauvegarde / restauration

Pour chaque serveur MySQL à sauvegarder vous devrez créer un fichier de configuration.

Voici a quoi ce fichier ressemble:

```yaml
# Remote MySQL host
host: mysql.explample.com
# Mysql user
dbuser: root
# Mysql password
dbpassword:
# SSH config
ssh:
  # SSH user
  user: root
  # Private key for ssh user
  key: /home/jdoe/.ssh/id_rsa
# Remote path of xtrabsckup binary
xtrabackup: /usr/bin/xtrabackup
# The number of threads to use to copy multiple data files concurrently when creating a backup
parallel: 2
# Compression 
compress:
  # Compress ?
  active: true
  # This option specifies the number of worker threads used by xtrabackup for parallel data compression
  threads: 2
  # This option when specified will remove .qp, .xbcrypt and .qp.xbcrypt files after decryption and decompression.
  remove-original: true
# This options creates the xtrabackup_galera_info file which contains the local node state at the time of the backup. 
galera: false
# Storage path for backup
backup-dir: /var/backup/mysql/mysql.example.com
# Remove backups older than 'keep' 
# A duration string is a possibly signed sequence of decimal numbers, each with optional fraction and a unit suffix,
# such as "300ms", "-1.5h" or "2h45m". Valid time units are "ns", "us" (or "µs"), "ms", "s", "m", "h".
keep: 168h
```

Je pense que je n'ai pas besoin de m'étendre sur les différents paramètres.

**Attention** n'oubliez pas d'ajouter l'extension *.yalm* au nom de votre fichier sinon vous allez avoir droit à un beau:
```
root@sd-52473:~# gox backup --config /home/backup/gox_dc3-1
2018/02/01 22:32:19 Unsupported Config Type ""
```

## Sauvegarde

Pour lancer une sauvegarde il ne vous reste plus qu'a lancer la commande suivante:
```
gox backup --config /chemin/vers/le/fichier/de/config-serveur1.yaml
``` 

gox va se connecter en SSH au serveur MySQL distant, lancer la sauvegarde et "piper" le flux de data vers un fichier local. Cette méthode permet d'être très véloce, à titre indicatif sur un serveur MySQL avec environ 2GB de data, il me faut  moins d'une minute pour réaliser une sauvegarde. 

Bin voila c'est fini. 

Il ne vous reste plus qu'a définir une tache cron qui va se charger de faire cette sauvegarde à intervalles régulier, par exemple pour faire un backup MySQL toutes les nuits à 1 heure, ajouter à votre crontab:

```
# Backup MySQL
0 1 * * * root /usr/local/bin/gox backup --config /chemin/vers/le/fichier/de/config-serveur1.yaml

```


## Restauration

Dans le dossier "backup-dir" vous allez avoir des répertoires qui ont des noms représentant les dates des différents backups disponibles. Par exemple *2018-02-01--17-18-52* pour un backup fait, je vous le donne en mille, le 1 février 2018 a 17 heures 18 minutes et 52 secondes.

Pour réstaurer ce backup il nous suffit de lancer la commande:

```
gox restore --config /chemin/vers/le/fichier/de/config-serveur1.yaml --from 2018-02-01--17-18-52
```

A noter que si il s'agit d'un nœud d'un [cluster Galera](http://galeracluster.com/products/), **gox** va récupérer le position lors du backup et relancer mysql avec l'option ```--wsrep_start_position=position``` ce qui va permettre au noeud de se resynchroniser plus rapidement.



