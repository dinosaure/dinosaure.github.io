---
layout:   post
date:     2017-05-29 03:49:08
title:    "Sirop de PACK"
category: blog
slug:     sirodepac
---

Voilà un petit moment (4 mois) que je travaille en ce moment sur
l'implémentation à la sérialisation et la déserialisation d'un fichier PACK
(fichier central au logiciel `git`).

J'ai vachement appris sur la structure de `git` et surtout sa puissance. En
effet, `git` est notamment en capacité de gérer des gros fichiers sans pour
autant que vous ayez besoin d'investir dans des disques durs. C'est notamment en
lien avec une architecture du traitement de l'information particulière qui prime
toujours le stockage dans le système de fichiers plutôt que la mémoire.

## Non bloquant

En réalité `git` s'articule avec l'idée simple qu'il n'est pas possible de
stocker vos objets `git` en mémoire. Ainsi, dès que vous manipulez un objet,
`git` va le traiter le traiter dans un flux (qu'il enverra ensuite dans la
sortie standard petit à petit) et techniquement parlant, c'est `less` qui montre
le résultat en lisant depuis l'entrée standard.

Ainsi, seule une représentation très succincte (correspondant aux *metadata* de
vos objets `git`) est représenté en mémoire. Le contenu (notamment celui de vos
*blob*) n'est jamais chargé en mémoire. On préférera alors traiter le contenu
petit à petit et écrire le résultat petit à petit.

L'idée de conception non-bloquante est donc cruciale et elle est d'autant plus
importante quand il s'agit de refaire `git` mais en OCaml - dans un contexte,
donc, où la gestion de la mémoire est forcément moins fine qu'en C vu qu'il y a
un Garbage Collector.

Elle permet ainsi d'allouer des *buffers* de taille fixe qui contiendront à
chaque étape les parties de notre objet `git` en continue. Et c'est ainsi qu'on
peut considérer ça comme un flux. Ensuite, l'aspect non bloquant exige à ce que
le processus de sérialisation/déserialisation puisse s'arrêter à n'importe
quelle moment (et spécifiquement quand le *buffer* est plein).

On se gardera, au niveau de l'interface, de bien cacher tout ce qui est
nécessaire afin de reprendre, plus tard, le traitement. Il s'agit surtout de
s'émanciper de pré-requis comme exiger à ce que le traitement commence,
imaginons avec un *buffer* de 32K bytes.

## Back to OCaml

Et c'est ainsi que l'on peut sérieusement contrôler l'empreinte mémoire de notre
logiciel même si on utilise un Garbage Collector. Pour ceci, il faut bien
comprendre que OCaml à un GC avec 2 générations. La première consiste aux
données petites et où l'allocation est très rapide, la deuxième génération
consiste à contenir nos grosses valeurs *caml*.

Il faut cependant comprendre un comportement d'OCaml qui consiste à promouvoir
une valeur *caml* de la première génération à la deuxième. Ceci arrive souvent
lorsqu'on utilise `mutable` (ainsi, les effets de bords). Ainsi, si une valeur
`mutable` dure longtemps en OCaml, elle peut être promu à la deuxième
génération.

Et dans notre contexte, il faut absolument éviter ça puisqu'en étant majoré à la
deuxième génération, la valeur restera alloué plus longtemps (et pourrait même
continuer à exister longtemps même si on ne l'utilise plus).

En ce sens, __toutes__ les valeurs OCaml utiles au traitement doivent être
localisées dans la première génération pour assurer une choses essentielle, à ce
que leurs cycle de vie soient le plus court possible (et que l'empreinte mémoire
requise reste toujours mineur). Et en une seule contrainte, ceci est possible:
ne pas utiliser `mutable`.

Il est préférable donc dans le contexte d'OCaml d'utiliser des données immuables
même si elles doivent être (pour se *modifier*) allouer fréquemment. Mais pour
le coup, l'allocation dans le tas mineur correspond à 3 instructions processus.
Le *deal* est donc bon.

## L'impact

La finalité est que pour des gros fichiers PACK, on contrôle l'empreinte mémoire
du traitement même si on ne le fait pas finement comme en C (où l'on peut
contrôler au byte près). Les *états* nécessaires aux traitements sont donc
localisés uniquement dans la première génération qui a une empreinte mémoire
totale limité (et qu'on peut contrôler avec `OCAMLPARAM`) et s'assurer le
stricte nécessaire à la sérialisation/déserialisation en allouant les *buffers*
nécessaire avec une taille fixe que l'on peut jauger selon le contexte
d'utilisation.

Ce travail n'est pourtant pas si simple, il s'agit de jouer entre les états sur
des *buffers* donc le contenu est très volatile et sur lesquelles une erreur
peut tout simplement nous faire perdre les données. Il s'agit donc d'être très
minutieux sur le traitement de ces *buffers* et savoir quand il est légitime de
les réutiliser (et donc écraser les données).

## L'état de l'art

Vous pouvez suivre l'implémentation sur
mon [GitHub](https://github.com/dinosaure/sirodepac). Le code est assez commenté
(en tout cas, les parties complexes) et vous pouvez vous amuser à re-créer un
fichier PACK (qu'on obtient avec un `git gc` et qui se trouve dans
`.git/objects/pack/`) avec cette commande:

> ./optimized_unpack.native pack-*.{pack,idx}

Pour ce qui est de la compression, je ferais un autre article qui décrit
précisément le point - car vous pouvez le constater, mon logiciel parvient à
être presque aussi efficace que `git`!


<!--  LocalWords:  l'implémentation sérialisation déserialisation
 -->
