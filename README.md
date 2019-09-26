# How I learned not to worry and loved FP in Scala

Since i've recently struggled trough understanding of the basics of FP,
and since the most hard part was to start thinking functional, 
i've decided to share some tips how to make first steps easier.  

### Pillars of FP

There are several concepts in functional programming, that once mastered, you're half way there into FP (the hard half).
In the following part I will _what??_ give

#### Table of contents
* [Why we choose functional programming?](#why-we-choose-functional-programming)
* [Magic](#magic)
* [Always produce result. No exceptions](result.md#always-produce-result-no-exceptions)


## Why we choose functional programming?

![Why FP](./gifs/types-algebras.gif)

Apart from being strictly aesthetic and cool, 
there are major advantages to use FP:

* Functional code is easy reasoning about due to:
    * [referential transparency](https://en.wikipedia.org/wiki/Referential_transparency) 
    (gist: An expression is called referentially transparent if it can be replaced with its corresponding value without changing the program's behavior)
    which means in less fancy words that on any level it is easy to understand what's going on in the code
    * types are more semantic //have more semantic meaning?//
    * pure functions tend to be smaller than non pure (about pure functions later)
    * functional code - means declarative code, which in turn means, 
    that all imperative code will be separated from the algorithms.

* Fewer bugs due to:
    * code is easier to reason about
    * since your code is much more mathematical model, 
    the compiler will check your model, and the compilation will fail in case your model is not converging.
    * logic written with pure functions is easier to test
* Easier to spot and deeper levels of abstractions due to when anything type-specific 
is removed from the algorithms, the algorithms are left bare-boned, unobstructed. And then it is much simpler 
to spot patterns and abstractions in there.

[to top ^][0]

## Magic

![magic](./gifs/magic.gif)

(For more complete example [see here](https://typelevel.org/cats/typeclasses.html))

Let's say we have two functions:

```scala
def sumInt(lst: List[Int]): Int = lst.foldLeft(0)(_ + _)
def sumSet[A](lst: List[Set[A]]): Set[A] = lst.foldLeft(Set.empty[A])(_ union _)
```

Both of these functions are the same except for the type-specific seed element and method for combining two elements.
So let's get this out:
```scala
trait Monoid[T] {
  def empty: T
  def combine(x: T, y: T): T
}
def sum[T: Monoid](lst: List[T]): T = lst.foldLeft(Monoid[T].empty)(Monoid[T].combine)
```
or with cats and further abstraction:
```scala
import cats.{Monoid, Foldable}
import cats.implicits._
def sum[T: Monoid, F[_]: Foldable](lst: F[T]): T = lst.foldLeft(Monoid[T].empty)(_ |+| _)
```
and then, having the last function we can suddenly do this:
```scala
implicit val fld = Foldable[List].compose[Option]
sum(List(Some(1), Some(2), None, Some(4))) // 7
```

Magic!!

[to top](#table-of-contents)

## Always produce result. No exceptions

![Result](./gifs/result.gif)

Function should always produce a value.

