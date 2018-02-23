+++
draft = false
title = "Découverte du service OVH Functions"
date = 2018-02-22T17:28:46+01:00
slug = "ovh-functions-golang"
tags = ["ovh", "golang", "faas"]
categories = ["devlogs"]
thumbnail = "images/2018/02/22/ovh-functions-golang.png"
toc = true # Optional
+++

Dans ce billet on va découvrir le service FaaS (Functions As A Service) d'OVH: [OVH Functions](https://labs.ovh.com/ovh-functions) en créant un service de création de miniature d'images en [Go](https://golang.org/).

![What ??](https://media.giphy.com/media/fd1TSJqq3b4GI/giphy.gif#center)

## Function As A Service ???

Oubliez tous les buzzwords, le FaaS c'est tout bête, ça consiste à faire héberger une fonction chez un prestataire et ne plus se soucier de rien. Voila c'est tout.

Je suis certain que parmi vous il y a de nombreux développeurs qui ont du se mettre à l'administration de serveurs pour faire tourner les services que vous avez créé. Et au final si vous faisiez ce boulot sérieusement et consciencieusement ça vous prenait un temps fou. Temps qu'il aurait été bien plus rentable d'investir dans votre métier de base, le développement.

Le FaaS répond à cette problématique en vous permettant de faire une complète abstraction de l'infra (d'où le terme **serverless** que vous allez souvent croiser dans les mois qui viennent). Une faille majeure vient d'être découverte ? Restez couché les admins de FaaS s'en occupent. Jean-Piere Pernaut à parlé de votre service au journal de 13 heures, le trafic fait un  X100 !!! Ce pas un problème, pour vous, ça va - devrait - scaler automatiquement.


## Au menu: un service de redimensionnement d'images

Allez fini le blabla on va passer à la pratique en créant un service de création de thumbnails.

Le principe va être simple: on transmet l'URL d'une image au service, et la fonction va nous retourner une miniature de cette image.


## De quoi allons nous avoir besoin

### D'un token d'accès au service 

Actuellement le FaaS d'OVH est en phase de test, il faudra vous rendre sur la page [OVH Labs](https://labs.ovh.com/ovh-functions) et faire une demande de token.


### Du binaire ovh-functions

Vous pourrez le télécharger ici [get.functions.ovh](https://get.functions.ovh/)

## On passe au code

Le service OVH Functions permet de programmer ses fonctions dans plusieurs langage, ici, ce ne sera un surprise pour personne je vais le faire en Go.

### On commence par initialiser notre fonction

```
ovh-functions init go
```

Cette initialisation va créer deux fichiers:
- functions.yml qui va nous permettre de définir et configurer nos fonctions.
- example.go qui est...un exemple de function qui va nous retourner un "hello world"

Voyons à quoi ressemble ce dernier fichier:

```go

package main

import (
        "fmt"

        "github.com/ovhlabs/functions/go-sdk/event"
)

func Hello(event event.Event) (string, error) {
        fmt.Println(event)

        name := event.Data
        if name == "" {
                name = "World"
        }

        return "Hello " + name + " from OVH Functions!", nil
}
```

Regardons de plus prés cette fonction, elle reçoit une structure instanciée de type event.Event et retourne une string et une erreur.

Hum on voit un premier problème potentiel, le type *string* retourné, car ce que l'on veut retourner c'est une image autrement dit un slice de bytes. Aller on ne se démoralise pas sachant que au final ce qui va être retourné par le serveur quel que soit le type réel sera un slice de bytes. On va donc essayer en "castant" notre slice de bytes représentant notre image en string ça devrait passer. Il restera le problème du content type que l'on ne peut fixer... a moins que le service le fixe automatiquement en fonction de ce qu'on lui retourne. On verra. (j'ai oublié de préciser que j'écris ce billet au fur et à mesure que je teste/développe).

Penchons nous à présent sur le type **Event**, la source est disponible [ici](https://github.com/ovhlabs/functions/blob/master/go-sdk/event/event.go) :

```go
type Event struct {
	Data    string
	Method  string
	Params  map[string]string
	Secrets map[string]string
}
```

Je suppose que:
- Data correspond au body de la requête
- Method la méthode HTTP (GET, POST, PUT,...)
- Params les paramètres passés dans l'URL
- Secrets... je ne sais pas.

Penchons nous à présent sur le fichier **functions.yml** qui défini nos fonctions:

```
functions:                      <- Début des definitions de nos functions
  hello:                        <- Nom de notre premiére fonctuion
    runtime: go                 <- Le runtime, ici Go
    handler: example.Hello      <- Le handler qui va exécuter la requète
```


### Le code complet de notre fonction

La chose la plus "délicate" va être de redimensionner notre image.
Il n'y a pas, dans la librairie standard Go, de quoi le faire directement, on va donc utiliser le package [gift](https://github.com/disintegration/gift) et là une autre question se pose:

Est ce que l'on peut importer des packages qui ne font pas parti de la librairie standard ?

Ce que je vous propose c'est de coder la fonction et ensuite je vous l'explique. A tout de suite....


.... alors je peux répondre à la question précédente, oui on peut importer des packages. Heureusement d'ailleurs.

Voici le code de notre fonction avec le commentaires pour vous permettre de suivre le déroulement:

```go
package main

import (
	"bytes"
	"errors"
	"fmt"
	"image"
	_ "image/gif"
	_ "image/jpeg"
	"image/png"
	"io/ioutil"
	"net/http"
	"strings"

	"github.com/ovhlabs/functions/go-sdk/event"

	"github.com/disintegration/gift"
)

// Thumb est une "OVH function" qui va retourner une miniature
// de l'image qui lui est transmise spar son URL
// Formats d'entré: jpeg, png ou gif
// Format sortie: png de 100 pixels de large
func Thumb(event event.Event) (string, error) {
	// debug
	fmt.Println(event)

	// on récupère l'URL de l'image à traiter passer en paramétre
	// que l'on devrait donc trouver dans event.Params
	picURL, ok := event.Params["pic"]

	// si ce parametre est manquant ou si c'est une chaine vide
	// on retourne une erreur
	if !ok || strings.TrimSpace(picURL) == "" {
		return "", errors.New("parameter pic is missing")
	}

	// on pourrait vérifier que le paramètre picURL est bien une URL
	// mais le http.Get du dessous va le faire et générera un erreur
	// si ce n'est pas le cas

	// on va chercher l'image
	resp, err := http.Get(picURL)
	if err != nil {
		return "", err
	}
	defer resp.Body.Close()

	// on instancie une nouvelle stucture Image
	imgIn, _, err := image.Decode(resp.Body)
	if err != nil {
		return "", err
	}

	// on instancie gift et on lui passe les modifications à faire
	g := gift.New(
		// on redimensione l'image pour que la largeur soit de 100px
		// en utilisant l'algorithme Lanczos
		gift.Resize(100, 0, gift.LanczosResampling),
	)

	// on crée un nouvelle image qui recevra l'image redimensionée
	imgOut := image.NewRGBA(g.Bounds(imgIn.Bounds()))

	// Draw va prendre l'image de départ (img), va appliquer les filtres
	// et copier le résultat dans
	g.Draw(imgOut, imgIn)

	// on encode l'image en png dans un buffer
	imgOutByte := bytes.NewBuffer([]byte{})
	err = png.Encode(imgOutByte, imgOut)
	if err != nil {
		return "", err
	}

	// on recupere le contenu du buffer dans un variable
	out, err := ioutil.ReadAll(imgOutByte)
	if err != nil {
		return "", err
	}

	// on retourne notre slice de bytes sous forme d'une string
	// puisque c'est ce qu'attend le gestionnaire
	return string(out), nil
}
```

Il est aussi disponible sur [github](https://github.com/toorop/ovh-functions-tuto)

## On déploie la fonction

Pour déployer la fonction sur le FaaS OVH on utilise le CLI **ovh-function**.

A l'issu du déploiement, si il s'est bien passé, nous recevrons l'URL à utiliser pour appeler notre fonction:

```
$ ovh-functions deploy
Deploying function...
Updated function thumbnail. Uploaded 28 files for a total of 23 KiB.
To call your function, you can use the cli or the command:
        curl -XPOST -s 'https://exec.functions.ovh/f/nzteaaarcw42c/thumbnail?token=HzmH0HoI5X1kn6enFzBp'
```        

## Le moment est venu de tester

Pour éviter la prison, je vais utiliser une photo dont je dispose de tous les droits (et entre autres ceux sur les deux affreux qui sont dessus):

![golden hour](https://depierrepont.photo/images/2017-09-23/1.jpg#center)


Cette photo est disponible à l'adresse ```https://depierrepont.photo/images/2017-09-23/1.jpg``` ou va donc appeler notre fonction en utilisant cette url comme valeur pour le paramètre pic: 

```https://exec.functions.ovh/f/nzteaaarcw42c/thumbnail?token=HzmH0HoI5X1kn6enFzBp&pic=https://depierrepont.photo/images/2017-09-23/1.jpg ``` 

Et voila le résultat:

![Golden hour miniature](/images/2018/02/22/thumbnail.png#center)

Si vous voulez tester par vous même <a href="https://exec.functions.ovh/f/nzteaaarcw42c/thumbnail?token=HzmH0HoI5X1kn6enFzBp&pic=https://depierrepont.photo/images/2017-09-23/1.jpg" target="_blank">cliquez ici</a>

N'hésitez pas à essayer avec d'autres URL d'images.

Il est interessant de regarder les logs de notre fonction (accessibles via la commande ```ovh-functions logs -f ```):

```
[2018/02/23 13:24:26] Function thumbnail updated
[2018/02/23 13:24:37] Executing function thumbnail
[2018/02/23 13:24:42] [  thumbnail ] { GET map[pic:https://depierrepont.photo/images/2017-09-23/1.jpg] map[]}
[2018/02/23 13:24:42] |200| Function thumbnail executed in 601.02ms
[2018/02/23 13:24:44] Executing function thumbnail
[2018/02/23 13:24:44] [  thumbnail ] { GET map[pic:https://depierrepont.photo/images/2017-09-23/1.jpg] map[]}
[2018/02/23 13:24:44] |200| Function thumbnail executed in 201.01ms
[2018/02/23 13:24:46] Executing function thumbnail
[2018/02/23 13:24:46] [  thumbnail ] { GET map[pic:https://depierrepont.photo/images/2017-09-23/1.jpg] map[]}
[2018/02/23 13:24:46] |200| Function thumbnail executed in 199.97ms
[2018/02/23 13:24:47] Executing function thumbnail
[2018/02/23 13:24:47] [  thumbnail ] { GET map[pic:https://depierrepont.photo/images/2017-09-23/1.jpg] map[]}
[2018/02/23 13:24:47] |200| Function thumbnail executed in 233.74ms
[2018/02/23 13:24:48] Executing function thumbnail
[2018/02/23 13:24:48] [  thumbnail ] { GET map[pic:https://depierrepont.photo/images/2017-09-23/1.jpg] map[]}
[2018/02/23 13:24:48] |200| Function thumbnail executed in 207.38ms
```

Qu'est ce qui saute aux yeux ? La diference dans le temps d'execution entre le premier appel et le suivant. On peut supposer que tant que la fonction n'est pas appélée elle est "mise en veille" puis "reveillée" au premier appel. D'ailleurs si on n'utilise pas la fonction pendant un certain delais on constatera le même phénoméne.

## Conclusion
Je ne vais pas donner mon avis sur le service OVH mais sur les **Functions As A Service**.

Les points positifs je les ai décris plus haut, si je devais ne retenir qu'un point négatif ce serait les limitations de notre champ d'action sur le processus HTTP. Ce que je veux dire c'est que j'aurais aimé avoir accées à la requète HTTP et à la reponse HTTP dans ma fonction. En gros que la signature soit celle d'un handler HTTP complet, par exemple en Go:

``` func faas(w http.ResponseWriter, r *http.Request) error ```

Ca nous aurait permis de gérer beaucoup plus de choses je pense en pariculier à la gestions de headers, des cookies, du retour (stream par exemple). On aurait aussi pu chainer des middlewares,...

Ca arrivera peut être un jour sur **HTTPaaS.com** ;) 

N'hésitez à me faire part de vos commentaires, et si vous avez envie de papoter il y a toujours du monde sur notre [serveur Discord](https://discord.toorop.fr)