+++
draft = false
date = "2018-01-19T10:07:53+02:00"
title = "Initiation WebAssembly avec Rust"
slug = "rust-webassembly"
tags = ["rustlang", "webassembly"]
categories = ["blog"]
thumbnail = "images/2018-01-19/rust-webassembly.jpg"
toc = true # Optional
+++
Toutes les années j'essaye de m'initier à un nouveau langage, pour 2018 ce sera [Rust](https://www.rust-lang.org/fr-FR/). Ce n'est pas la première fois que je m'y frotte mais je dois avouer que jusqu'alors je n'avais pas vraiment insisté. La raison principale est que *pour mes besoins* je ne voyais pas ce que je pouvais faire avec Rust que je ne pouvais déjà faire avec [Go](https://golang.org/). Or pour apprendre un langage, il faut bien entendu lire la doc pour avoir les bases, mais il faut surtout pratiquer. N'ayant pas d'idée de dev, mes précédentes tentatives ont vite manquées de motivation.

Bref, cette année je m'y suis remis et comme j'ai bien envie de tester un truc que je ne peux pas (encore) faire avec Go, générer du webassembly, je vous propose qu'on le fasse ensemble.

# WebAssembly ? heu c'est quoi donc ???

Et bien demandons a Wikipedia:

> WebAssembly, ou wasm, est un langage de programmation binaire de bas niveau pour le développement d’applications dans les navigateurs web. Webassembly est standardisé dans le cadre du W3C.

Plus concrètement WebAssembly va nous permet de "faire tourner du code" dans un navigateur en ayant plus de performance qu'en utilisant du javascript (et accessoirement de façon plus sure). On voit tout de suite l'intérêt quand on va avoir besoin d'exécuter des taches assez lourdes (chiffrement par exemple). A noter que l'on peu mixer Javascript et WebAssembly, l'un peut causer à l'autre et réciproquement. Ainsi on peut convertir nos fonctions gourmandes en WebAssembly et les appeler depuis le code javascript éxistant.


Vu que le but de ce post est avant tout de mettre la chose en pratique, je vous renvoie vers le [site officiel](http://webassembly.org/) pour plus d'info.


# Heu... comment on produit du WebAssembly ?

Et bien en codant dans certains langages et en compilant en WebAssembly. A l'heure actuelle et pour ce que j'en sais, on peut coder en C, C++ et.... Rust. On va utiliser ce dernier pour notre petite initiation.


# Coool ! on y va ?

Euh... oui, mais où ?

On ne va pas partir sur des trucs trop complexes, le but ici c'est plus de compiler du Rust en WebAssembly et de l'exécuter dans le navigateur, que d'écrire un client mail intégrant PGP.

On va donc prendre les exemples issus du *crate* [stdweb](https://github.com/koute/stdweb). 

Hum... je sens que je suis en train d'en perdre quelques uns.

Un *crate* dans l'univers Rust est une librairie/package.

Le crate *stdweb* à pour vocation d'assurer l'interopérabilité entre d'une part Rust et l'API Web (parcours et manipulation du DOM) et d'autre part Rust et Javascript. 

# Installation de Rust

Ce n'est pas l'étape la plus compliquée, puisqu'il suffit d'éxecuter la commande:

```
$ curl https://sh.rustup.rs -sSf | sh
```

et de suivre les instructions.

# Installation de la toolchain "nightly"
Comme beaucoup de projets, Rust à une branche stable et une branche de dev.
La possiblité de compiler en WebAssembly n'est pour le moment disponible que sur la branche de dev appelée "nightly"

```
$ rustup install nightly
```

Et pour l'activer en tant que toolchain par defaut:

```
$ rustup default nightly
```

# Installation de la "cible WebAssembly"
On peut compiler du code Rust pour différentes plateformes/OS, pour compiler en WebAssembly il nous faut installer la *target* wasm32-unknown-unknown :

```
$ rustup target add wasm32-unknown-unknown
```

# Installation de cargo web

cargo est l'outil qui permet de gérer les projets Rust, on va lui ajouter quelques commandes utiles pour le développement web coté client par le biais de [cargo-web](https://github.com/koute/cargo-web).

```
$ cargo install -f cargo-web
```

# Passons aux choses sérieuses

Comme dit quelque part plus haut, enfin je pense, le but de ce post n'est pas tant de coder que de compiler du code en WebAssembly et le faire tourner dans le navigateur. On va donc utiliser les exemples du projet stdweb.
La première chose à faire est donc de récupérer le code.

```
$ cd projects/rust/
$ git clone https://github.com/koute/stdweb
$ cd stdweb/examples/
```

# Exemple de base
Jetons un oeil dans le répertoire "minimal" qui contient un exemple basique.

A la racine vous trouverez un fichier *Cargo.toml* dont on ne va pas se préoccuper, il s'agit d'informations utiles pour Cargo.

Dans le répertoire *src*, nous avons un fichier *main.rs* qui contient:

```rust
#[macro_use]
extern crate stdweb;

fn main() {
    stdweb::initialize();

    let message = "Hello, 世界!";
    js! {
        alert( @{message} );
    }

    stdweb::event_loop();
}
```

Comme vous pouvez vous y attendre c'est du Rust, et il est assez simple, même si on ne connait pas ce langage, de deviner ce que ce code fait: il va ouvrir un boite d'alerte qui va afficher *Hello, 世界!*

L'équivalent Javascript serait:

```js
alert("Hello, 世界!");
```

Bon c'est bien beau tout ça mais si on testait ?

Pour compiler ce code en Webassembly et packager le necessaire pour que ce soit exploitable on utilise *cargo* avec la commande suivante:

```
$ cargo web start --target-webasm
warning: debug builds on the wasm32-unknown-unknown are currently totally broken
         forcing a release build
   Compiling itoa v0.3.4
   Compiling serde v1.0.27
   Compiling num-traits v0.1.41
   Compiling dtoa v0.4.2
   Compiling stdweb v0.3.0 (file:///home/toorop/Projects/rust/stdweb)
   Compiling serde_json v1.0.9
   Compiling minimal v0.1.0 (file:///home/toorop/Projects/rust/stdweb/examples/minimal)
    Finished release [optimized] target(s) in 20.39 secs
    Garbage collecting "minimal.wasm"...
    Processing "minimal.wasm"...
    Finished processing of "minimal.wasm"!

If you need to serve any extra files put them in the 'static' directory
in the root of your crate; they will be served alongside your application.
You can also put a 'static' directory in your 'src' directory.

Your application is being served at '/js/app.js'. It will be automatically
rebuilt if you make any changes in your code.

You can access the web server at `http://127.0.0.1:8000`.
```

Rendez vous à présent à l'adresse http://127.0.0.1:8000 et .... tadin !!!

![webassembly rust](/images/2018-01-19/capture-1.png#center)


Si vous ouvrez les devtools vous verrez les différents fichiers et entre autres le fichier *minimal.wasm* issu du code Rust qui a été compilé en WebAssembly.

Voila vous avez fait votre premier site utilisant WebAssembly, vite allez l'ajouter à votre CV dans quelques mois ça vaudra de l'or ;)

Si vous voulez allez plus loin, jetez un coup d'oeil au autres exemples. Pour les "lancer" c'est la même procédure que pour l'exemple de base.

Et si vous voulez vraiement allez plus loin et en particulier vous mettre à Rust je vous recommande l'excellent [Rust Book second edition](https://doc.rust-lang.org/book/second-edition/) comme point d'entré.

A++
