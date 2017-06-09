+++
categories = ["tuto"]
date = "2017-06-07T10:37:11+02:00"
draft = false
slug = "tuto-backup-restic-object-storage-ovh"
tags = ["restic","openstack","golang","ovh"]
thumbnail = "images/1/restic.png"
title = "Réalisez vos backups avec Restic et l'object storage OVH"
toc = true

+++

Voici le fameux article qui à signé la mort de mon ancien blog, c'est en effet pendant que je le rédigeais que tout à crashé.  
J'espère que j'aurais plus de chances dans cette nouvelle tentative...

Aujourd'hui je vous propose une courte introduction à <a href="https://restic.github.io/Restic" target="_blank" alt="backup avec restic">Restic</a> et surtout à sa mise en oeuvre en utilisant le service <a href="https://www.ovh.com/fr/public-cloud/storage/object-storage/" target="_blank" alt="public object storage ovh swift">Object Storage d'OVH</a> comme "conteneur" pour vos sauvegardes. 
<!-- more -->

![restic](/images/1/restic.png#center)

Dans ce qui suit je vais utiliser ce que j'ai sous la main, autrement dit mon poste de travail sous Ubuntu. Pour autant Restic étant multiplateforme ça ne devrait pas trop différer si vous utilisez un autre OS. (d'autant plus qu'il m'a semblé entendre qu'il y avait maintenant un vrai shell sous windows !!).

# Preambule
Restic est compatible avec de nombreux sytémes de stockage, si j'ai choisi <a href="https://www.ovh.com/fr/public-cloud/storage/object-storage/" target="_blank" alt="public object storage ovh swift">l'Object Storage d'OVH</a> c'est essentiellement par que je l'utilise déja pour d'autres projets et que le coût est trés compétitif: 1 centime d'euro pas mois par Go stocké, plus les transferts (aux mêmes tarifs). 

En passant, j'en vois qui se jettent sur les offres de **stockage illimitées dans le cloud**, sans parler du fait que l'illimité n'est qu'un concept marketing pour vous transformer en clients, et une fois que c'est fait les limites apparaissent, posez vous simplement cette question:

* je paierai combien sur l'object storage OVH pour stocker ce que je stocke actuellement ?


# Quelles sont les spécificités de Restic ?

La principale, pour moi, réside dans son système de stockage. Beaucoup de système de backup vont "se contenter" de faire un copie de vos fichiers vers l'espace de stockage (avec des options comme du chiffrement, compression,... mais ça va rester de la copie de fichiers). Restic va plus loin, en utilisant un *repo* où il va stocker les data sous forme de "petits" paquets de bits. Du coup le moteur de stockage va pouvoir, entre autre, utiliser pleinement les systèmes déduplication pour économiser de la place.  

Hum je vois que j'en ais perdu... la déduplication c'est tout con, un mec c'est réveillé un matin en se disant: 

> Mais quel est l'intérêt d'enregistrer dans un espace de stockage deux fois les mêmes données ? Il suffirait de les enregistrer une seule fois et d'y faire référence quand c'est utilisé ailleurs.

Voila, la dedup c'est ça.

Accessoirement, le fait d'utiliser un "repo binaire" pour stocker les data, ça permet aussi quand on à beaucoup de fichiers, de ne pas avoir a faire autant de transferts que de fichiers (ceux qui utilisent Swift verront tout de suite où je veux en venir). A titre d'exemple la sauvegarde initiale d'un répertoire contenant près de 4000 fichiers pour un total d'un peu plus de 100Mo à pris 19 secondes.

Pour le reste Restic fait dans le classique mais il le fait bien, je pense en particulier aux plus paranos d'entre vous, je vous rassure donc, vos photos de vacances (vous savez celles où vous êtes en string au bord de la piscine avec une pinte de bière à la main), ne finirons pas sur le bureau de Trump. Les sauvegardes sont chiffrées.


# Creation du conteneur chez OVH

En tout premier lieu, il va vous falloir un compte chez OVH pour utiliser l'object storage. (il y a des gens qui n'en ont pas !? mais mais, comment cela est ce possible ?)  
Ensuite vous vous connectez sur votre [espace client OVH](https://www.ovh.com/manager/cloud/), vous vous rendez à la section *Cloud*, vous cliquez sur *commander*, puis *projet cloud*. 
Une fois que votre projet créé, rendez vous dans la section *stockage* et créez un nouveau container **privé** que l'on va nommer *restic*

![restic](/images/1/OVH-Cloud.png#center)

Côté OVH il va nous falloir ensuite le fichier *openrc* qui contient un petit script qui va mettre en variables d'environnements les info necessaires pour vous authentifier. Dans votre projet, cliquez sur le menu *Openstack*, puis *ajoutez un nouvel utilisateur*. Mettez le mot de passe au chaud, par exemple sur un Postit collé à votre moniteur, on va en avoir besoin plus tard. 
Cliquez sur la petite clé au bout de la ligne correspondant à votre utilisateur, puis sur *Télécharger un fichier de configuration Openstack*.  

Un petit tip crad pour éviter d'avoir à vous taper le mot de passe à chaque fois, éditez le script en remplaçant:

```
# With Keystone you pass the keystone password. 
echo "Please enter your OpenStack Password: "
read -sr OS_PASSWORD_INPUT
export OS_PASSWORD=$OS_PASSWORD_INPUT
```  

Par:
```
# With Keystone you pass the keystone password. 
export OS_PASSWORD="VOTRE MOT DE PASSE"
```  

Pour mettre les parametres de ce fichier en variabled d'environnement exécutez:
```
source openrc.sh
```
ou en fonction des OS:
```
. openrc.sh
```

Bien à présent le nécessaire pour s'authentifier auprès de l'Object Storage OVH est en place.


# Installation de Restic
Je ne vais pas paraphraser la [documentation de Restic](http://restic.readthedocs.io/en/latest/installation.html), je vous invite donc à la lire pour installer la bestiole.  

Sachez quand même que:

* A la date ou j'écris cet article, le protocole Swift n'est supporté que dans la version de dev.
* Comme Restic est codé en <a href="https://golang.org/" target="_blank" alt="go programming language">Go</a> il y a donc des binaires pour tout le monde, quelque soit votre OS (ou presque).

# Initialisation du "repo" Restic
Je rappelle que l'on va utiliser le conteneur nommé *restic* que l'on à crée.  
La commande pour initialiser le repo est:

```
restic -r swift:restic:/ init
```

**ATTENTION:** un mot de passe va vous être demandé, il sert à chiffrer vos données. Ne le perdez pas sinon c'est fini, vous pourrez dire adieu à vos backups.

# Création d'un nouveau snapshot
Je ne vous l'ai pas dit mais Restic va créer des snapshots, donc vous pourrez conserver autant d'état de vos données que vous voulez correspondant à différentes dates. Du fait de son système de dedup, seuls les données qui ont été modifiées vont être stockées lorsque vous ferrez un nouveau snapshot.

Et bien dans mon cas figurez vous que je vais faire une sauvegarde de ce blog. Son chemin sur mon poste de travail est:

```
/home/toorop/Projects/www/toorop.fr
```

Pour réaliser le snapshot vers l'espace de stockage distant, il me suffit donc de lancer la commande:

```
restic -r swift:restic:/ backup /home/toorop/Projects/www/toorop.fr
```

Voila ce que ça donne:

```
time restic -r swift:restic:/ backup /home/toorop/Projects/www/toorop.fr
enter password for repository: 
scan [/home/toorop/Projects/www/toorop.fr]
scanned 263 directories, 383 files in 0:00
[0:05] 100.00%  3.254 MiB/s  16.271 MiB / 16.271 MiB  646 / 646 items  0 errors  ETA 0:00 
duration: 0:05, 2.94MiB/s
snapshot 2f9fbf01 saved

real	0m15.222s
user	0m1.160s
sys	0m0.068s
```

J'ai ajouté, la commande *time*, elle nous permet de voir combien de temps ça a pris, ici 15 secondes pour 646 dossiers ou fichiers et un total de 16 MiB. 

Ah mais je viens de faire des modifications, vite, vite, il faut que je fasse un nouveau snapshot !!!

Pour vérifier les snapshots:

```
restic -r swift:restic:/ snapshots
enter password for repository: 
ID        Date                 Host        Tags        Directory
----------------------------------------------------------------------
2f9fbf01  2017-06-07 19:37:07  trooper                 /home/toorop/Projects/www/toorop.fr
bbe74d2a  2017-06-07 19:40:07  trooper                 /home/toorop/Projects/www/toorop.fr
```

Pour supprimer un snapshot on utilise son ID, par exemple dans mon cas si je veux supprimer le premier:
```
restic -r swift:restic:/ forget 2f9fbf01
restic -r swift:restic:/ snapshots
ID        Date                 Host        Tags        Directory
----------------------------------------------------------------------
bbe74d2a  2017-06-07 19:40:07  trooper                 /home/toorop/Projects/www/toorop.fr
```

Attention cela va uniquement supprimer la référence aux data, pour supprimer les data, après avoir supprimé un ou plusieurs snapshots, faites:
```
restic -r swift:restic:/ prune
```
C'est la que le bas blesse un peu en utilisant swift comme backend pour le repo, vous constaterez que c'est un peu plus lent que les autres commande. C'est normal Restic doit analyser ce qui est en local, puis ce qui est en remote et faire le dedup.

Pour restaurer un backup:
```
restic -r swift:restic:/ restore bbe74d2a --target ~/tmp/blog-restored
```

Vous pouvez bien entendu ne restaurer qu'un fichier en ajoutant le paramètre --path

Un petite derniére pour la route: vous pouvez monter le repo distant:

```
mkdir -p ~/mnt/restic
restic -r swift:restic:/ mount ~/mnt/restic
```
Abracadabra !

![restic](/images/1/restic-mount.png#center)

Et le pire, c'est ce que vous ne voyez pas, la navigation est fluide !

Pour connaitre toutes les possibilités de la bête, je vous invite à consulter la [documentation de Restic](http://restic.readthedocs.io/en/latest/index.html)


Hope this helps ;)


