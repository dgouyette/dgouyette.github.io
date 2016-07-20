---
layout: post
title:  "Introduction pratique aux Macros Scala"
date:   2016-07-15 09:00:00
categories: scala
---

Des macros, oui, mais pour quoi faire ?
----------------

Ce sont des fonctions appelées lors de la compilation du projet.

Cela permet de générer par exemple :

* l'implémentation d'une méthode,
* des méthodes dans un objet compagnon


Quelques macros que j'utilise régulièrement
===

* Génération de reader / writer json : [play-json](https://github.com/playframework/playframework/tree/master/framework/src/play-json/src/main/scala/play/api/libs/json),
* [Shapeless](https://github.com/milessabin/shapeless) utilise également les macros scala, par exemple sur [shapeless.Generic](https://github.com/milessabin/shapeless/blob/master/core/src/main/scala/shapeless/generic.scala), cela permet de passer facilement d'une case class à une HList,
* Librairie d'injection de dépendance [macwire](https://github.com/adamw/macwire),
* Lenser [Monocle](https://github.com/julien-truffaut/Monocle)...

Structure du build d'un projet de macro
---

        + core
        + macros
        - build.sbt



Premier cas pratique : afficher les infos Git à la compilation
---

Exemple basé sur [sbt-buildinfo](https://github.com/sbt/sbt-buildinfo)
Le but est d'afficher les informations courantes GIT (commit courant, branche, dernier utilisateur...)

~~~ scala
        case class Commit(author : String, hash : String, time : Date)
        case object BuildInfo {
          val branch : String = ???
          val commit: Commit = ???
        }
~~~


Deuxième cas pratique : ajouter des méthodes à la volée sur un objet compagnon
---

Générer des méthodes de conversions implicites

Statut des macros
---

* Scala macros
* Scala meta
* Quasiquotes

...


En savoir +
----------
[^footnote]:
  >[Documentation](http://docs.scala-lang.org/overviews/macros/overview.html)
