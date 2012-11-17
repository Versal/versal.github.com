---
layout: post
title: "Beyond the Reader Monad: Dependency Injection with Jellyfish"
date: 2012-11-16 10:37
comments: true
categories: frameworks
---

{% include JB/setup %}

One handy use of the reader monad is to build up a program that has a single external dependency which can be provided at the appropriate time.

In Scala a reader might look something like this:

{% highlight scala %}
trait Monad[A, F[_]] {
  def map[B](f: A => B): F[B]
  def flatMap[B](f: A => F[B]): F[B]
}

class Reader[E, A](g: E => A) extends Monad[A, ({type λ[α] = E => α})#λ] {
  def apply(e: E): A = g(e)
  def map[B](f: A => B): E => B = { e => f(g(e)) }
  def flatMap[B](f: A => (E => B)): E => B = { e => f(g(e))(e) }
}

object Reader {
  implicit def function1ToReader[E, A](g: E => A): Reader[E, A] = new Reader(g)
}
{% endhighlight %}

We can use this to compose a program from parts which each depend on the same external context:

{% highlight scala %}
import scala.xml.NodeSeq
import Reader.function1ToReader

val htmlHeader: String => NodeSeq =
  { title => <head><title>{title}</title></head> }

val htmlBody: String => NodeSeq =
  { title => <body><h1>{title}</h1></body> }

val html: NodeSeq => NodeSeq => NodeSeq =
  { header => body => <html>{header}{body}</html> }

val pageFromTitle: Reader[String, NodeSeq] =
  for {
    header <- htmlHeader
    body   <- htmlBody
    page   =  html(header)(body)
  } yield page // a program which, given a title, produces an HTML page

val page: NodeSeq = pageFromTitle("Reader Example")
println(page) // <html><head><title>Reader Example</title></head><body> ...
{% endhighlight %}

## The problem

This is an easy way to do dependency injection, but it has a drawback: the dependency has to be a single thing.  This makes it hard to build a program from components that have different depenencies.

Consider if we change `htmlBody` to take a `Date` instead of a `String`:

{% highlight scala %}
val htmlBody: Date => NodeSeq = date => <body><h1>{date.toString}</h1></body>
{% endhighlight %}

The for expression above will no longer compile, because the reader is an instance of `Reader[String, NodeSeq]`, so both its `map` and `flatMap` methods expect to be passed a function which takes a `String`, not a `Date`:

{% highlight scala %}
[error]  found   : java.util.Date => scala.xml.NodeSeq
[error]  required: String => ?
[error]   body   <- htmlBody
{% endhighlight %}

## The solution

What we really want is a way to construct a program, and keep track of all the different dependencies as we go.  At the end, we'll have a sort of arbitrary-order curried function that we can interpret recursively, providing each dependency as needed.

{% highlight scala %}
val simpleProgram: Program =
  program {
    val bar: Bar = read[Bar] // retrieve the `Bar` dependency
    val foo: Foo = read[Foo] // retrieve the `Foo` dependency
    Return("foo is " + foo.x + ", bar is " + bar.x)
  }
{% endhighlight %}

This is what Jellyfish gives us.

## How it works

A Jellyfish program is represented as an instance of the `Program` trait, which has two implementations:

{% highlight scala %}
case class Return(a: Any) extends Program
case class With[A](c: Class[A], f: A => Program) extends Program
{% endhighlight %}

The `read` function, which wraps Scala's `shift` function, takes a generic function of type `A => Program` and wraps it in a `With` which tracks the type of `A`.  This can happen an arbitrary number of times, resulting in a data structure analogous to a curried function.

Ignoring some of the wrappers, this:

{% highlight scala %}
val bar: Bar = read[Bar] // retrieve the `Bar` dependency
val foo: Foo = read[Foo] // retrieve the `Foo` dependency
Return("foo is " + foo.x + ", bar is " + bar.x)
{% endhighlight %}

becomes:

{% highlight scala %}
bar: Bar => {
  val foo: Foo = read[Foo] // retrieve the `Foo` dependency
  Return("foo is " + foo.x + ", bar is " + bar.x)
}
{% endhighlight %}

which becomes:

{% highlight scala %}
bar: Bar => {
  foo: Foo => {
    Return("foo is " + foo.x + ", bar is " + bar.x)
  }
}
{% endhighlight %}

which is a curried function with two dependencies.

An interpreter is then built to unwrap each nested `With`, extract the function of type `A => Program`, provide the appropriate instance of `A`, and continue until the program completes with a `Return`.


## An example

First, write a program which retrieves dependencies via the `read` function:

{% highlight scala %}
case class Foo(x: Int)
case class Bar(x: String)

object SimpleProgram {

  import com.versal.jellyfish.{program, Program, read, Return}

  // create a program with some dependencies
  val simpleProgram: Program =
   program {
      val bar: Bar = read[Bar] // retrieve the `Bar` dependency
      val foo: Foo = read[Foo] // retrieve the `Foo` dependency
      Return("foo is " + foo.x + ", bar is " + bar.x)
    }

}
{% endhighlight %}

Second, write an interpreter which provides the dependencies to the program:

{% highlight scala %}
object SimpleInterpreter {

  import com.versal.jellyfish.{classy, Program, Return, With}

  val foo = Foo(42)
  val bar = Bar("baz")

  // run a program, injecting dependencies as needed
  def run(p: Program): Any =
    p match {
      case With(c, f) if c.isA[Foo] => run(f(foo)) // inject the `Foo` dependency
      case With(c, f) if c.isA[Bar] => run(f(bar)) // inject the `Bar` dependency
      case Return(a)                => a           // all done - return the result
    }

}
{% endhighlight %}

Third, run the interpreter:

{% highlight scala %}
val result = SimpleInterpreter.run(SimpleProgram.simpleProgram)
println(result) // prints "foo is 42, bar is baz"
{% endhighlight %}

## Use it

Jellyfish lives on GitHub at [github.com/Versal/jellyfish](https://github.com/Versal/jellyfish).  To use it in your own project, [enable continuations](http://www.scala-sbt.org/release/docs/Detailed-Topics/Compiler-Plugins.html#continuations-plugin-example) and add the Jellyfish library to your _build.sbt_ file:

{% highlight scala %}
libraryDependencies += "com.versal" %% "jellyfish" % "0.1-RC1"
{% endhighlight %}
