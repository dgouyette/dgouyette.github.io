---
layout: post
title:  "Bowling avec Shapeless"
date:   2015-09-25 09:00:00
categories: scala Shapeless
---


Préambule
----------
Cet article est la synthèse d'un [kata](http://butunclebob.com/ArticleS.UncleBob.TheBowlingGameKata) realisé à Vidal.
Cet exercice a été réalisé  afin de découvrir (un peu) la librairie Shapeless.
Afin de sortir des exemples académiques, nous sommes partis sur un cas d'usage ludique pouvant s'y prêter.

Les personnes ayant participé :

* [@malk_zameth](https://twitter.com/malk_zameth)
* [Sophie Wachs](https://fr.linkedin.com/pub/sophie-wachs/64/9b9/528)
* [@Le_3K](https://twitter.com/Le_3K)


Nous l'avions précédemment résolu en utilisant une List[Int] (de longueur non définie), du pattern matching et de la récursion.
Vous pouvez [retrouver ici le code du précédent kata](https://gist.github.com/dgouyette/e10573e6a8a110e902a0)

Suite à cet exercice, nous  voulions ajouter davantage de contraintes de types sur les manches et sur le jeu.

Cet article présente la façon dont nous avons résolu ce problème en utilisant [Shapeless](https://github.com/milessabin/shapeless)

Les différents types
--------------------

* Une manche peut contenir de 1 à 2 boules,
* La dernière manche peut contenir de 2 à 3 boules,
* Un jeu contient 9 manches normales et une dernière manche.


Listes hétérogènes (HList)
------------------
Une **HList** est un tuple de longueur arbitraire.

Afin de stocker les informations des manches, nous allons devoir définir deux alias de type :

{% highlight scala %}
//les manches peuvent avoir 1 ou 2 boules
type Round = Int :: Int :: HNil
{% endhighlight%}

{% highlight scala %}
//la dernière manche peut avoir 1, 2 ou 3 boules
type LastRound  Int :: Int :: Int :: HNil
{% endhighlight%}

Round et LastRound sont des sous-types d'une HList.

Rappel sur le polymorphisme
----------------------------------------------

Le polymorphisme est une fonctionnalité d'un langage informatique consistant à fournir à une interface unique des entités pouvant avoir des types différents :

On distingue généralement 3 types de polymorphisme :

* **Ad hoc (ou surcharge)** : permet de choisir différentes implémentation d'une même fonction selon le nombre et le type des arguments reçus.
* **Paramètrique**  : une  fonction unique prend en argument un type générique,
* **Héritage & sous-types** : une fonction est fournie prennant en argument un type de classe de la même hierarchie (animal :  chat, chien...)  

En scala, il est possible d'adapter le type des argument d'une fonction en utilisant une **conversion implicite**.


Score d'une manche
-------------------------------

Le polymorphisme paramétré associé à une fonction de conversion implicite nous permettra de manipuler facilement les types **Round** et **LastRound**.

Dans notre cas, nous devons calculer un **Score** à partir d'un **Round** ou d'un **LastRound**.

* La fonction de conversion implicite **caseRound**  permet de passer du type Round  à Int.
* La fonction de conversion implicite **caseLastRound**  permet de passer du type LastRound  à Int.

Round et LastRound sont également des HList, ce qui nous permet d'utiliser la même fonction **sum**.

{% highlight scala  %}
  Score(3::1::HNil) shouldEqual 4
{% endhighlight scala %}

Le paramètre d'entrée (ici un Round qui est aussi une HList) sera passé à la fonction sum qui nous retournera finalement un **Int**

{% highlight scala  %}
object Score extends Poly1 {
   implicit def caseRound = at[Round](sum)
   implicit def caseLastRound = at[LastRound](sum)
   ...
   def sum(round: HList):Int = round match {
        //Round = somme des deux boules
        case (h1: Int) :: (h2: Int) :: HNil => h1 + h2
        //LastRound = somme des trois boules en mode récursif
        case (h1: Int) :: (h2: Int) :: (h3: Int) :: HNil => h1 + sum(h2 :: h3 :: HNil)
        case _ => 0
     }
    ...
 }

{% endhighlight scala %}

La première branche du pattern matching sera utilisée soit **h1+h2**

{% highlight scala  %}
test("A round  0 + 0  = 0"){
  Score(0::0::HNil) shouldEqual 0
}
test("A  round 3 + 1  =  4") {
  Score(3::1::HNil) shouldEqual 4
}
{% endhighlight %}

Score d'une dernière manche :
-------------------------------
La deuxième branche du pattern matching sera utilisée.

{% highlight scala  %}
test(" Last round 0 + 0 + 0 = 0" ) {
  Score(0:: 0:: 0::HNil) shouldEqual 0
}
{% endhighlight %}

Score sur plusieurs manches :
-------------------------------

{% highlight scala  %}
test("Three rounds 1 + 2 +  + 1 +  2  +  1 + 2 =  1+2 + 1+2 + 1+2") {
  Score((1::2::HNil) :: (1::2::HNil) :: (1::2::HNil) :: HNil) shouldEqual 1+2 + 1+2 + 1+2
}
{% endhighlight %}

Alias de type nous permettant de faire un TU avec seulement 3 Rounds

{% highlight scala  %}
type GameWithThreeRound = Round :: Round :: Round :: HNil
{% endhighlight %}


Pour calculer le score d'une partie (soit une HList de Round & LastRound), nous aurons besoin d'une fonction de conversion qui pars d'une HList vers un Int soit **rounds2Score**

Ici chaque calcul de Round se fera via la première branche du pattern matching. Quand la liste sera terminée, soit HNil, fin de la récursion.

{% highlight scala  %}
object Score extends Poly1 {
...
implicit def caseGameWithThreeRound = at[GameWithThreeRound](rounds2Score)
...
def rounds2Score: (HList) => Int = {
        ...
        case (m1: Round) :: (t: HList) => Score(m1) + rounds2Score(t)
        case HNil => 0
     }
}
{% endhighlight %}


Calcul d'un spare.
------------------

Un **spare** est comptabilisé quand la h1+h2 = 10 mais que h1 est < 10 (sinon c'est un strike).
La première boule de la prochaine manche sera alors comptée deux fois.


{% highlight scala  %}
//La première boule de la deuxième et troisième manche devra être comptée 2 fois.
test("Three rounds  0+10 + 5+5 +  1+2  =  0+10+ 5+5+5 + 1+1 + 2") {
    Score((0::10::HNil) :: (5::5::HNil) :: (1::2::HNil) :: HNil) shouldEqual 0+10 + 5+5+5 + 1+1 + 2
  }
{% endhighlight %}


{% highlight scala  %}
//un spare n'est pas un strike, mais la somme des deux boules fait 10
def hasSpare(round: Round) = round match {
        case h1 :: h2 :: HNil if !hasStrike(round) && h1 + h2 == 10 => true
        case _ => false
}
def spare(m: Round): Int = {
    m match {
      //compte la première boule une fois (Round & LastRound)
      case (a:Int)::_ => a
      case _ => 0
    }
}
...
def rounds2Score: (HList) => Int = {
        ...
        case (m1: Round) :: (m2: Round) :: tail if hasSpare(m1) => sum(m1) + spare(m2) + rounds2Score(m2 :: tail)
        ...
}
{% endhighlight %}

Calcul d'un strike.
------------------

Un **Strike** est comptabilisé quand la h1 = 10 (cas d'un Round simple) ou h1, h2, h3 = 10 (LastRound).
Les deux prochaines boules (qui sont, ou pas dans la même manche) seront comptées deux fois.

{% highlight scala %}
test("two strikes = 30"){
  Score(strike :: strike :: HNil) shouldEqual 30
}

test("three strikes = 60"){
  Score(strike :: strike ::  strike ::  HNil) shouldEqual 60
}
{%endhighlight%}

{% highlight scala  %}
//matching uniquement sur Round car LastRound sera traité autrement
def hasStrike(round: Round) = {
        round match {
           case 10 :: _ :: HNil => true
           case _ => false
        }
}
def rounds2Score: (HList) => Int = {
        //strike appliqué sur la dernière manche
        case (10 :: _ :: HNil) :: ((b1: Int) :: (b2: Int) :: (b3: Int) :: HNil) :: HNil => 10 + b1 + b2   +   b1 + b2 + b3
        //deux strike de suite (cas spécifique = les boules comptées doubles sont réparties sur deux manches différentes)
        case (m1: Round) :: (m2: Round) :: ((b1: Int) :: (b2: Int) :: xs) :: tail if hasStrike(m1) && hasStrike(m2) => 10 + strike(m2) + b1 + rounds2Score(m2 :: (b1 :: b2 :: xs) :: tail)
        //strike simple = 10 + strike appliqué sur la manche d'après
        case (10 :: _ :: HNil) :: (m2: Round) :: tail => 10 + strike(m2) + rounds2Score(m2 :: tail)
     }
{% endhighlight %}


Jeu parfait
----------

{% highlight scala  %}
test("A perfect game with 10 + 10 + 10 + 10 + 10 + 10 + 10 + 10 + 10 + 10 + 10 + 10 + 10 = 300") {
    //ce test vérifie que deux strike de suite sont bien comptés ainsi que les strikes de dernière manche.
    //C'est sur ce dernier test que nous avons passé le + de temps
    Score(strike :: strike :: strike :: strike :: strike :: strike :: strike :: strike :: strike  :: lastStrike :: HNil) shouldEqual 300
}
{%endhighlight%}


Pro/Cons
-----------

**Les +**

* Il a été facile d'exprimer les types pour Round, LastRound, ainsi que pour le jeu.
* Le pattern matching appliqué sur les HList est simple et puissant, il nous permet de détecter facilement les strikes et les spares.

**Les -**

* Aucun scaladoc (au moins sur les Poly* et HList).
* La [documentation](https://github.com/milessabin/shapeless/wiki/Feature-overview:-shapeless-2.0.0) a été difficile à trouver, et elle ne s'applique pas à la version courante de Shapeless.
* Les erreurs de compilation ne sont pas détectées par l'IDE (IntelliJ),  il faut lancer les TU pour obtenir les erreurs de compilation, qui sont généralement des implicits manquants ou ambigüs.



Références :
-----------
* [Code source de la solution](https://gist.github.com/dgouyette/bbf941d34e1a14c901ea)
* [https://github.com/milessabin/shapeless](https://github.com/milessabin/shapeless)
* [Feature overview Shapeless](https://github.com/milessabin/shapeless/wiki/Feature-overview:-shapeless-2.0.0)
* [http://scala.org.ua/presentations/scala-polymorphism/](http://scala.org.ua/presentations/scala-polymorphism/)
