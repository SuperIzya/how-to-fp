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

Apart from being strictly aesthetic and cool, there are major advantages to use FP:

* Functional code is easy reasoning about due to:
    * [referential transparency][RP] 
    (gist: An expression is called referentially transparent if it can be replaced with its corresponding value without changing the program's behavior) which means in less fancy words that on any level it is easy to understand what's going on in the code
    * types are more semantic //have more semantic meaning?//
    * pure functions tend to be smaller than non pure (about pure functions later)
    * functional code - means declarative code, which in turn means, that all imperative code will be separated from the algorithms.
* Fewer bugs due to:
    * code is easier to reason about (see above)
    * since your code is much more like a mathematical model, the compiler will check your model, and the compilation will fail in case your model is not converging, or there are illegal (in current model) operations
    * logic written with pure functions is easier to test
* Easier to spot and deeper levels of abstractions due to when anything type-specific 
is removed from the algorithms, the algorithms are left bare-boned, unobstructed. And then it is much simpler to spot patterns and abstractions in there.
* It is always better to tell __what__ to do instead of __how__. Functional code tells computer __what__ to do ([declarative code](https://en.wikipedia.org/wiki/Declarative_programming)), while your regular C-like code is telling the machine __how__ to do what it should do ([imperative code](https://en.wikipedia.org/wiki/Imperative_programming)).

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
So let's get them out and pass this data as a parameter the algorithm:

```scala
trait Monoid[T] {
  def empty: T
  def combine(x: T, y: T): T
}
def sum[T](lst: List[T], m: Monoid[T]): T = lst.foldLeft(m.empty)(m.combine)
```
or with functional library [Cats][Cats] and further optimizations:

```scala
import cats.Monoid
import cats.implicits._
def sum[T: Monoid](lst: List[T]): T = lst.foldLeft(Monoid[T].empty)(_ |+| _)
```

and with further abstraction:

```scala
import cats.{Monoid, Foldable}
import cats.implicits._
def sum[T: Monoid, F[_]: Foldable](lst: F[T]): T = lst.foldLeft(Monoid[T].empty)(_ |+| _)
```
and then, having the last implementation of `sum` we can suddenly do this:
```scala
type G[T] = List[Option[T]]
implicit val fld: Foldable[G] = Foldable[List].compose[Option]
sum[Int, G](List(Some(1), Some(2), None, Some(4))) // 7
```

This may look like magic or gibberish (or both) , but I'll do my best to get you (at least closer) to understanding the whys and hows of this parlor trick. But do remember, this is a language, and as with any other language, you need to practice it if you want to think it.

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

The concept of pure function is of the most importance in FP. My favorite example of one such function is arithmetic operator `+`:
1. it does not change it arguments. After evaluating `2+3`, both `2` and `3` are the very same, they were not changed by the function
1. it returns a value (always). Doesn't matter how many times the function is called, for the same arguments set the result is  the same. The order of calls with different sets of arguments is also does not change the results of each individual evaluation.

When starting writing pure function, write full signature:
```scala
def foo[T, R](x: T): R
```
This will also help you later: since it is pure function, you know that it does nothing to the arguments (you may use them again), and it always returns value of type `R`. So when you see application of the function in the code, in order to get the idea what is going on here, you just need to see it's signature, without the need to dive into all the nuances of the implementation (hello, [referential transparency][RP]).

Result of evaluation of pure function depends only on arguments, so no internal (or external via closure) variables should be in the function. 

#### No loops. 
Loops are essentially imperative constructions, which also ties via closure internal mutable(!) variable to the algorithm inside it. 

Use iteration and transformation over collection. Both standard library and functional libraries ([Cats][Cats], [Scalaz](https://scalaz.github.io), etc.) have many specialized functions like `map`, `filter`, `fold`, etc. Among many benefits of using these functions are:

* increased readability of the code
* functional (declarative) way of iterating over elements
* for some types there can be more robust implementation of the same iterative function
* some non-collection types implement the same functions (like `map` for `Option`) with the same meaning 

Only when you have no other chose left, write recursive function with [tail call][TailCall]. This kind of recursions is stack safe, since it is transformed by the compiler to loop (oh irony, but also an example of [postponing the effects](#postpone_effects) - another perk of FP). BTW, all the library functions use [tail call][TailCall] recursions under the hood.

The function [sum](#magic) from example above may be rewritten with [@tailrec](https://www.scala-lang.org/api/2.13.1/scala/annotation/tailrec.html) as follows:
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

Good explanation of tail recursion can be found [here](https://www.scala-exercises.org/scala_tutorial/tail_recursion).

#### State

Another popular use case is calculations based on previous element(s), in other words state (very thorough discussion of the state and it's use in FP can be found [here](https://typelevel.org/cats/datatypes/state.html)). This is very counterintuitive part of FP: the state should be outside. It is against the rules to introduce mutable variable inside function: the function becomes non-pure, it has side-effect built-in. The solution is (as almost always) in the return type. The function will receive state and return new state and result:

```scala
type State = ???
type Res = ???
def foo(s: State): (State, Res) = ???
```

As you can see, within this implementation, `foo` is pure function, and given the same `State` will return essentially the same pair `(State, Res)`.

[to top][0]

## Immutability

![immutability](./gifs/immutability.gif)

Another cornerstone of FP is immutability. Think again of `2 + 3`. Neither `2`, nor `3` can't suddenly change to something else (same goes to the result `5`). Each pure function returns new instance. Except for when it returns `case object` or some `val`, pure function __always__ returns new instance. Arguments are __never__ mutated.

In the inner function `sumRec` (from recursion example), first argument is `List[T]` and the second is accumulator.

```scala
@tailrec
def sumRec(l: List[T], acc: T): T = {
    if(l.isEmpty) acc 
    else sumRec(l.tail, Monoid[T].combine(acc, l.head))
}
sumRec(lst, Modoid[T].empty)
```

The original list (passed to enclosing function `sum`) is not changes, nor does any of the sub-lists. This is also true to accumulator. 

Immutability of arguments and result has it advantages:

* the same instance can be shared indefinitely. One of the consequences of _this_ fact is that parallel access is trivial - no need to sync since only read is possible

* the order of functions calls depends only on types (they should match, dah!), since each function is pure function and the result depends only on arguments passed. This also means, that compiler may deduce a call tree and run some evaluations in parallel.

One may argue that always creating new objects adds pressure to the memory (in terms of usage and consumption) and to garbage collector. I will argue with that with the facts that:

* most of these immutable instances are short-lived instances. They are forgotten during the first generation (Gen-1). It is the mutable instances that usually survive to Gen-2. As for immutable long-live instances, most of those are application-level constants or singleton instances that lives through to Gen-3.

* As mentioned above, when using immutable values, sharing them became trivial. No need neither for defensive copy-on-share, nor for synchronization of access. Which is beneficial both for memory and GC.

* And as a last argument (in this article) in favor for immutability, since immutable object doesn't mutate (dah, again), GC does not need to take it as root on the next scan. 

More about it [here](https://stackoverflow.com/questions/35384393/how-do-immutable-objects-help-decrease-overhead-due-to-garbage-collection) and [here](https://www.reddit.com/r/programming/comments/1i3738/is_immutability_good_for_gc_performance/). And, from Oracle [docs](https://docs.oracle.com/javase/tutorial/essential/concurrency/immutable.html):

> The impact of object creation is often overestimated, and can be offset by some of the efficiencies associated with immutable objects. These include decreased overhead due to garbage collection, and the elimination of code needed to protect mutable objects from corruption.

## Function - first class citizen
Each function has it's type, which consists of types of the arguments and type of the result:
```scala
def foo(i: Int, d: Double): String = ??? // (Int, Double) => String
def bar(b: BigInt): Boolean = ??? // BigInt => Boolean
```

A function with a proper type can be passed to or returned from another function. Most probably you already have seen/used it. `Array.map`, `Array.sort`, etc. in JavaScript, C#, Scala, etc., all these functions in all these languages are accepting transformation/sort/etc. function as one of the parameters. If you've used one of those, you've used function as a first class citizen. And the next reasonable step is to assign function to a variable:

```scala
val foo: (Int, Double) => String = (i, d) => s"$i $d"
val baz: BigInt => Boolean = _.isValidInt
```





## Types

![types](./gifs/types.gif)





## Postpone effects

![postpone](./gifs/postpone.gif)



[to top][0]

All images are taken from movie [Dr. Strangelove or: How I Learned to Stop Worrying and Love the Bomb](https://www.imdb.com/title/tt0057012/)

[0]: #how-i-learned-to-stop-worrying-and-love-fp-in-scala
[RP]: https://en.wikipedia.org/wiki/Referential_transparency
[Cats]: https://typelevel.org/cats/
[TailCall]: https://en.wikipedia.org/wiki/Tail_call
