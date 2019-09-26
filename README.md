# How I learned to stop worrying and love FP in Scala

Since i've recently struggled trough understanding of the basics of FP,
and since the most hard part was to start thinking functional, 
i've decided to share some tips how to make first steps easier.  

### Pillars of FP

There are several concepts in functional programming, that once mastered, you're half way there into FP (the hard half).
In the following part I will _what??_ give

#### Table of contents
* [Why we choose functional programming?](#why-we-choose-functional-programming)
* [Magic](#magic)
* [Always produce result. No exceptions](#always-produce-result-no-exceptions)
* [Pure functions](#pure-functions)


## Why we choose functional programming?

![Why FP](./gifs/types-algebras.gif)

Apart from being strictly aesthetic and cool, 
there are major advantages to use FP:

* Functional code is easy reasoning about due to:
    * [referential transparency][RP] 
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

[to top][0]

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

[to top][0]

## Always produce result. No exceptions

![Result](./gifs/result.gif)

Function should always produce a value. Whenever there is a possibility of non-value result (exception, void, undefined, null, etc.), it should be incorporated in the result type. Such types include but not limited to:
```scala
Option[T]
Try[T]
JsResult[T]
Future[T]
//etc.
```
And of course other such types may be introduced as aliases via `type` or as case-classes:
```scala
type Result[T] = Either[String, T]
```

Functional approach not only better because it is functional, but also it is more robust in general case and in the edge cases gives more control over error handling to developer.

#### Exception is expensive

Whenever exception is thrown, runtime have to rollback the stack, collect a lot of data along the way, while all the functional types just stops the following execution with the value (since it is absent).

#### Easier error handling

Since error is allowed as result, we can work with errors same way as with 'valid' result, e.g. transform to default value, collect all errors from many executions, define retry strategy, etc. without excessive code branching as in case of exception/null/undefined based error-handling. 

[to top][0]

## Pure functions

![pure](./gifs/pure.gif)

The concept of pure function is of the most importance in FP. My favorite example of one such function is arithmetical operator `+`:
1. it does not change it arguments. After evaluating `2+3`, both `2` and `3` are the very same, they were not changed by the function
1. it returns a value (always). Doesn't matter how many times the function is called, the result is  the same. The order of calls with different sets of arguments is also does not change the results of each individual evaluation.

When starting writing pure function, write full signature:
```scala
def foo[T, R](x: T): R
```
This will also help you later. Since it is pure function, you know that it does nothing to the arguments (you may use them again), and it always returns type `R`. So when you see application of the function in the code, you just need to see it's signature, without the need to dive into all the nuances of the implementation. Hello, [referential transparency][RP].

Result of evaluation of pure function depends only on arguments, so no internal (or external via closure) variables should be in the function. Don't use loops. Use iteration and transformation over collection. Or, if it is impossible, write [tail recursive function](https://en.wikipedia.org/wiki/Tail_call).

For example our [sum](#magic) may be rewritten with[`@tailrec`](https://www.scala-lang.org/api/2.13.1/scala/annotation/tailrec.html) as follows:
```scala
def sum[T: Monoid](lst: List[T]): T = {
	@tailrec
	def sumRec(l: List[T], acc: T): T = {
		if(l.isEmpty) acc
		else sumRec(l.tail, Monoid[T].combine(acc, l.head))
	}
	
	sumRec(lst, Modoid[T].empty)
}
```



[to top][0]

[0]: #how-i-learned-not-to-worry-and-loved-fp-in-scala
[RP]: https://en.wikipedia.org/wiki/Referential_transparency