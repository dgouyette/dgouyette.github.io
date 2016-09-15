---
layout: post
title:  "Introduction pratique aux Macros Scala"
date:   2016-09-12 06:00:00
categories: scala
---

* TOC
{:toc}

## Que sont les macros ?

Ce sont des fonctions appelées lors de la compilation du projet, le programmeur a alors accès aux API du compilateur.
Un arbre syntaxique abstrait est produit qui sera ajouté au code annoté.
On les utilise pour produire du code rébarbatif à écrire (boilerplate), analyser du code ou vérifier les types.

Quelques exemples :

* Génération de reader / writer json : [play-json](https://github.com/playframework/playframework/tree/master/framework/src/play-json/src/main/scala/play/api/libs/json),
* [shapeless.Generic](https://github.com/milessabin/shapeless/blob/master/core/src/main/scala/shapeless/generic.scala), cela permet de passer facilement d'une case class à une HList,
* [macwire](https://github.com/adamw/macwire) Librairie d'injection de dépendance
* [Monocle](https://github.com/julien-truffaut/Monocle) Génère des lenser de case classes

Cet article vous présentera les macros Scala à travers deux exemples.

## 1er cas : afficher les infos Git à la compilation

Celui-ci s'inspire du projet [sbt-buildinfo](https://github.com/sbt/sbt-buildinfo).
L'objectif est de récupérer les informations courantes d'un projet sous GIT (commit courant, branche, commiter...) lors de la compilation.

Exemples d'utilisations de celle-ci :

~~~ scala
GitInfo.lastRevCommitAuthor()//Damien GOUYETTE <damien.gouyette @ gmail.com>
GitInfo.currentBranch()//master
GitInfo.lastRevCommitMessage()//the commit message
GitInfo.lastRevCommitTime()//Wed Jul 27 09:15:52 UTC 2016
GitInfo.lastRevCommitName()//f6b8219c6920a70539566cf740c3aabae839b10a
~~~

### Structure du build d'un projet de macro

#### Structure du projet :

~~~ scala
+ webapp/ //pour tester la macro un projet play en compilation continue
+ webapp/macros/ //le projet contenant le code de la macro
- build.sbt
~~~

#### build.sbt :

~~~scala
lazy val macros = project //scala macros équivalent à project in file("macros")

//La compilation du projet play dépend de la compilation du projet macros
lazy val root = (project in file(".")).enablePlugins(PlayScala).dependsOn(macros)
~~~

### Whitebox vs blackbox

L'un nécessite de lire l'implémentation pour comprendre le type de retour de la macro.
Il est possible pour la macro whitebox de retourner un type plus précis que celui indiqué sur l'API.

La [documentation des macros Scala](http://docs.scala-lang.org/overviews/macros/blackbox-whitebox.html) indique que **seule la blackblox sera standardisée**.

Dans nos exemples nous n'utiliserons que blackbox

### API de la macro

Celle ci référence l'implémentation tout en spécifiant le type de retour

~~~scala
import scala.language.experimental.macros
def lastRevCommitMessage(): String = macro lastRevCommitMessage_impl
~~~

### Implémentation de la macro

Le type de retour de la méthode d'implémentation est **c.Expr[T]**

Le type **T** doit être le même que celui indiqué sur l'API, cela peut être String, Any...
Expr est un wrapper d'AST (abstract syntax tree) tout en le taggant avec son type

La méthode lastRevCommitName(git) retourne un String ici :  "59adc7948a9aa4014fe219c5618b9405e097563c"

**showRaw** permet de visualiser l'AST brut sous forme de case classes
Dans notre cas, cela nous retourne :

~~~scala
//L'AST d'une instance de String
Expr(Literal(Constant("59adc7948a9aa4014fe219c5618b9405e097563c")))
~~~

**showCode** permer lui de voir le pseudo code généré

~~~scala
println(showRaw(c.Expr[String](q"""${lastRevCommitName(git)}""")))
//Expr(Literal(Constant("59adc7948a9aa4014fe219c5618b9405e097563c")))

println(showCode(q"""${lastRevCommitName(git)}"""))
//59adc7948a9aa4014fe219c5618b9405e097563c
~~~

#### L'implémentation complète :

~~~scala
def lastRevCommitMessage_impl(c: blackbox.Context)(): c.Expr[String] = {
  import c.universe._
  val git:org.eclipse.jgit.api.Git = Git.wrap(loadGitRepositoryfromEnclosingSourcePosition(c))
  c.Expr[String](q""" ${lastRevCommitMessage(git)} """)
}

//Récupères le repo git correspondant au projet utilisant la macro
//récupéré via  c.enclosingPosition
private def loadGitRepositoryfromEnclosingSourcePosition(c: blackbox.Context): Repository = {
  new FileRepositoryBuilder()
    .findGitDir(c.enclosingPosition.source.file.file)
    .setMustExist(true)
    .build()
}

private def lastRevCommitMessage(git: Git): String = git.log().call().toList match {
    case h :: _ => h.getFullMessage
    case Nil => "N/A"
}
~~~



Deuxième cas : générer des méthodes de conversion implicite.
--------

Dans cette partie, nous allons générer des méthodes de conversions implicites sur l'objet compagnon d'une case class, afin de passer d'un UnionXType à UnionX.

### Qu'est ce qu'un Union Type ?

Se dit d'une variable pouvant être de types différents (définis à l'avance), mais un seul type à la fois.

### Exemples d'utilisation :

~~~scala
case class Union2[T1, T2](v1: Option[T1], v2: Option[T2])
//up to 22

@union
class Union2Type[T1, T2]

//usages
val stringOrInt        : Union2[String, Int]          = Union2[String, Int](Some("0"), None)
val stringOrIntBoolean : Union3[String, Int, Boolean] =  Union3[String, Int, Boolean](None, None, false)
~~~

Pour convertir le type **UnionXType** vers  **UnionX**, nous devons écrire des **méthodes de conversions** de **T1** vers **Union2[T1, T2]**

Pour que cela soit fait automatiquement, nous ajouterons le mot clé *implicit*.


### Exemple de classes & méthodes devant être générées :

~~~scala
class Union2Type[T1, T2] extends scala.AnyRef {
  implicit def toUnion0(t: T1) = Union2[T1, T2](Some(t), None);
  implicit def toUnion1(t: T2) = Union2[T1, T2](None, Some(t))
}

class Union3Type[T1, T2, T3] extends scala.AnyRef {
  implicit def toUnion0(t: T1) = Union3[T1, T2, T3](Some(t), None, None);
  implicit def toUnion1(t: T2) = Union3[T1, T2, T3](None, Some(t), None);
  implicit def toUnion2(t: T3) = Union3[T1, T2, T3](None, None, Some(t))
}
~~~

### Test unitaire correspondant :

~~~scala
"computed value > 0 " should "be a '0' String" in {
    val stringOrInt = new Union2Type[String, Int]
    import stringOrInt._

    def computeValue(x: Int): Union2[String, Int] = {
      if (x > 0) "0"
      else 1
    }

    val result = computeValue(5)

    result.v1 shouldBe Some("0")
    result.v2 shouldBe None
  }
~~~

### scala.annotation.StaticAnnotation
La méthode présentée dans la première partie, ne permet de générer que des implémentations de méthodes dont le type de retour est plus ou moins défini à l'avance.
Pour ajouter des méthodes sur une classe, nous devons l'annoter avec une annotation héritant de **StaticAnnotation**.

 Usages de StaticAnnotation dans Scala lui-même :

 * deprecated,
 * transient,
 * serializable...


#### API de la macro

Cette fois ci, nous devons également fournir une API pour notre macro devant hériter de StaticAnnotation

La variable **annottees** représente les éléments annotés.

~~~scala
class union extends StaticAnnotation {
  def macroTransform(annottees: Any*) = macro union.impl
}
~~~

#### Implémentation de la macro

#### Quasiquotes & Pattern matching

**Quasiquotes** est une notation permettant de manipuler plus simplement un arbre syntaxique scala.

~~~scala
q"""println("hello world")"""
~~~

sera équivalent à

~~~scala
Apply(Ident(TermName("println")), List(Literal(Constant("hello world"))))
~~~

La **pattern matching** simplifie la **récupéreration des caractéristiques de l'élément annoté** (nom, types, champs, parent et corps de la classe).

Dans l'exemple suivant, nous la réécrivons à l'identique.

Si l'élément annoté ne correspond pas au pattern attendu (case class par exemple), une erreur sera alors remontée à la compilation.

~~~scala
annottees.map(_.tree).toList match {
  case q"class $name[..$tpes](..$lFields) extends ..$parents  { ..$body }" :: Nil =>
  q"""class $name[..$tpes](..$lFields) extends ..$parents   {
    ..$body
  }
  """
  case _ => c.abort(c.enclosingPosition, "union annotation only works on class")
}
~~~

**Can't unquote Seq[c.universe.DefDef], consider using ..** : Erreur remontée lorsque que l'on essaye *d'unquoter* un élément itérable sans le préfixer de **..$**

~~~scala
//Retourne une valeur si x et y correspondent
def argByXY(idx: Int, i: Int) = if (idx == i) {
  q"""Some(t)"""
}
else {
  q"""None"""
}
~~~

Names (TermName ou TypeName) sont des wrappers de String.

* TermName = noms de termes (objects ou members)
* TypeName = types (classes, traits ou type members).

~~~scala
case q"case class $name[..$tpes](..$lFields) extends ..$parents  { ..$body }" :: Nil =>
    val typesName: Seq[c.universe.TypeName] = tpes.map { case TypeDef(_, typeName, _, _) => typeName }
    //typesName = List(T1, T2)

    q"""
      class $name[..$tpes](..$lFields) extends ..$parents   {
        ..$body
       ..${
      typesName.zipWithIndex.map { case (currentTypeName, y) =>

        // Une méthode de conversion par type présent sur la classe annotée
        val conversionMethodName = TermName("toUnion" + y)

        val unionTermName = TermName("Union" + tpes.size)

        val constructorArgs = for (x <- typesName.indices) yield q"""${argByXY(x, y)}"""

        //Quasiquotes dans Quasiquotes permet ici de générer les méthodes de conversion en boucle  
        q"""
         implicit def $conversionMethodName(t: $currentTypeName) = $unionTermName[..$typesName](..$constructorArgs);
         """
      }
    }
      }
    """
~~~

## Implementation de méthode vs StaticAnnotation

Le premier cas pratique illustre la possibilité de générer l'implémentation d'une méthode et le second, d'ajouter des méthodes sur une classes (mais pas que).

La mise en place du premier cas pratique a été plus simple.
Pas besoin d'ajouter le plugin de compilation scalamacros-paradise.

Dans les deux cas, l'IDE (IntelliJ) a un peu de mal à s'y retrouver (erreurs de compilation).
Le code est plutôt compliqué à écrire et à comprendre pour ceux qui comme moi ne parlent pas l'AST couramment

Martin ODERSKY a  annoncé lors du dernier Scala Days que les macros seraient abandonnés (fonctionnalité expérimentale).

Mais le besoin de générer du code est un vrai besoin. Scala-meta est là pour pallier à ce genre de problème.

En savoir +
----------
*  [Code source du premier cas pratique : git-info](https://github.com/dgouyette/git-info)
*  [Définition d'un type union (Wikipedia)](https://en.wikipedia.org/wiki/Union_type)
*  [Documentation générale sur les macros](http://docs.scala-lang.org/overviews/macros/overview.html)
*  [l'AST et les macros ici](http://docs.scala-lang.org/overviews/reflection/annotations-names-scopes.html)
* [Scala meta slides d'Eugene Burmako](http://scalamacros.org/paperstalks/2016-06-17-Metaprogramming20.pdf)
