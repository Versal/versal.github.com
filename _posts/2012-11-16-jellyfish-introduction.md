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

<pre>
trait Monad[A, F[_]] {
  def map[B](f: A =&gt; B): F[B]
  def flatMap[B](f: A =&gt; F[B]): F[B]
}

class Reader[E, A](g: E =&gt; A) extends Monad[A, ({type λ[α] = E =&gt; α})#λ] {
  def apply(e: E): A = g(e)
  def map[B](f: A =&gt; B): E =&gt; B = { e =&gt; f(g(e)) }
  def flatMap[B](f: A =&gt; (E =&gt; B)): E =&gt; B = { e =&gt; f(g(e))(e) }
}

object Reader {
  implicit def function1ToReader[E, A](g: E =&gt; A): Reader[E, A] = new Reader(g)
}
</pre>

We can use this to compose a program from parts which each depend on the same external context:

<pre>
import scala.xml.NodeSeq
import Reader.function1ToReader

val htmlHeader: String =&gt; NodeSeq =
  { title =&gt; &lt;head&gt;&lt;title&gt;{title}&lt;/title&gt;&lt;/head&gt; }

val htmlBody: String =&gt; NodeSeq =
  { title =&gt; &lt;body&gt;&lt;h1&gt;{title}&lt;/h1&gt;&lt;/body&gt; }

val html: NodeSeq =&gt; NodeSeq =&gt; NodeSeq =
  { header =&gt; body =&gt; &lt;html&gt;{header}{body}&lt;/html&gt; }

val pageFromTitle: Reader[String, NodeSeq] =
  for {
    header &lt;- htmlHeader
    body   &lt;- htmlBody
    page   =  html(header)(body)
  } yield page // a program which, given a title, produces an HTML page

val page: NodeSeq = pageFromTitle("Reader Example")
println(page) // &lt;html&gt;&lt;head&gt;&lt;title&gt;Reader Example&lt;/title&gt;&lt;/head&gt;&lt;body&gt;&lt;h1&gt;Reader Example&lt;/h1&gt;&lt;/body&gt;&lt;/html&gt;
</pre>

## The problem

This is an easy way to do dependency injection, but it has a drawback: the dependency has to be a single thing.  This makes it hard to build a program from components that have different depenencies.

Consider if we change `htmlBody` to take a `Date` instead of a `String`:

<pre>
val htmlBody: Date =&gt; NodeSeq = date =&gt; &lt;body&gt;&lt;h1&gt;{date.toString}&lt;/h1&gt;&lt;/body&gt;
</pre>

The for expression above will no longer compile, because the reader is an instance of `Reader[String, NodeSeq]`, so both its `map` and `flatMap` methods expect to be passed a function which takes a `String`, not a `Date`:

<pre>
[error]  found   : java.util.Date =&gt; scala.xml.NodeSeq
[error]  required: String =&gt; ?
[error]   body   &lt;- htmlBody
</pre>

## The solution

What we really want is a way to construct a program, and keep track of all the different dependencies as we go.  At the end, we'll have a sort of arbitrary-order curried function that we can interpret recursively, providing each dependency as needed.

<pre>
val simpleProgram: Program =
  program {
    val bar: Bar = read[Bar]  // retrieve the `Bar` dependency
    val foo: Foo = read[Foo]  // retrieve the `Foo` dependency
    Return("foo is " + foo.x + ", bar is " + bar.x)
  }
</pre>

This is what Jellyfish gives us.

## How it works

A Jellyfish program is represented as an instance of the `Program` trait, which has two implementations:

<pre>
case class Return(a: Any) extends Program
case class With[A](c: Class[A], f: A =&gt; Program) extends Program
</pre>

The `read` function, which wraps Scala's `shift` function, takes a generic function of type `A =&gt; Program` and wraps it in a `With` which tracks the type of `A`.  This can happen an arbitrary number of times, resulting in a data structure analogous to a curried function.

Ignoring some of the wrappers, this:

<pre>
val bar: Bar = read[Bar]  // retrieve the `Bar` dependency
val foo: Foo = read[Foo]  // retrieve the `Foo` dependency
Return("foo is " + foo.x + ", bar is " + bar.x)
</pre>

becomes:

<pre>
bar: Bar =&gt; {
  val foo: Foo = read[Foo]  // retrieve the `Foo` dependency
  Return("foo is " + foo.x + ", bar is " + bar.x)
}
</pre>

which becomes:

<pre>
bar: Bar =&gt; {
  foo: Foo =&gt; {
    Return("foo is " + foo.x + ", bar is " + bar.x)
  }
}
</pre>

which is a curried function with two dependencies.

An interpreter is then built to unwrap each nested `With`, extract the function of type `A =&gt; Program`, provide the appropriate instance of `A`, and continue until the program completes with a `Return`.


## An example

First, write a program which retrieves dependencies via the `read` function:

<pre>
case class Foo(x: Int)
case class Bar(x: String)

object SimpleProgram {

  import com.versal.jellyfish.{program, Program, read, Return}

  // create a program with some dependencies
  val simpleProgram: Program =
   program {
      val bar: Bar = read[Bar]  // retrieve the `Bar` dependency
      val foo: Foo = read[Foo]  // retrieve the `Foo` dependency
      Return("foo is " + foo.x + ", bar is " + bar.x)
    }

}
</pre>

Second, write an interpreter provides the dependencies to the program:

<pre>
object SimpleInterpreter {

  import com.versal.jellyfish.{classy, Program, Return, With}

  val foo = Foo(42)
  val bar = Bar("baz")

  // run a program, injecting dependencies as needed
  def run(p: Program): Any =
    p match {
      case With(c, f) if c.isA[Foo] =&gt; run(f(foo)) // inject the `Foo` dependency and continue
      case With(c, f) if c.isA[Bar] =&gt; run(f(bar)) // inject the `Bar` dependency and continue
      case Return(a)                =&gt; a           // all done - return the result
    }

}
</pre>

Third, run the interpreter:

<pre>
val result = SimpleInterpreter.run(SimpleProgram.simpleProgram)
println(result) // prints "foo is 42, bar is baz"
</pre>

## Use it

Jellyfish lives on GitHub at [github.com/Versal/jellyfish](https://github.com/Versal/jellyfish).  To use it in your own project, [enable continuations](http://www.scala-sbt.org/release/docs/Detailed-Topics/Compiler-Plugins.html#continuations-plugin-example) and add the Jellyfish library to your _build.sbt_ file:

<pre>
libraryDependencies += "com.versal" %% "jellyfish" % "0.1-RC1"
</pre>
