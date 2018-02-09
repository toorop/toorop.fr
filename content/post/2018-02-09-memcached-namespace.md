+++
draft = false
title = "Memcached et espace de noms"
date = 2018-02-09T08:56:32+01:00
slug = "memcached-wildcard-namespace"
tags = ["memcached", "golang"]
categories = ["devlogs"]
thumbnail = "images/2018/02/09/memcached-namespace.png"
toc = true # Optional
+++

Je suppose que si vous êtes là c'est que vous connaissez [Memcached](https://memcached.org/) ?  
Non ?  
Alors je vais commencer par un bref rappel, [Memcached](https://memcached.org/) est un système de **stockage clé->valeur**. Une de ses particularités est que le stockage des données est faite en RAM ce qui rends la bestiole très véloce. Le cas d'usage typique  de Memcached est de s'en servir comme cache pour soulager et accélérer les requêtes vers les bases de données. 

Prenons un exemple concret avec un blog, à chaque affichage d'un billet, le moteur du blog va aller le chercher dans la base de données. En y réfléchissant on se dit: hum les billets ne changent pas si souvent, du coup c'est quand même couillon de faire des requêtes vers la base de données qui vont consommer des ressources, prendre du temps et tout ça pour retourner le même résultat à chaque fois !

Et on en arrive à:  

**Bon sang mais c'est bien sûr ! Utilisons un cache !**

Une utilisation classique d'un cache est de se dire que l'on peut accepter un certain décalage entre l'état d'un objet en DB et l'état que l'on présente à l'utilisateur. Revenons à notre blog, on peut considérer que la modification d'un billet peut être répercuté aux visiteurs 15 minutes plus tard. Du coup lorsque l'on va cacher le billet, on va définir une durée de validité de 15 minutes et tout le monde est content. 

Cette façon de faire présente cependant quelques problèmes: 

- en général un article change rarement du coup c'est dommage de le mettre en cache que 15 minutes.
- il y a des cas où il va être nécessaire qu'un changement du billet soit répercuté sans délais aux visiteurs.

Comment faire ?

Et bien il suffit de ne pas mettre de durée de vie à la valeur cachée et de révoquer le cache quand l'article change pardi !!

On va rentrer un peu dans le concret avec une implémentation en pseudo code:

```
func GetArticle(ID) article {    
    
    // on commence par définir la clé qui va correspondre à cet article dans le cache
    cle = "article_"+ID
    
    // on regarde dans le cache si il y a l'article
    article = CacheGet(cle)

    // si on a trouvé l'article dans le cache on le retourne
    if article {
        return article
    }

    // il n'y avait l'article en cache on va le chercher dans la base de donnée
    article = DBGet(ID)

    // on le met en cache
    CacheSet(cle, arcticle)

    // on retourne l'article
    return article
}
```

Maintenant voyons la fonction pour mettre a jour un article

```
func UpdateArticle(article){
    
    // comme tout à l'heure defini la clé qui correspont à cet article en cache
    cle = "article_"+article.ID

    // on update la base de données pour quelle reflete le nouvel état de l'article
    DBUpdate(article)

    // la valeur en cache n'est plus valide, il nous faut la supprimer
    CacheDelete(cle)
}
```

Comme on est devenu un pro du cache on se dit que l'on va en coller sur toutes les fonctions qui vont accéder à la base de données. Les visiteurs seront contents car ce sera plus rapide et les admins aussi car la DB sera moins sollicitée.

Allez on commence par une fonction très gourmande pour la base de données, celle qui va retourner les X derniers articles de la catégories Y qui ont le tag Z:

```
func GetArticles(x,y,z) article {    
    
    // on commence par définir la clé qui va correspondre à cette recherche
    cle = "articles_"+x+"_"+y+"_"+z
    
    // on regarde dans le cache si on a le résultat
    articles = CacheGet(cle)

    // si oui on les retourne
    if articles {
        return articles
    }

    // si non on tape dans la DB
    articles = DBGetArticles(x,y,z)

    // on met le resultat en cache
    CacheSet(cle, arcticles)

    // on retourne les articles
    return articles
}
```

Easy non ?!

Pris dans notre emballement, on colle du cache partout.

**Et là c'est le drame !**

Pourquoi ?  

Et bien si on modifie un article, la fonction GetArticles(x,y,z) ne va pas en tenir compte et va continuer à nous retourner les mêmes articles depuis le cache.

Comment solutionner le problème ?  

Facile ! lorsque l'on appelle *UpdateArticle()* il suffit de supprimer toutes les clés associées aux articles.

C'est facile en apparence mais ça ne l'est pas tant que ça car on ne connaît pas les clés qui ont été créés, par exemple la fonctions *GetArticles(x,y,z)* peut en avoir crée des centaines.

Et bien il suffit de supprimer toutes celles qui commencent par un même préfixe, par exemple par "article*" ?

Oui mais Memcached ne permet pas de faire des requêtes avec des "wildcard" sur les clés.

Hum et bien dans ce cas il suffit d'itérer sur toutes les clés et de supprimer celles qui commencent par "article" ?

Heu... je te rappelle qu'a la base, on a mis un cache pour gagner en rapidité et en ressources !!!

Mais comment faire alors ???

La solution à cette problématique est d'utiliser des **espaces de noms**.

On va ajouter les fonctions GetNamespace() et InvalidateNamespace(), voici leur implémentation en pseudo code :

```
// cette fonction va retouner le namespace à utiliser en fonction de la clé de namespace
GetNamespace(){
    // on va checher le namespace en cache
    nameSpace = CacheGet("namespaceArticle")

    // un namespace correspondant à cette clé existe on le retourne
    if namespace {
        retun namespace
    }

    // il n'y a pas de namespace défini pour cette clé on en cré un
    // on crée un namespace unique avec un rand ou un ts ou..
    namespace = "namespaceArticle" + rand() 

    // on met ce namespace en cache
    CacheSet("namespaceArticle", namespace)

    // on retourne le namespace
	return namespace
}
```

Et pour InvalidateNamespace():

```
func InvalidateNamespace(nameSpaceKey) {
    // on crée un nouveau namespace
    namespace = "namespaceArticle" + rand() 

    // on met ce namespace en cache
    CacheSet("namespaceArticle", namespace)
}
```

Ensuite il nous suffit d'utiliser ces namespaces avec nos fonctions. GetNamespace() va être utilisée par nos fonctions de lecture et on va utiliser InvalidateNamespace() pour chaque fonction qui va écrire dans la base de données.

Reprenons les exemples du début, GetArticle devient:

```
func GetArticle(namespaceKey, ID) article {    
    
    // on commence par définir la clé qui va correspondre à cet article dans le cache
    cle = GetNamespace() + "article_" + ID
    
    // on regarde dans le cache si il y a l'article
    article = CacheGet(cle)

    // si on a trouvé l'article dans le cache on le retourne
    if article {
        return article
    }

    // il n'y avait l'article en cache on va le chercher dans la base de donnée
    article = DBGet(ID)

    // on le met en cache
    CacheSet(cle, article)

    // on retourne l'article
    return article
}
```
Et UpdateArticle devient:

```
func UpdateArticle(namespaceKey,article){
    
    // on update la base de données pour quelle reflete le nouvel état de l'article
    DBUpdate(article)

    // on revoke le précedent cache en créant un nouveau namespace
    InvalidateNamespace()
}
```

Ainsi à chaque modification d'article, le **namespace** utilisé pour cacher les résultats des requêtes change, ce qui va entraîner l'invalidation des précédentes valeurs cachées.

Comme rien n'est parfait cette méthode apporte quelques effets négatifs:

- on va devoir faire deux requêtes vers le cache au lieu d'une. La première pour récupérer le namespace, le second pour récupérer la valeur cachée. Rassurez vous ça reste nettement plus rapide et moins gourmand que de requêter directement la base de données.
- les anciennes valeurs cachées, contenues dans les namespaces invalidés, ne sont pas supprimées du cache. Il va donc arriver un moment où toutes la RAM réservée pour memcached va être utilisée. Ce n'est pas un réel problème car quand memcached n'a plus de place pour stocker des nouvelles valeurs il va de lui même supprimer celles qui sont le moins utilisées. Donc le ménage va se faire tout seul.

Voila, si vous avez des questions n'hésitez pas.

Et n'hésitez pas à nous rejoindre sur [Discord](https://discord.gg/0eUtmzEdD3A06LmH)

PS: je sais que l'on peut utiliser d'autres systèmes de stockage clé->valeur qui eux implémentent du "wildcarding" (redis par exemple).


