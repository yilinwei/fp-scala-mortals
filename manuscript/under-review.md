

# For Comprehensions

Scala's `for` comprehension is heavily used in FP --- it is the ideal
abstraction for pure sequential code. But most Scala developers only
use `for` to loop over collections and are not aware of its full
potential.

In this chapter, we're going to relearn the principles of `for` and
how cats can help us to write cleaner code. This chapter doesn't try
to write pure programs and the techniques can be immediately applied
to a non-FP codebase.

## Syntax Sugar

Scala's `for` is just a simple rewrite rule, also called *syntax
sugar*, that doesn't have any contextual information.

To see what a `for` comprehension is doing, we use the `show` and
`reify` feature in the REPL to print out what code looks like after
type inference.

{lang="text"}
~~~~~~~~
scala> import scala.reflect.runtime.universe._
scala> val a, b, c = Option(1)
scala> show { reify {
         for { i <- a ; j <- b ; k <- c } yield (i + j + k)
       } }

$read.a.flatMap(
  ((i) => $read.b.flatMap(
    ((j) => $read.c.map(
      ((k) => i.$plus(j).$plus(k)))))))
~~~~~~~~

There is a lot of noise due to additional sugarings (e.g. `+` is
rewritten `$plus`, etc). We'll skip the `show` and `reify` for brevity
when the REPL line is `reify>`, and manually clean up the generated
code so that it doesn't become a distraction.

{lang="text"}
~~~~~~~~
reify> for { i <- a ; j <- b ; k <- c } yield (i + j + k)

a.flatMap {
  i => b.flatMap {
    j => c.map {
      k => i + j + k }}}
~~~~~~~~

The rule of thumb is that every `<-` (called a *generator*) is a
nested `flatMap` call, with the final generator a `map` containing the
`yield`.

### Assignment

We can assign values inline like `ij = i + j` (a `val` keyword is not
needed).

{lang="text"}
~~~~~~~~
reify> for {
         i <- a
         j <- b
         ij = i + j
         k <- c
       } yield (ij + k)

a.flatMap {
  i => b.map { j => (j, i + j) }.flatMap {
    case (j, ij) => c.map {
      k => ij + k }}}
~~~~~~~~

A `map` over the `b` introduces the `ij` which is flat-mapped along
with the `j`, then the final `map` for the code in the `yield`.

A> `val` doesn't have to assign to a single value, it can be anything
A> that works as a `case` in a pattern match. The same is true for
A> assignment in `for` comprehensions.
A> 
A> {lang="text"}
A> ~~~~~~~~
A> scala> val (first, second) = ("hello", "world")
A> first: String = hello
A> second: String = world
A> 
A> scala> val list: List[Int] = ...
A> scala> val head :: tail = list
A> head: Int = 1
A> tail: List[Int] = List(2, 3)
A> ~~~~~~~~
A> 
A> But be careful that you don't miss any cases or you'll get a runtime
A> exception (a *totality* failure).
A> 
A> {lang="text"}
A> ~~~~~~~~
A> scala> val a :: tail = list
A> scala.MatchError: List()
A> ~~~~~~~~

