+++
categories = ["blog"]
date = "2018-01-03T10:07:53+02:00"
draft = false
slug = "ovh-mail-spam-vpn"
tags = ["ovh"]
thumbnail = "images/2018-01-03/spam.jpg"
title = "Vos mails vers OVH n'aboutissent pas si vous utilisez un VPN ?"
toc = true
+++



**tl;dr** : si vous envoyez des mails vers OVH depuis un poste qui est derrière un VPN il y a de fortes chances pour que vos mails soient, au mieux classifiés comme spams, au pire dropés.

**UDATE 05 janvier 2018**

Suite à la publication de ce billet sur Twitter le support OVH à fait remonter le problème aux équipes concernées:

<div align="center">
<blockquote class="twitter-tweet" data-lang="fr"><p lang="fr" dir="ltr">Bjr. Intéressant effectivement, j&#39;ai remonté aux admins pour vérifications. Merci. Cdlt, ^ThoT</p>&mdash; OVH Support FR (@ovh_support_fr) <a href="https://twitter.com/ovh_support_fr/status/948564866276585472?ref_src=twsrc%5Etfw">3 janvier 2018</a></blockquote>
<script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>
</div>

Ce matin j'ai refait deux tests, un vers une des mailing OVH (sd-pro) et l'autre vers une adrsse hébergée sur leur plateforme mutualisée, il semblerait que le problème n'est plus d'actualité puisque les deux mails sont arrivés.

Le *spamcause* de ces mails sont identiques et ne montrent aucun probleme:

```Vade Retro 01.401.76#118 AS+AV+AP+RT Profile: OVH|dp|CHECKCE; Bailout: 300;```

**Billet original**

Comme je ne sais pas trop où remonter le problème, je le pose ici en espérant que ça remonte aux personnes concernées.

Depuis quelque temps je constatais que mes mails envoyés vers les ml OVH n'aboutissaient pas. Mes logs montraient qu'ils étaient bien pris en charge par OVH mais ensuite ils tombaient dans un trou noir.

Ce matin j'ai voulu creuser un peu car autant ce n'est pas dramatique si mes mails vers les mailing-lists OVH n'aboutissent pas autant ça l'est beaucoup plus si ceux à destination de mes contacts pro subissaient le même sort. J'ai donc fait des tests en envoyant des mails vers une adresse hébergée sur la plateforme mutualisée OVH.

Grace à ces tests j'ai vite pu localiser la source du problème: le service antispam utilisé par OVH, Vade Retro, considère que mes mails sont des spams. Voici un exemple des headers après classification par Vade Retro:

```
X-VR-SPAMSTATE: SPAM
X-VR-SPAMSCORE: 300
X-VR-SPAMCAUSE: gggruggvucftvghtrhhoucdtuddrgedtuddrieehgddufeeiucetufdoteggodetrfdotffvucfrrhhofhhilhgvmecuqfggjfdpvefjgfevmfevgfenuceurghilhhouhhtmecufedttdenucgorfhhihhshhhinhhgqdfkphfpvghtfihorhhkucdlfedttddm
X-Ovh-Spam-Status: SPAM
X-Ovh-Spam-Reason: vr: SPAM; dkim: disabled; spf: disabled
X-Ovh-Message-Type: SPAM
```
Pour connaître la raison de cette classification il suffit de décoder le spamcause:

```Vade Retro 01.401.66#15 AS+AV+AP+RT Profile: OVH|pd|CHUCKCU; Bailout: 300; Phishing-Ipnetwork (300)```

La règle qui a engendré la classification comme spam est donc **Phishing-Ipnetwork**

C'est le moment de se creuser un peu la tête, qu'est que ça peut bien être ?

Au début j'ai pensé a une URL utilisant une IP, un truc du style:

```<a href="http://111.222.333.444/login> https://ovh.com/managerv42</>```

Mais non pour la bonne et simple raison que je poste en texte brut.

Une gorgée de café plus loin je me suis dis: et si c'était parce que mon IP, celle qui est à l'origine du mail, n'est pas une IP résidentielle mais mais une IP d'un serveur, en l’occurrence l'IP de sortie de mon VPN.

J'ai désactivé le VPN et... **miracle !!!!**:

```
X-VR-SPAMSTATE: OK
X-VR-SPAMSCORE: 49
X-VR-SPAMCAUSE: gggruggvucftvghtrhhoucdtuddrgedtuddrieehgdduheduucetufdoteggodetrfdotffvucfrrhhofhhilhgvmecuqfggjfdpvefjgfevmfevgfenuceurghilhhouhhtmecufedttdenucgoufhushhpvggtthffohhmrghinhculdegledm
```

Bon ce n'est pas encore parfait car d'aprés le spamcause j'utilise un domaine suspect:

```Vade Retro 01.401.65#151 AS+AV+AP+RT Profile: OVH|dp|CHECKCE; Bailout: 300; SuspectDomain (49)```

Je me demande bien lequel... ;-)

Ha ! je vois une main qui se lève au fond de la salle, vas y je t'écoute:

> heu mais du coup ça veut dire que les mails générés depuis mes serveurs n'aboutiront pas si le destinataire est chez OVH ??!!!

Non, enfin je n'ai pas eu le temps de tester, mais je suppose que l'algo de vade retro fonctionne de la façon suivante:

```
Si headers indiquant qu'un client mail à été utilisé:
    Si l'IP de sortie est une IPnetwork®™©: 
        -> SPAM.
```
Ou alors c'est beaucoup plus simple, c'est juste qu'il y a une IP qui ne leur plait pas dans la liste des relais,  celle de l'instance [Scaleway](https://www.scaleway.com/) de mon VPN... Non ça ne peut être aussi simple ~~iste~~...

Cela dit ça reste assez problématique quand même car si vous "sortez" via une IP qui est cataloguée dans la liste des IPnetwork®™© vos correspondants, si ils ont leur messagerie chez OVH, ont peu de chances de recevoir vos mails.