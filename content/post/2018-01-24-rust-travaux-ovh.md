---
title: "Mon premier programme Rust !! Un bot Discord"
date: 2018-01-24T10:01:03+01:00
draft:  false
slug: "rust-discord-bot-ovh"
tags: ["rustlang", "ovh", "discord", "ovh"]
categories: ["devlogs", "tuto"]
thumbnail: "images/2018/01/24/bot-discord-rust.png"
toc: true # Optional
---

Si vous suivez mes petites aventures, vous n'êtes pas sans savoir que j'ai décidé de m'initier à [Rust](https://www.rust-lang.org/fr-FR/). Comme je le disais dans mon [précédent post](/post/rust-webassembly/) quand on apprend un nouveau langage, lire les docs c'est bien mais il faut pratiquer, c'est indispensable pour être confronté au langage. Parmi les techniques d'apprentissage il y en a une autre qui fonctionne très bien, celle qui consiste à transmettre ce que l'on a appris. Si on y arrive et que ceux à qui on l'explique ont compris c'est que nous même on à compris. Poil au kiki !!

Je vais essayer de mettre tout ça en pratique dans ce billet. Let's go !

## Ce que l'on souhaite obtenir

Alors pour ma première application Rust, je vais faire quelque chose qui, a priori ne devrait pas être trop compliqué (j'ai oublié de préciser que je rédigeais ce billet en parallèle de mes expérimentations...): un CLI, autrement dit une application console qui va surveiller la page [travaux d'OVH](http://travaux.ovh.net/) et afficher sur la console les événements. 

Cette page fournis un flux RSS et c'est par ce biais que l'on va récupérer les informations.

Donc en gros ça va fonctionner de la façon suivante:

* on lance l'application
* elle va lire le flux RSS .
* elle va boucler sur:
    * pause de X secondes (x va dépendre d'un éventuel cache et/ou de la périodicité de mise à jour du flux).
    * lecture du flux
    * si il y a de nouveaux articles, elle les affiche. 

> **Update:** ça c'était l'idée de départ, au final ce sera un bot Discord qui va afficher dans un canal spécifique les news de la page travaux@ovh. Puisqu'on en parle n'hésitez pas réjoindre notre petite - mais de qualité - communauté sur [Discord](https://discord.gg/0eUtmzEdD3A06LmH). Retournons dans le passé pour suivre la suite des mes aventures.


## De quoi avons nous besoin ?

- Un client HTTP pour récupérer le flux RSS.
- Une librairie permettant d'extraire les données du flux RSS.
- Un librairie pour mettre en forme la sortie console.


## On pose les fondations

En route, on commence par créer un nouveau projet en utilisant *cargo*, on va l'appeler **ovht**:

```
$ cargo init --bin ovht
```

*cargo* est un outil Rust qui va permettre de faciliter la gestion du projet, la sous-commande *init* va nous permettre d'initialiser un nouveau projet Rust, l'option --bin indique que ce sera un exécutable (et pas une librairie), et enfin on indique le nom du projet.

Cargo va générer une hiérarchie de fichiers, commune à tout projet Rust, voila ce que ça donne juste après l'initialisation:

```
$ tree .
.
├── Cargo.toml
└── src
    └── main.rs
```

Le fichier Cargo.toml va nous permettre de définir les différentes caractéristiques du projet, voyons ce qu'il contient:

```
$ cat Cargo.toml
[package]
name = "ovht"
version = "0.1.0"
authors = ["Stéphane Depierrepont aka Toorop <toorop@toorop.fr>"]

[dependencies]
```
Par défaut Cargo va renseigner, le nom du projet, la version et l'auteur dans la section ```[package]```.

Ensuite il y à la section ```[dependencies]``` qui, va permettre de définir les dépendances nécessaires à notre projet.

Avant d'y ajouter notre première dépendance, penchons nous sur le fichier *main.rs*:

```rust
$ cat src/main.rs
fn main() {
    println!("Hello, world!");
}
```

On voit que par défaut Cargo à ajouté du code, celui du plus célèbre des programmes le fameux **Hello World**. Allez soyons fous exécutons le:

```
$ cargo run
   Compiling ovht v0.1.0 (file:///home/toorop/Projects/rust/ovht)
    Finished dev [unoptimized + debuginfo] target(s) in 0.68 secs
     Running `target/debug/ovht`
Hello, world!
```

Épatant non ? ;)

Bon au moins ça à le mérite de nous rappeler comme exécuter notre projet en une seule commande (ie sans passer par build + exec).

On remarquera aussi que l'arborescence du projet à changée:

```
$ tree .
.
├── Cargo.lock
├── Cargo.toml
├── src
│   └── main.rs
└── target
    └── debug
        ├── build
        ├── deps
        │   └── ovht-a2ed38c6a31eadf8
        ├── examples
        ├── incremental
        ├── native
        ├── ovht
        └── ovht.d
```

Bien ajoutons notre première dépendance au fichier *Cargo.toml*.
Il va s'agir de [Clap](https://crates.io/crates/clap), une librairie qui facilite la création de CLI et en particulier qui va nous permettre de récupérer les options passées en ligne de commande.


```
[package]
name = "ovht"
version = "0.1.0"
authors = ["Stéphane Depierrepont aka Toorop <toorop@toorop.fr>"]

[dependencies]
clap = "2.29"
```

Sans rien changer d'autre essayons d'exécuter de nouveau notre programme:

```
$ cargo run
    Updating registry `https://github.com/rust-lang/crates.io-index`
   Compiling bitflags v1.0.1
   Compiling libc v0.2.36
   Compiling unicode-width v0.1.4
   Compiling ansi_term v0.10.2
   Compiling strsim v0.6.0
   Compiling vec_map v0.8.0
   Compiling textwrap v0.9.0
   Compiling atty v0.2.6
   Compiling clap v2.29.2
   Compiling ovht v0.1.0 (file:///home/toorop/Projects/rust/ovht)
    Finished dev [unoptimized + debuginfo] target(s) in 8.3 secs
     Running `target/debug/ovht`
Hello, world!
```

On voit que Cargo est allé chercher notre dépendance *clap* et les dépendances de cette dépendance, puis les a compilé.

Regardons de nouveau notre arborescence:

```
$ tree .
.
├── Cargo.lock
├── Cargo.toml
├── src
│   └── main.rs
└── target
    └── debug
        ├── build
        ├── deps
        │   ├── libansi_term-012690530882c051.rlib
        │   ├── libatty-209c85d0f86fc3ff.rlib
        │   ├── libbitflags-d9077c45affafc32.rlib
        │   ├── libclap-254b3a57492f88df.rlib
        │   ├── liblibc-b1ca85687f9f2272.rlib
        │   ├── libstrsim-5b26b7d204f15494.rlib
        │   ├── libtextwrap-94d7c2b82652f6c9.rlib
        │   ├── libunicode_width-915ad14b945324b2.rlib
        │   ├── libvec_map-a6e38ae82d25e4cd.rlib
        │   ├── ovht-6ddf473b72ce520f
        │   └── ovht-a2ed38c6a31eadf8
        ├── examples
        ├── incremental
        ├── native
        ├── ovht
        └── ovht.d
```

On voit que les différentes librairies sont à présent dans l'arborescence de notre projet.

Bien on va - enfin - commencer à coder pour obtenir une application qui ne fait rien d'autre que de lire les options passées en ligne de commande.... 30 minutes plus tard:


```rust
extern crate clap;
use clap::App;

fn main() {
    let matches = App::new("ovht")
        .version("0.1.0")
        .author("Stéphane Depierrepont")
        .about("Watch http://travaux.ovh.net feed")
        .args_from_usage(
            "-c, --count=[COUNT]   'Display last count entries (defaul=10)'
                          -s, --section=[SECTION] 'Section to watch (default=all)'",
        )
        .get_matches();

    let section = matches.value_of("section").unwrap_or("all");
    let display_count = matches.value_of("count").unwrap_or("10");

    println!("section: {} display_count: {}", section, display_count )
}
```

```
$ cargo run -- --section=5 --count=20
   Compiling ovht v0.1.0 (file:///home/toorop/Projects/rust/ovht)
    Finished dev [unoptimized + debuginfo] target(s) in 0.69 secs
     Running `target/debug/ovht --section=5 --count=20`
section: 5 display_count: 20
```

Hourra !!!! (et oui parfois je me réjouis de choses ~~qui semblent êtres~~ simples).

Je vais juste revenir sur le *unwrap_or*, le reste me semble assez clair. Donc à quoi sert ce *unwrap_or* ou *unwrap* que vous rencontrerez souvent en Rust ?

Imaginons que je ne passe pas de paramètre *section* lors de l'exécution, l'expression *matches.value_of("section")* va avoir quelques problèmes.
Cette expression retourne un 'objet' de type [Result](https://doc.rust-lang.org/std/result/enum.Result.html), un *enum* qui peut avoir deux types: *Ok(V)* si tout se passe bien, *Err(error)* dans le cas contraire. En fait ce *Result* va nous permettre de gérer correctement les cas où ça se passe mal. En l'occurrence ici, si le parseur ne trouve pas le paramètre *section*, autrement dit si la valeur retournée de type *Result* est Err, grace à la méthode *unwrap_or(val)*, l'expression va retourner *val*.

Qu'est ce qui m'a posé problème ?

Dans un premier temps je n'avais pas ajouté le ```[OPTION]``` de ```--option=[OPTION]``` et... ça marchait beaucoup moins bien forcement.

Qu'est ce qui me chiffonne ?

Je n'ai pas lu toute la [documentation du crate app](https://docs.rs/clap/2.29.2/clap/) (oui je sais c'est mal) mais à priori on ne peut pas définir de type pour les options. Autrement dit on ne récupère que des *string* qu'il faudra convertir plus tard dans le type approprié à l'usage.

## Et si on allait lire le feed

Bonne nouvelle ! Je suis tombé sur une librairie, un *crate*, [RSS](https://crates.io/crates/rss) qui à va nous permettre de ne pas avoir à gérer la partie HTTP. On va pouvoir directement instancier un *channel* à partir de son URL. Je vais essayer d'en faire quelque chose et je reviens...

[30 minutes passent]

Gardez toujours en tête que la première loi de la thermodynamique est toujours respecté, autrement dit une bonne nouvelle sera toujours accompagnée d'une mauvaise pour que l'univers reste en équilibre.

Donc je me suis retrouvé bloqué ici:

```rust
extern crate clap;
use clap::App;

extern crate rss;
use rss::Channel;

fn main() {
    let matches = App::new("ovht")
        .version("0.1.0")
        .author("Stéphane Depierrepont")
        .about("Watch http://travaux.ovh.net feed")
        .args_from_usage("-c, --count=[COUNT]   'Display last count entries (defaul=10)'
                          -s, --section=[SECTION] 'Section to watch (default=all)'")
        .get_matches();

    let section = matches.value_of("section").unwrap_or("all");
    let display_count = matches.value_of("count").unwrap_or("10");

    println!("section: {} display_count: {}", section, display_count );

    let channel = Channel::from_url("http://travaux.ovh.net/rss.php").unwrap();

    for item in channel.items() {
        match item.title() {
            None => { println!("???"); }
            Some(s) => { println!("{}", s); }
        }
    }
}
```

Avec le gentil message d'erreur:

```
Compiling ovht v0.1.0 (file:///home/toorop/Projects/rust/ovht)
error[E0599]: no function or associated item named `from_url` found for type `rss::Channel` in the current scope
  --> src/main.rs:23:19
   |
23 |     let channel = Channel::from_url("http://example.com/feed.xml").unwrap();
   |                   ^^^^^^^^^^^^^^^^^

error: aborting due to previous error
```
Mais ! mais !  cette méthode existe d'après [la doc](https://crates.io/crates/rss)

Je vais vous passer les détails car ça dépasse mes connaissances actuelles de Rust mais en gros il est possible d'activer, ou pas, des *features* dans une libraire. Je suppose que le but est de ne conserver dans le binaire que ce qui est utile.

Donc pour que cela fonctionne il ne faut pas juste mettre ```rss"= "1.0"``` dans la section *dependencies* mais:

```
[package]
name = "ovht"
version = "0.1.0"
authors = ["Stéphane Depierrepont aka Toorop <toorop@toorop.fr>"]

[dependencies]
clap = "2.29"

[dependencies.rss]
version = "1.0"
features = ["from_url"]
```

Bien on à notre *channel* RSS, si on regarde la [doc de la structure rss::Channel](https://docs.rs/rss/1.2.1/rss/struct.Channel.html) on voit la [méthode Items](https://docs.rs/rss/1.2.1/rss/struct.Channel.html#method.items) qui retourne un Vecteur d'éléments de type [Item](https://docs.rs/rss/1.2.1/rss/struct.Item.html)

Pour ceux qui se pose la question la différence entre un vecteur et un tableau (Array), c'est que l'*array* est de taille fixe alors que le *vector* est de taille variable. On peut ajouter/supprimer des éléments d'un vecteur pas d'un array.

Un vecteur implémente le *trait* [std::iter::Iterator](https://doc.rust-lang.org/std/iter/trait.Iterator.html), on va donc pouvoir bloucler sur les items de la façon suivante:

```rust
for item in channel.items() {
    on fait quelque chose avec notre item
}
```
Et ce quelque chose que l'on va faire c'est l'afficher à l'écran pardi !
Argh mais je crois que l'on va avoir un petit problème, voyons ça:

```
extern crate clap;
use clap::App;

extern crate rss;
use rss::Channel;

extern crate html2text;

use std::io::{BufReader, BufRead};

fn main() {
    let matches = App::new("ovht")
        .version("0.1.0")
        .author("Stéphane Depierrepont")
        .about("Watch http://travaux.ovh.net feed")
        .args_from_usage("-c, --count=[COUNT]   'Display last count entries (defaul=10)'
                          -s, --section=[SECTION] 'Section to watch (default=all)'")
        .get_matches();

    let section = matches.value_of("section").unwrap_or("all");
    let display_count = matches.value_of("count").unwrap_or("10");

    let channel = Channel::from_url("http://travaux.ovh.net/rss.php").unwrap();

    for item in channel.items() {
        match item.pub_date() {
            None => {println!("no date");}
            Some(s) => {
                println!("------------------------------------------------------------------------------");
                println!("{}\n", s)
            }
        }
        match item.title() {
            None => { println!("no title"); }
            Some(s) => { 
                println!("{}", s);
                }
        };
        match item.description() {
            None => { println!("no description"); }
            Some(s) => {
                println!("{}", s); 
            }
        };
        match item.link() {
            None => { println!("no link"); }
            Some(s) => { 
                println!("More info: {}", s); 
                println!("------------------------------------------------------------------------------\n")
            }
        };
    }
}

```

Quelques explications du code avant de causer du "petit problème", et en particulier sur le *pattern matching* amené par l'opérateur match :

```
match item.title() {
    None => { println!("no title"); }
    Some(s) => { 
        println!("{}", s);
    }
};
```

*match* permet de comparer une valeur à une série de valeurs/types possibles, nous comparons *item.title* à deux "entités"

- None: rien, le néant, peau de balle... autrement dit on va exécuter cette branche si notre *item* n'a pas d'attribut *title* de défini.
- Some(s): au contraire si notre *item* à un attribut *title* de défini, alors on va exécuter cette seconde branche.

Bien venons en à notre 'petit problème', que va t'il se passer si on exécute ce code ? Et bien testons
(je supprime tout ce qui ne sert à rien sinon ce billet va faire des kilomètres):

```
Wed, 24 Jan 2018 15:28:29 +0100

Web Hosting / CloudDB:: Logs cluster021
Logs and statistics (<a href="https://logs.cluster021.hosting.ovh.net">https://logs.cluster021.hosting.ovh.net</a>) are unavailable on cluster021.<br />
Investigations in progress.
More info: http://travaux.ovh.net/?do=details&id=29550

```

Et oui ! Le rendu HTML en console ce n'est pas ce qu'il y a de mieux !

Quand on à un problème dans la vie, il faut se dire que d'autres l'ont déjà eu (un peu comme les idées géniales), d'une part on se sent moins ~~con~~ seul et d'autre part il y a fort à parier qu'en demandant à notre moteur de recherches préféré on va tomber sur la solution de ce problème. Après de longues minutes de recherche et d'expérimentation, j'ai fini par trouver un *crate* qui fait le job: [html2text](https://crates.io/crates/html2text).

On remplace donc la section dédiée à la description par:

```rust
match item.description() {
    None => { println!("???"); }
    Some(s) => {
        println!("{}",  html2text::from_read(s.as_bytes(), 80)); 
    }
};
```

Et c'est tout de suite beaucoup plus beau non ?

```
Wed, 24 Jan 2018 15:28:29 +0100

Web Hosting / CloudDB:: Logs cluster021
Logs and statistics ([https://logs.cluster021.hosting.ovh.net][1]) are
unavailable on cluster021.
Investigations in progress.

[1] https://logs.cluster021.hosting.ovh.net

More info: http://travaux.ovh.net/?do=details&id=29550
```

Bien on je vous propose que l'on fasse un petite pause...


## On efface tout et on recommence

... pause qui a duré 2 jours ;)

Bon ce temps m'a permis de réfléchir à ce petit programme, et je me suis dis que ce serait quand même mieux si c'était un minimum utile. Afficher les événements dans une console mouais bof, c'est rigolo mais on ne va pas laisser une console ouverte pour ça. 

J'ai d'abord pensé à les afficher sous forme de notifications, mais c'était pour le coup un peu trop intrusif, et puis je me suis dit est si je les affichais dans mon serveur **Discord** préféré ? ([celui là ?](https://discord.gg/0eUtmzEdD3A06LmH))

Du coup j'ai cherché ~~une librairie~~ un crate pour causer avec Discord et j'ai trouvé [Serenity](https://crates.io/crates/serenity). Le principe de base consiste a créer un utilisateur sous forme d'un bot et une fois connecté au serveur il pourra envoyer des messages dans un channel Discord dédié.

Par la même occasion je me suis dit qu'on allait plus avoir besoin de transmettre des options via la ligne de commande, et j'ai donc supprimé ce que l'on a fait plus haut avec **Clap**.


## Le bot Discord

Je vous passe la phase de création d'une "app Discord", qui va nous permettre d'obtenir un token d'authentification pour notre bot. Vous trouverez de nombreux tutos sur le net.

L'implémentation la plus rudimentaire (pas de gestion d'erreur) pour envoyer un message dans un channel est :

```rust
let channel = ChannelId(ID_DU_CHANNEL);
channel.say("hello world");
```
On initialise un channel et on cause dedans, rien de mystérieux.

Voici le code final:

```rust
extern crate rss;
use rss::Channel;

extern crate html2text;

extern crate chrono;
use chrono::prelude::*;

extern crate serenity;

use serenity::model::gateway::Ready;
use serenity::prelude::*;
use serenity::model::id::ChannelId;


use std::time::Duration;
use std::thread;
use std::env;



// Discord Handler
struct Handler;

impl EventHandler for Handler {
    fn ready(&self, _: Context, ready: Ready) {
        println!("{} is connected!", ready.user.name);

        // last OVH event timestamp (now() at launch)
        let mut last_update = Utc::now().timestamp();

        // to infinity and beyond... 
        loop {
            // RSS channel
            let channel = Channel::from_url("http://travaux.ovh.net/rss.php").unwrap();

            // reverse items -> older to newer
            // there must be a more idiomatic way... (alert gopher spotted !!!)
            let mut items = Vec::new();
            for item in channel.items() {
                items.push(item)
            }
            //let mut items = vec![channel.items()];
            items.reverse();

            // Discord channel where message will be pushed
            let ovh_channel = ChannelId(CHANNEL_ID);

            // loop over feed items
            for item in items {
                // convert string pub_date to timestamp 
                let pub_date = match item.pub_date() {
                    None => "Thu, 25 Jan 2118 11:53:40 +0100",
                    Some(s) => s,
                };
                let ts = match DateTime::parse_from_str(pub_date, "%a, %d %b %Y %T %z") {
                    Ok(ts) => ts.timestamp(),
                    Err(e) => {
                        println!("error {}", e);
                        continue;
                    }
                };

                // new event ?
                if ts > last_update {
                    last_update = ts;
                    let title = match item.title() {
                        None => "",
                        Some(s) => s,
                    };
                    if title == "" {
                        continue;
                    };

                    // strip HTML tags
                    let description = match item.description() {
                        None => String::from(""),
                        Some(s) => html2text::from_read(s.as_bytes(), 80),
                    };

                    // get link 
                    let link = match item.link() {
                        None => "",
                        Some(s) => s,
                    };

                    // display event on console
                    print!("{}\n{}\n{}\n\n", title, description, link);

                    // send envent to Discord Channel
                    if let Err(e) =
                        ovh_channel.say(format!("{}\n{}\n{}\n\n", title, description, link))
                    {
                        println!("Error sending message {:?}", e);
                    }
                };
                // go work ! take a snap
                thread::sleep(Duration::from_secs(60));
            }
        }
    }
}



// Main
fn main() {
    // init Discord Client
    let token = env::var("DISCORD_TOKEN").expect("Expected a token in the environment");
    let mut client = Client::new(&token, Handler).expect("Err creating client");

    if let Err(e) = client.start() {
        println!("Client error: {:?}", e);
    }
}
```

J'ai commenté, a minima, ce qui devrait vous permettre de suivre le déroulement du code.

Et vous savez quoi ? ça fonctionne !!  


![rust discord bot](/images/2018/01/24/bot-discord-rust-ovh.png#center)

Félicitations si vous avez tenu le coup jusque là !!!

Si vous voulez voir le bot en action [c'est ici](https://discord.gg/0eUtmzEdD3A06LmH)

Amis développeurs Rust n'hésitez pas à me corriger et/ou à suggérer des améliorations dans le code existant.

A++

