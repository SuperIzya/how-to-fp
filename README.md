# How I learned to stop worrying and love FP in Scala

![pure](./gifs/pure.gif)

Since I've recently struggled trough understanding of the basics of FP, and since the hardest part was to start thinking functional, I've decided to share some tips how to make first steps easier.  

#### Disclaimer

Among first thing to learn about functional programming, is that functional code is without IO nor any side-effects and it is immutable. Quite obviously, purely functional program will be absolutely useless, so I wouldn't declare FP the new silver bullet. But it is a good way to implement complex business-logic or core functionality. 

_There are several libraries like [Cats](Cats) or [Scalaz](Scalaz) defining some helpful types and functions for FP. I personally prefer [Cats](Cats), but the same result can be achieved with [Scalaz](Scalaz) or any other mature FP library._

#### Table of contents
* [Why we choose functional programming?](#why-we-choose-functional-programming)
* [Magic](#magic)
* [Always produce result. No exceptions](#always-produce-result-no-exceptions)
* [Pure functions](#pure-functions)
* [Immutability](#immutability)
* [Function - first class citizen](#function---first-class-citizen)
* [Types](#types)
* [Postpone effects](#postpone-effects)


## Why we choose functional programming?

Functional programming is a programming paradigm based on and very close to [lambda calculus](https://en.wikipedia.org/wiki/Lambda_calculus), which is

> ...formal system in mathematical logic for expressing computation...
> It is a universal model of computation that can be used to simulate any Turing machine.

Apart from the fact that functional code looks a lot like definition of mathematical model of the problem, there are other major advantages to use FP:

* Functional code is easy reasoning about due to [referential transparency][RP]: a function is referential transparent if by replacing the function call in code by the result of this call (given that the arguments are known), the behavior of the code will stay the same. 

  ```scala
  def add(x: Int, y: Int): Int = x + y	
  ```

  Function `add` is referentially transparent. Every call to this function can be replace by its result. 

  It is easier to understand this kind of code. Referentially transparent function is dependent only on it's arguments and it produces only one result, which is returned. No change is introduced to the system by call to this function. 

  This also means that no matter the scale, it takes about the same amount of effort to understand the code.

* Fewer bugs due to:
    * code is easier to reason about (see above)
    * since the code is much more like a mathematical model, the compiler can check it more thorough. The compilation will fail in case your model is not converging, or there are illegal (in current model) operations
    
* It is almost always better to tell __what__ to do instead of __how__. By telling __what__ to do, you're using higher level of abstractions. Smart people dedicated a lot of time and effort to implement the abstraction in efficient, precise and correct way. In rare cases, when you need something better, you'll dedicate all the resources to make it the best way possible. You will not write your own web server just for home page. But in case you need something special, it will become one of _core businesses_. 

    Functional code tells computer __what__ to do ([declarative code](https://en.wikipedia.org/wiki/Declarative_programming)), while your regular C-like code is telling the machine __how__ to do what it should do ([imperative code](https://en.wikipedia.org/wiki/Imperative_programming)).

[to top][0]

## Magic

I felt in love with functional paradigm in Scala because of the ability to build type-independent algorithms on simple cases, and then, with a pinch of magic dust, use totally unexpected types in those algorithms. All of this with static type-safety, compile-time optimizations and other goodies.

I will provide a short example here and i will give a sketchy explanation also. My hopes are, that after reading this article, the reader will be able to understand the example fully (since it is very simple). 

Given, with functional library [Cats](Cats), function `all` defined as:

```scala
import cats.{Monoid, Foldable}
import cats.implicits._
def all[T: Monoid, F[_]: Foldable](lst: F[T]): T = lst.foldLeft(Monoid[T].empty)(_ |+| _)
```

which allows us to use it as:

```scala
all(List(1, 2, 3, 4)) // 10
all(List(Set(1, 2, 3), Set(3, 4), Set(5))) // Set(1, 2, 3, 4, 5)
```

Then, if we define new type

```scala
type G[T] = List[Option[T]]
```

and add a pinch of magic

```scala
implicit val fld: Foldable[G] = Foldable[List].compose[Option]
```

 we can use the `all` function with this new type:

```scala
all[Int, G](List(Option(1), Option(2), None, Option(4))) // 7
```

##### Explanation

For more complete example and thorough explanation [see here](https://typelevel.org/cats/typeclasses.html). Here I'll give brief explanation in hope, that all the reset you will understand after finishing the article. 

Let's say we have two functions:

```scala
def allInt(lst: List[Int]): Int = lst.foldLeft(0)(_ + _)
def allSet[A](lst: List[Set[A]]): Set[A] = lst.foldLeft(Set.empty[A])(_ union _)
```

Both of these functions are the same except for the type-specific seed element and a method for combining two elements. In mathematics there is a special name for these pairs (seed - combine) - [Groups](https://en.wikipedia.org/wiki/Group_(mathematics)). In FP they are called _Monoid_.
So let's get them out and pass this data as a parameter the algorithm (to the function `all`):

```scala
trait Monoid[T] {
  def empty: T
  def combine(x: T, y: T): T
}
def all[T](lst: List[T], m: Monoid[T]): T = lst.foldLeft(m.empty)(m.combine)
```
or with functional library [Cats][Cats] (which contains among other things definition of `Monoid`) and further optimizations:

```scala
import cats.Monoid
import cats.implicits._
def all[T: Monoid](lst: List[T]): T = lst.foldLeft(Monoid[T].empty)(_ |+| _)
```

Since in the function we need only `foldLeft`, `List[T]` can be replaced with further abstraction:

```scala
import cats.{Monoid, Foldable}
import cats.implicits._
def all[T: Monoid, F[_]: Foldable](lst: F[T]): T = lst.foldLeft(Monoid[T].empty)(_ |+| _)
```
`Foldable` also have this nice feature: it can be composed. 

```scala
F[_] <=> Foldable[F]
G[_] <=> Foldable[G]
type FG[T] = F[G[T]]
F[G[_]] <=> Foldable[F].compose[G] <=> Foldable[FG]
```

[to top][0]

## Pure functions

The concept of pure function is of the most importance in FP. My favorite example of one such function is arithmetic operator `+`. It produces new instance based on arguments and it does only this. It does not change passed objects (which in itself is very bad habit), nor does it change or use some hidden data to produce results. 

When starting writing pure function, write full signature:
```scala
def foo[T, R](x: T): R
```
This will also help you later: since it is pure function, you know that it does nothing to the arguments (you may use them again), and it always returns value of type `R`. So when you see application of the function in the code, in order to get the idea what is going on here, you just need to see it's signature, without the need to dive into all the nuances of the implementation ([referential transparency][RP]).

#### No loops. 
Loops are essentially imperative constructions, which also ties via closure internal mutable(!) variable to the algorithm inside it. 

Use iteration and transformation over collection. Both standard library and functional libraries ([Cats][Cats], [Scalaz](Scalaz), etc.) have many specialized functions like `map`, `filter`, `fold`, etc. Among many benefits of using these functions are:

* increased readability of the code - explicit names of the functions
* functional (declarative) way of iterating over elements
* for some types there can be more robust implementation of the same iterative function
* some non-collection types implement the same functions (like `map` for `Option`) with the same meaning 

Only when you have no other chose left, write recursive function with [tail call][TailCall]. This kind of recursions is stack safe, since it is transformed by the compiler to loop (oh irony, but also an example of [postponing the effects](#postpone_effects) - another perk of FP). BTW, all the library functions use [tail call][TailCall] recursions under the hood.

The function [all](#magic) from example above may be rewritten with [@tailrec](https://www.scala-lang.org/api/2.13.1/scala/annotation/tailrec.html) as follows:
```scala
def all[T: Monoid](lst: List[T]): T = {
  val M = Monoid[T]
  @tailrec
  def rec(l: List[T], acc: T): T = {
    if(l.isEmpty) acc 
	else rec(l.tail, M.combine(acc, l.head))
  }
  rec(lst, M.empty)
}
```

Good explanation of tail recursion can be found [here](https://www.scala-exercises.org/scala_tutorial/tail_recursion).

#### State

Another popular use case is calculations based on previous element(s), in other words state. This is very counterintuitive part of FP: the state of the [Finite-State-Machine](https://en.wikipedia.org/wiki/Finite-state_machine) should be an outside parameter. But it is against the rules to introduce mutable variable inside function since the function becomes non-pure. With mutable state the function has side-effect built-in. The solution is (as almost always) in the type that will be returned. The function will receive state and return result **and** new state.

Very thorough discussion of the state and it's use in FP can be found [here](https://typelevel.org/cats/datatypes/state.html), but in I'll give a gist. Random generator as it is - it is not functional way. Each call produce new result without arguments. Not a pure function. The solution is to define `random` function as follows:

```scala
def random(prev: Long): Long = prev * 6364136223846793005L + 1442695040888963407L
```

(see Knuth’s 64-bit [linear congruential generator](https://en.wikipedia.org/wiki/Linear_congruential_generator) for the formula). In this particular case, the state is the same as desired value. But let's say we need to know the total number of calls to `random`:

```scala
final case class Seed(seed: Long, count: Long) 
def random(s: Seed): (Seed, Long) = {
    val n = Seed(s.seed * 6364136223846793005L + 1442695040888963407L, count + 1)
    (n, n.seed)
}
```

This way `random` still depends on state while being pure function.

[to top][0]

## Always produce result. No exceptions.

Function should always produce a value. Exception is not thrown. Whenever there is a possibility of non-value result (exception, void, undefined, null, etc.), it should be incorporated in the result type. Such types include but not limited to:

```scala
Option[T]
Try[T]
JsResult[T]
Future[T]
//etc.
```

And of course other such types may be introduced as aliases via `type` or as case-classes:

```scala
type Result[T] = Either[Throwable, T]
```

Functional approach not only better because it is functional, but also it is more robust in general case and in the edge cases gives more control over error handling to developer.

#### Throwing exception is expensive

Whenever exception is thrown, runtime have to rollback the call stack till the closest appropriate `catch`  collecting a lot of data along the way, while, with functional approach, the execution chain just stops whenever any function returns an "error":

```scala
def foo: Result[T]
def bar(t: T): Result[T]

foo.flatMap(bar).map(...)
```

In this example, if `foo` returns `Left`, the `flatMap` is not evaluated (see implementation of `Either`), and neither is the rest of the chain. If `foo` returns `Right`, but `bar` returns `Left`, then the `map` would not run. As simple as that.

#### Easier error processing

Since error is allowed as result, we can work with errors same way as with 'valid' result, e.g. transform to default value, collect all errors from many executions, define retry strategy, etc. without excessive code branching as in case of exception/null/undefined based error-handling. 

[to top][0]

## Immutability

Another cornerstone of FP is immutability. Functional paradigm does not allow to change values. As I've mentioned earlier, absolutely functional code will be absolutely useless, the same is with immutability. The goal is not to write 100% immutable code (although this is good), but to limit mutable code to some reservations, which should be thoroughly tested. Also, these places are the first suspects for bugs.

BTW: while writing immutable code, you will also write [pure functions](#pure_functions).

Immutability has it's advantages:

* the same instance can be shared indefinitely. One of the consequences of _this_ fact is that parallel access is trivial - no need to sync since only read is possible

* since each function is (should be) pure function and the result depends only on arguments passed, the order of the calls important only if some function uses results of evaluation of another functions. This also means, that compiler may deduce a call tree and run some evaluations in parallel.

One may argue that always creating new objects adds pressure to the memory (in terms of usage and consumption) and to garbage collector. I will argue with that with the facts that:

* most of these immutable instances are short-lived instances. They are forgotten during the first generation (Gen-1). It is the mutable instances that usually survive to Gen-2. As for immutable long-live instances, most of those are application-level constants or singleton instances that lives through to Gen-3.

* As mentioned above, when using immutable values, sharing them became trivial. No need neither for defensive copy-on-share, nor for synchronization of access. Which is beneficial for code complexity, footprint, memory usage and GC.

* And as a last argument (in this article) in favor for immutability, since immutable object doesn't mutate, GC does not need to take it as root on the next scan, again less work for GC. 

More about it [here](https://stackoverflow.com/questions/35384393/how-do-immutable-objects-help-decrease-overhead-due-to-garbage-collection) and [here](https://www.reddit.com/r/programming/comments/1i3738/is_immutability_good_for_gc_performance/). And, from Oracle [docs](https://docs.oracle.com/javase/tutorial/essential/concurrency/immutable.html):

> The impact of object creation is often overestimated, and can be offset by some of the efficiencies associated with immutable objects. These include decreased overhead due to garbage collection, and the elimination of code needed to protect mutable objects from corruption.

[to top][0]



## Function - first class citizen

Each function has it's type, which consists of types of the arguments and type of the result:

```scala
def foo(i: Int, d: Double): String = ??? // (Int, Double) => String
def bar(b: BigInt): Boolean = ??? // BigInt => Boolean
```

A function with a proper type can be passed to or returned from another function. Most probably you already have seen/used it. `Array.map`, `Array.sort`, etc. in JavaScript, C#, Scala, etc., all these functions in all these languages are accepting transformation/sort/etc. function as one of the parameters. If you've used one of those, you've used function as a first class citizen. Functions like `Array.map` are called **Higher Order Functions**, because these functions are dealing with other functions either as arguments, or as result value (or both).

And the next reasonable step is to assign function to a variable:

```scala
val foo: (Int, Double) => String = (i, d) => s"$i $d"
val baz: BigInt => Boolean = _.isValidInt
```

[to top][0]



## Types

There are two sorts of types in FP: **Data Type** and **Type Class**.

##### Data Type

Data type is a type to store data. With data type you describe measurement from sensor, payment, event or even some executable code (see [Function - is first class citizen](#function---is-first-class-citizen)). Usually `case class` is used to define this type. "Methods" for this type are defined either in [value class](https://docs.scala-lang.org/overviews/core/value-classes.html) or in [universal traits](https://docs.scala-lang.org/overviews/core/value-classes.html) (the trait doesn't have to extend `Any` but the methods should be [pure functions](#pure_functions)).

##### Type Class

Type class defines algebra (set of operations) for particular data type. Type class interface defines some operations available on some data type. Usually it is **Higher-Kinded Type** (which only means that the `trait` has type-parameter: `trait Monoid[M]`) so that instance of the type class defines operations for one specific data type.

In most cases data type is just a way to pass some meaningful data, while type class holds all the know-how of how to handle this type.

What are the benefits of this separation, one might ask. One obvious advantage is the ability to generalize the algorithms over wider range of type than with mere inheritance (see thorough explanation [here](https://typelevel.org/cats/typeclasses.html#type-classes-vs-subtyping)). Less obvious benefit arises from the *constrains* that this approach enforce on the developer. It enforces to think of the data type as chunk of data and nothing more. The algorithms become type-independent - all type-related know-how are encapsulated in the type class. And so, by designing the model in accordance with the types-duality (type class & data type), the code is separated to two distinct areas of **what** (algorithms) and **how** (type classes).

To illustrate this, let's define simple payment system:

```scala
trait Currency
case object Dollar extends Currency
case object Pound extends Currency
case class Purchase(amount: Double, currency: Currency)
```

Before we bill the user, it is better to collect all payments per each currency and charge once per currence. It can be achieved by quite simple folding:

```scala
def collect(charges: List[Purchase]): Map[Currency, Purchase] = {
  def add(a: Purchase, b: Purchase): Purchase = a.copy(amount = a.amount + b.amount)
  def empty(c: Currency): Purchase = Purchase(0.0, c)    
  charges.foldLeft(Map.empty[Currency, Purchase]){
    (m, p) => m + (p.currency -> add(p, m.getOrElse(p.currency, empty(p.currency)))
  }
}
```

On the other hand, we will have measurements from different parts of the system:

```scala
case class Measurements(unit: String, data: Double)
```

These measurements may be in completely different units (number of clicks vs. avg time on page) or in different scales - KB/MB. In this case the measurement should be separated by units, but transformed to common scale. The function `add` for `Measurement` will be a little different:

```scala
def scale(a: Measurement): Measurement = a.unit.splitAt(1) match {
  case ("K", u) => Measurement(u, a.data / 1000)
  case ("M", u) => Measurement(u, a.data / 1000000)
}
def addMeasurement(a: Measurement, b: Measurement): Measurement = {
  val sa = scale(a)
  sa.copy(amount = sa.amount + scale(b).amount)
}
```

With this additions the `collect` can be repeated for `Measurement`. But let's try and not repeat ourselves. First let's define the number of operations needed for `collect`:

```scala
trait Collectable[T, Key] {
  def add(a: T, b: T): T
  def empty(key: Key): T
  def key(a: T): Key
}
```

With this trait `collect` can be transformed to:

```scala
def collect[T, Key](lst: List[T], C: Collectable[T, Key]): Map[Key, T] = 
  lst.map(x => C.key(x) -> x)
    .foldLeft(Map.empty[Key, T]){
      (m, t) => m + (t._1 -> C.add(t._2, m.getOrElse(t._1, C.empty(t._1))))        
    }
```

All is left is to define `Collectable` for both `Measurement` and `Purchase`:

```scala
val purColl = new Collectable[Purchase, Currency] {
  def add(a: Purchase, b: Purchase): Purchase = a.copy(amount = a.amount + b.amount)
  def empyt(key: Currency): Purchase = Purchase(0.0, key)
  def key(a: Purchase): Currency = a.currency
}
val measColl = new Collectable[Measurement, String] {
  def add(a: Measurement, b: Measurement): Measurement = addMeasurement(a, b)
  def empty(key: String): Measurement = scale(Measurement(key, 0.0))
  def key(m: Measurement): String = m.unit
}
```

And thus, we have generic algorithm `collect`, and all type-specific know-hows are encapsulated in the instance of `Collectable` type class. More on this can be found [here](https://typelevel.org/cats/typeclasses.html#a-note-on-syntax).

[to top][0]

## Conclusion

In order to become able to write functional code, or better yet to start thinking functional, you have to exercise it (as with any other language), otherwise it will stay curios academic tricks not applicable to real-life problems. I hope this article will help you to make the first step (leap of faith) to _start_ writing functional code.

[to top][0]

[0]: #how-i-learned-to-stop-worrying-and-love-fp-in-scala
[RP]: https://en.wikipedia.org/wiki/Referential_transparency
[Cats]: https://typelevel.org/cats/
[TailCall]: https://en.wikipedia.org/wiki/Tail_call
[Scalaz]: https://scalaz.github.io