Unfortunately we [cannot assign a value before any generators](https://github.com/typelevel/scala/issues/143):

{lang="text"}
~~~~~~~~
scala> for {
         initial = getDefault
         i <- a
       } yield initial + i
<console>:1: error: '<-' expected but '=' found.
~~~~~~~~

but we can workaround it by defining a `val` outside the `for` or wrap
the initial assignment:

{lang="text"}
~~~~~~~~
scala> val initial = getDefault
       for { i <- a } yield initial + i

scala> for {
         initial <- Option(getDefault)
         i <- a
       } yield initial + i
~~~~~~~~

### Filter

It is possible to put `if` statements after a generator to filter
values by a predicate

{lang="text"}
~~~~~~~~
reify> for {
         i  <- a
         j  <- b
         if i > j
         k  <- c
       } yield (i + j + k)

a.flatMap {
  i => b.withFilter {
    j => i > j }.flatMap {
      j => c.map {
        k => i + j + k }}}
~~~~~~~~

Older versions of scala used `filter`, but `Traversable.filter`
creates new collections for every predicate, so `withFilter` was
introduced as the more performant alternative.

### For Each

Finally, if there is no `yield`, the compiler will use `foreach`
instead of `flatMap`, which is only useful for side-effects.

{lang="text"}
~~~~~~~~
reify> for { i <- a ; j <- b } println(s"$i $j")

a.foreach { i => b.foreach { j => println(s"$i $j") } }
~~~~~~~~

### Summary

The full set of methods supported by `for` comprehensions do not share
a common super type; each generated snippet is independently compiled.
If there were a trait, it would roughly look like:

{lang="text"}
~~~~~~~~
trait ForComprehendable[C[_]] {
  def map[A, B](f: A => B): C[B]
  def flatMap[A, B](f: A => C[B]): C[B]
  def withFilter[A](p: A => Boolean): C[A]
  def foreach[A](f: A => Unit): Unit
}
~~~~~~~~

If an implicit `cats.FlatMap[T]` is available for `T`, then `map` and
`flatMap` are available and `T` can be used in a `for` comprehension.

`cats.Monad` implements `cats.FlatMap`, so anything that is monadic
(i.e. has an implicit `Monad[T]`) can be used in a `for`. But just
because something can be used as the container in a `for`
comprehension does not mean it is monadic (e.g. `Future` is not
monadic). We'll learn the difference when we discuss *laws*.

`withFilter` and `foreach` are not concepts that are useful in
functional programming, so we won't discuss them any further.

A> It often surprises developers when inline `Future` calculations in a
A> `for` comprehension do not run in parallel:
A> 
A> {lang="text"}
A> ~~~~~~~~
A> import scala.concurrent._
A> import ExecutionContext.Implicits.global
A> 
A> for {
A>   i <- Future { expensiveCalc() }
A>   j <- Future { anotherExpensiveCalc() }
A> } yield (i + j)
A> ~~~~~~~~
A> 
A> This is because the `flatMap` spawning `anotherExpensiveCalc` is
A> strictly **after** `expensiveCalc`. To ensure that two `Future`
A> calculations begin in parallel, start them outside the `for`
A> comprehension.
A> 
A> {lang="text"}
A> ~~~~~~~~
A> val a = Future { expensiveCalc() }
A> val b = Future { anotherExpensiveCalc() }
A> for { i <- a ; j <- b } yield (i + j)
A> ~~~~~~~~
A> 
A> `for` comprehensions are fundamentally for defining sequential
A> programs. We will show a far superior way of defining parallel
A> computations in a later chapter.

## Unhappy path

So far we've only look at the rewrite rules, not what is happening in
`map` and `flatMap`. Let's consider what happens when the `for`
container decides that it can't proceed any further.

In the `Option` example, the `yield` is only called when `i,j,k` are
all defined.

{lang="text"}
~~~~~~~~
for {
  i <- a
  j <- b
  k <- c
} yield (i + j + k)
~~~~~~~~

If any of `a,b,c` are `None`, the comprehension short-circuits with
`None` but it doesn't tell us what went wrong.

A> How often have you seen a function that takes `Option` parameters but
A> requires them all to exist? An alternative to throwing a runtime
A> exception is to use a `for` comprehension, giving us totality (a
A> return value for every input):
A> 
A> {lang="text"}
A> ~~~~~~~~
A> def namedThings(
A>   someName  : Option[String],
A>   someNumber: Option[Int]
A> ): Option[String] = for {
A>   name   <- someName
A>   number <- someNumber
A> } yield s"$number ${name}s"
A> ~~~~~~~~
A> 
A> but this is verbose, clunky and bad style. If a function requires
A> every input then it should make its requirement explicit, pushing the
A> responsibility of dealing with optional parameters to its caller ---
A> don't use `for` unless you need to.

If we use `Either`, then a `Left` will cause the `for` comprehension
to short circuit with extra information, much better than `Option` for
error reporting:

{lang="text"}
~~~~~~~~
scala> val a = Right(1)
scala> val b = Right(2)
scala> val c: Either[String, Int] = Left("sorry, no c")
scala> for { i <- a ; j <- b ; k <- c } yield (i + j + k)

Left(sorry, no c)
~~~~~~~~

And lastly, let's see what happens with a `Future` that fails:

{lang="text"}
~~~~~~~~
scala> import scala.concurrent._
scala> import ExecutionContext.Implicits.global
scala> for {
         i <- Future.failed[Int](new Throwable)
         j <- Future { println("hello") ; 1 }
       } yield (i + j)
scala> Await.result(f, duration.Duration.Inf)

java.lang.Throwable
~~~~~~~~

The `Future` that prints to the terminal is never called because, like
`Option` and `Either`, the `for` comprehension short circuits.

Short circuiting for the unhappy path is a common and important theme.
`for` comprehensions cannot express resource cleanup: there is no way
to `try` / `finally`. This is good, in FP it puts a clear ownership of
responsibility for unexpected error recovery and resource cleanup onto
the `Monad`, not the business logic.

## Gymnastics

Although it's easy to rewrite simple sequential code as a `for`
comprehension, sometimes we'll want to do something that appears to
require mental summersaults. This section collects some practical
examples and how to deal with them.

### Fallback

Let's say we are calling out to a method that returns an `Option` and
if it's not successful we want to fallback to another method, like
when we're using a cache:

{lang="text"}
~~~~~~~~
def getFromReddis(s: String): Option[String] = ...
def getFromSql(s: String): Option[String] = ...

getFromReddis(key) orElse getFromSql(key)
~~~~~~~~

If we have to do this for an asynchronous version of the same API

{lang="text"}
~~~~~~~~
def getFromReddis(s: String): Future[Option[String]]
def getFromSql(s: String): Future[Option[String]]
~~~~~~~~

then we have to be careful not to do extra work because

{lang="text"}
~~~~~~~~
for {
  cache <- getFromReddis(key)
  sql   <- getFromSql(key)
} yield cache orElse sql
~~~~~~~~

will run both queries. We can pattern match on the first result but
the type is wrong

{lang="text"}
~~~~~~~~
for {
  cache <- getFromReddis(key)
  res   <- cache match {
             case Some(_) => cache !!! wrong type !!!
             case None    => getFromSql(key)
           }
} yield res
~~~~~~~~

We need to wrap the `cache` value as a `Future` again:

{lang="text"}
~~~~~~~~
for {
  cache <- getFromReddis(key)
  res   <- cache match {
             case Some(_) => Future.successful(cache)
             case None    => getFromSql(key)
           }
} yield res
~~~~~~~~

The call to `Future.successful` is like the `Option` constructor and
wraps / lifts / binds the value into the `for` container. Every
`Monad` in cats has a method called `pure` on its companion, adding
some consistency to this pattern.

If functional programming was like this all the time, it'd be a
nightmare. Thankfully these tricky situations are the corner cases.

A> We could play code golf and write
A> 
A> {lang="text"}
A> ~~~~~~~~
A> getFromReddis(key) orElseM getFromSql(key)
A> ~~~~~~~~
A> 
A> by defining <https://github.com/typelevel/cats/issues/1625> but it can
A> be a cognitive burden to remember all these monadic variants.

## TODO Monad Transformers

