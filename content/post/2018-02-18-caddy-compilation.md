+++
draft = false
title = "Compilation du server HTTP Caddy"
date = 2018-02-18T17:03:38+01:00
slug = "caddy-server-compilation"
tags = ["golang","caddy"]
categories = ["blog"]
thumbnail = "images/2018/02/18/caddy-server.png"
toc = true # Optional
+++


Le but de ce billet n'est pas de vous faire l'éloge mérité du [serveur HTTP Caddy](https://caddyserver.com/), mais juste de vous expliquer comment le compiler.

Pour la petite histoire [Matthew Holt](https://twitter.com/mholt6) l'auteur/créateur de Caddy à dernièrement changé le mode de distribution et la licence du binaire pour essayer de gagner quelques sousous avec sa création. Et vous savez quoi ?

 **Il a bien raison !!** 

 <div style="width:100%;height:0;padding-bottom:53%;position:relative;"><iframe src="https://giphy.com/embed/9HQRIttS5C4Za" width="100%" height="100%" style="position:absolute" frameBorder="0" class="giphy-embed" allowFullScreen></iframe></div>
 
 C'est juste dommage qu'il soit obligé d'en arriver à imposer le paiement d'une licence. Je veux dire par là qu'il serait peut être temps que "les gens"© JLM2017, réalisent que l'open-source, si c'est bon pour le Karma, ça ne rempli pas l'assiette des gosses et qu'il n'est donc pas interdit de donner en retour.
 
Bref... revenons à Caddy et à sa compilation, car si la distribution et la licence du binaire ont changés, Caddy est toujours un projet open-source, que vous pouvez utiliser comme bon vous semble.

Dans ce tuto je vais donc vous expliquer comment compiler Caddy avec le plugin [IP Filter](https://github.com/pyed/ipfilter). Je vais le faire sur mon poste de travail sous [Ubuntu 16.04](https://www.ubuntu.com/)


## Installation de Go

Caddy est codé en [Go](https://golang.org/), la première chose à faire c'est donc d'installer Go, vous pouvez le faire via votre gestionnaire de packages (apt, yum, brew,...), ou en utilisant les installateurs fournis sur la page de [téléchargement du site](https://golang.org/dl/) où pour les plus poilu d'entre vous, [depuis les sources](https://golang.org/doc/install/source).

Dites, maintenant que vous avez Go installé, ce serait ballot de passez pas à coté de l'opportunité d'apprendre un nouveau langage... ;)

## Récupération du code source de Caddy

Vous allez avoir besoin de *git*, si vous ne l'avez pas sur votre machine, c'est le moment de l'installer.

Puis on "go get" les sources de Caddy:

```
go get github.com/mholt/caddy/caddy
```

## Récupération du code source du plugin

Dans mon cas je n'ai qu'un plugin, si vous voulez en ajouter plusieurs il vous suffit de répéter cette étape.

```
go get github.com/pyed/ipfilter
```

## Modification des sources de Caddy pour activer le plugin

Pour activer un plugin, il suffit de l'importer dans Caddy. Ca se passe dans le fichier ```$GOPATH/src/github.com/mholt/caddy/blob/master/caddy/caddymain/run.go```

Donc on édite le fichier pour y coller notre/nos import(s) de plugin(s)

```
nano $GOPATH/src/github.com/mholt/caddy/caddy/caddymain/run.go
```

Vous verrez à la fin des la section d'import la ligne ```// This is where other plugins get plugged in (imported)``` et bien figurez vous que l'on va mettre nos directives d'import de plugins sous cette ligne. C'est fooouuuuu non ?!

Dans mon cas ça ressemble donc à:

```go
_ "github.com/mholt/caddy/caddyhttp"

"github.com/mholt/caddy/caddytls"
// This is where other plugins get plugged in (imported)
_ "github.com/pyed/ipfilter"
```

## Compilation de Caddy

Attention, accrochez vous on arrive au point le plus délicat de l'opération. Vous êtes prêt ? Vos 
chakras sont bien ouverts ? On respire un grand coup et...c'est parti !

```
cd $GOPATH/src/github.com/mholt/caddy/caddy 
go build -o /tmp/caddy
```

Et voila c'est tout, votre binaire est compilé et est disponible ici: ```/tmp/caddy```

Je rappelle que si Caddy doit tourner sous un autre OS et/ou système que la machine qui vous sert à le compiler, il vous suffit d'indiquer la cible lors du ```go build```. Par exemple si vous voulez faire tourner Caddy sur votre Raspberry Pi, il vous suffit de de le compiler avec les options suivantes:

```
env GOOS=linux GOARCH=arm go build -o /tmp/caddy
```
Voila c'est tout pour aujourd'hui.

A plus les puces, bisous les loulous.


