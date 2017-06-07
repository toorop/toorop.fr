+++
date = "2017-06-06T11:32:00+02:00"
draft = false
title = "Mon blog est mort, vive mon blog !"
slug = "Mon blog est mort vive mon blog"
categories = [ "Blog", "Plog"]
tags = [ "dev", "golang"]
toc = true # Optional
thumbnail = "images/hugo.png"

+++
Mon ancien blog, qui avait plus de 10 ans, nous a quitté hier :(  
Le pire ne décevant jamais les différentes sauvegardes que j'avais faites, étaient corrompues.  
<!--more-->  
Je ne rentre pas dans les détails, si la petite histoire vous intéresse je vous invite à écouter l'épisode de l'instant T consacré à ce drame:

<div class="player">
    <audio controls preload="none">
        <!-- Audio files -->
        <source src="http://api.spreaker.com/download/episode/12044770/mon_blog_est_mort_vive_mon_blog.mp3" type="audio/mp3">
        <!-- Fallback for browsers that don't support the <audio> element -->
        <div>
            <a href="http://api.spreaker.com/download/episode/12044770/mon_blog_est_mort_vive_mon_blog.mp3">Votre navigateur ne supporte pas la lecture du fichier audio. Cliquez ici pour télécharger l'épisode.</a>
        </div>
    </audio>
</div>

Puisque l'on en parle, si vous souhaitez suivre mes aventures trépidantes en audio, vous pouvez vous abonner à mon podcast | streetcast | plog : l'instant T disponible à cette adresse <a href="http://feeds.feedburner.com/InstantT" target="_blank">http://feeds.feedburner.com/InstantT</a>

Comme vous pouvez le constater, puisque vous êtes ici, pour cette renaissance j'ai changé de moteur de blog, pour utiliser <a href="https://gohugo.io/" target="_blank">Hugo</a> plutôt que <a href="https://fr.wordpress.org/" target="_blank">Worpdress</a>.

Pourquoi ?

* Hugo génère du contenu statique (html + css + js + ...), autrement du côté serveur, il n'y aucune création de page.  
* c'est trés rapide.
* un stack minimal à installer et à maintenir côté serveur. Ce blog tourne par exemple sur une <a href="https://www.ovh.com/fr/public-cloud/instances/" target="_blank" alt="instance ovh pour Hugo Caddy">instance OVH</a> de base.
Et si vous ne voulez pas vous embêter à gérer une instance/serveur, vous pourrez héberger votre site n'importe où puisque l'on ne sert que tu contenu statique (pas besoin de CGI, PHP, Perl, Ruby, Node,...)
* pas de risque de hack (que celui qui n'a jamais croisé un Wordpress hacké... se rendorme).
* facilement personalisable. Il existe de nombreux thèmes et si vous voulez mettre les mains dans le cambouis vous pouvez relativement simplement créer vos propre template  classique.
* c'est programmé en <a href="https://golang.org/" target="_blank" alt="go programming language">Go</a>. C'est peut être un détail pour vous mais pour moi ça veut dire beaucoup, ça veut dire qu'il était... heu... qu'est ec que je disais déjà ?
* vous pouvez rédiger des articles en étant hors ligne (c'est du <a href="https://fr.wikipedia.org/wiki/Markdown" target="_blank" alt="qu'est ce que markdown">Markdown</a>), il vous suffira de les pousser sur votre hébergement une fois que vous retrouverez le monde civilisé.
* PARCE QUE J'AVAIS ENVIE !!!

Bon il y a quelques inconvénients liés au fait que ce soit 100% statique... mais il sont assez facilement contournables:

* commentaires: même si ce n'est pas l'idéal, il suffit d'utiliser <a href="https://disqus.com/" target="_blank" alt="systeme de commentaire disqus">Disqus</a>
* recherches dans le site: honnêtement, vous utilisez souvent la fonction de recherche intégrée à un blog ? Au pire il y a Google.

Voila, il ne me reste plus qu'a trouver, le temps et la motivation pour vous rédiger des billets ;)

