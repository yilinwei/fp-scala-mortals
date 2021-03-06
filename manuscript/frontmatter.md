
{frontmatter}

# Preface

This book is for Scala developers with an Object Oriented (OOP)
background who wish to learn the **Functional Programming** (FP)
paradigm.

Until now, Scala has lacked a practical introduction to FP. We do not
believe that learning Haskell should be a prerequisite. We also do not
accept that the merits of FP are obvious. Therefore, this book
justifies every concept with practical examples, in Scala.

We recommend [The Red Book](https://www.manning.com/books/functional-programming-in-scala) as optional further reading. It is a
textbook to learn the fundamentals and write your own FP library in
Scala, serving a different purpose than this book.

We also recommend [Haskell Programming from First Principles](http://haskellbook.com/) as
optional further reading. FP innovation has traditionally been in
Haskell because it is the academic standard. But Scala has gained much
more traction in industry and brings with it features, stability,
interop, powerful frameworks and a commercial ecosystem.

# Copyleft Notice

This book is **Libre** and follows the philosophy of [Free Software](https://www.gnu.org/philosophy/free-sw.en.html): you
can use this book as you like, you can redistribute this book and you
can distribute your own version. That means you can print it,
photocopy it, e-mail it, post it on social media, upload it to
websites, change it, translate it, remix it, delete bits, and draw all
over it. You can even sell it, although you should agree a royalty
share with the author or you'll not be getting an xmas card.

This book is **Copyleft**: if you change the book and distribute your
own version, you must also pass these freedoms to its recipients.

This book uses the [Creative Commons Attribution ShareAlike 4.0
International](https://creativecommons.org/licenses/by-sa/4.0/legalcode) (CC BY-SA 4.0) license. Attribution means you cannot
claim that you wrote the book.

All code examples in this book are separately [Apache 2.0](https://www.apache.org/licenses/LICENSE-2.0) licensed.

# Thanks

Diego Esteban Alonso Blas, Raúl Raja Martínez and Peter Neyens of 47
degrees for their help with understanding the principles of FP, cats
and freestyle. Yi Lin Wei and Zainab Ali for their tutorials at Hack
The Tower meetups.

Rory Graves and Ani Chakraborty for giving feedback on early drafts of
this text.

Juan Manuel Serrano for [All Roads Lead to Lambda](https://skillsmatter.com/skillscasts/9904-london-scala-march-meetup#video), Pere Villega for [On
Free Monads](http://perevillega.com/understanding-free-monads), Dick Wall and Josh Suereth for [For: What is it Good For?](https://www.youtube.com/watch?v=WDaw2yXAa50),
John de Goes for [A Beginner Friendly Tour](http://degoes.net/articles/easy-monads).

The helpul souls who helped explain the concepts needed to write the
example project [drone-dynamic-agents](https://github.com/fommil/drone-dynamic-agents/issues?q=is%3Aissue+is%3Aopen+label%3A%22needs+guru%22): Merlin Göttlinger, Edmund Noble,
Rob Norris, Adelbert Chang, Kai(luo) Wang, Michael Pilquist, Adam
Chlupacek, Pavel Chlupacek, Paul Snively.

# Practicalities

If you'd like to set up a project that uses the libraries presented in
this book, you will need to use a recent version of Scala with
FP-specific features enabled (e.g. in `build.sbt`):

{lang="text"}
~~~~~~~~
scalaVersion in ThisBuild := "2.12.2"
scalacOptions in ThisBuild ++= Seq(
  "-language:_",
  "-Ypartial-unification"
)
~~~~~~~~

and add the following dependencies to your project's settings:

{lang="text"}
~~~~~~~~
libraryDependencies ++= Seq(
  "io.circe" %% "circe-core",
  "io.circe" %% "circe-generic",
  "io.circe" %% "circe-parser"
).map(_ % "0.7.0") ++ Seq(
  "org.typelevel" %% "cats"     % "0.9.0",
  "com.spinoco"   %% "fs2-http" % "0.1.6"
)

resolvers += Resolver.sonatypeRepo("snapshots")
addCompilerPlugin("org.scalamacros" %  "paradise"  % "2.1.0" cross CrossVersion.full)
libraryDependencies += "com.47deg"  %% "freestyle" % "0.1.0-SNAPSHOT"
~~~~~~~~

In order to keep our snippets short, we will omit the `import`
section. Unless told otherwise, assume that all snippets have the
following imports:

{lang="text"}
~~~~~~~~
import cats._
import cats.implicits._
import freestyle._
import freestyle.implicits._
import fs2._
~~~~~~~~


