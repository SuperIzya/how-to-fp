# How I learned to stop worrying and love FP in Scala

Since I've recently struggled trough understanding of the basics of FP, and since the hardest part was to start thinking functional, I've decided to share some tips how to make first steps easier.  

### Pillars of FP

There are several major concepts in functional programming. And once you understand them, you're half way through.

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

![Why FP](./gifs/types-algebras.gif)

Functional programming is a programming paradigm based on [lambda calculus](https://en.wikipedia.org/wiki/Lambda_calculus), which is
> ...formal system in mathematical logic for expressing computation...
> It is a universal model of computation that can be used to simulate any Turing machine.

There are major advantages to use FP:

* Functional code is easy reasoning about due to:

  * [referential transparency][RP]: a function is referential transparent if by replacing the function call in code by the result of evaluation of the function with current arguments (given that they are known), the behavior of the code wouldn't change. Function 

    ```scala
    def add(x: Int, y: Int): Int = x + y
    ```

    is referential transparent. Every call to this function in the code can be replace by its result. This also means that no matter the scale, it takes about the same amount of effort to understand the code.

  * functional code - declarative code.

* Fewer bugs due to:
    * code is easier to reason about (see above)
    * since the code is much more like a mathematical model, the compiler can check it more thorough. The compilation will fail in case your model is not converging, or there are illegal (in current model) operations
* It is almost always better to tell __what__ to do instead of __how__. By telling __what__ to do, you're using higher level of abstractions. Some smart people dedicated a lot of time and effort to implement the abstraction efficient and correct. In rare cases, when you need something better, you'll dedicate all the resources to make it the best way possible. You will not write your own web server, in case you need something special, and then it will become one of _core businesses_. Functional code tells computer __what__ to do ([declarative code](https://en.wikipedia.org/wiki/Declarative_programming)), while your regular C-like code is telling the machine __how__ to do what it should do ([imperative code](https://en.wikipedia.org/wiki/Imperative_programming)).

[to top][0]

## Magic

![magic](./gifs/magic.gif)

(For more complete example [see here](https://typelevel.org/cats/typeclasses.html))

Let's say we have two functions:

```scala
def sumInt(lst: List[Int]): Int = lst.foldLeft(0)(_ + _)
def sumSet[A](lst: List[Set[A]]): Set[A] = lst.foldLeft(Set.empty[A])(_ union _)
```

Both of these functions are the same except for the type-specific seed element and method for combining two elements. _Define Monoid_. _Monoid is mathematical Group_
So let's get them out and pass this data as a parameter the algorithm (to the function `sum`):

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
sum[Int, G](List(Option(1), Option(2), None, Option(4))) // 7
```

This may look like magic or gibberish (or both) , but I'll do my best to get you (at least closer) to understanding the whys and hows of this parlor trick. But do remember, this is a language, and as with any other language, you need to practice it if you want to think it.

[to top][0]

## Always produce result. Don't throw exceptions

![Result](./gifs/result.gif)

Function should always produce a value. Exception is not thrown!!!. Whenever there is a possibility of non-value result (exception, void, undefined, null, etc.), it should be incorporated in the result type. Such types include but not limited to:
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

#### Exception is expensive

Whenever exception is thrown, runtime have to rollback the call stack till the closest appropriate `catch`  collecting a lot of data along the way, while, with functional approach, the execution chain just stops whenever any function returns an error:

```scala
def foo: Option[T]
def bar(t: T): Option[T]

foo.flatMap(bar).map(...)
```

In this example, if `foo` returns `None`, the `flatMap` is not called (see implementation of `Option`). If `foo` returns `Some`, but `bar` returns `None`, then the `map` would not run. As simple as that.

#### Easier error processing

Since error is allowed as result, we can work with errors same way as with 'valid' result, e.g. transform to default value, collect all errors from many executions, define retry strategy, etc. without excessive code branching as in case of exception/null/undefined based error-handling. 

[to top][0]



## Pure functions (move up)

![pure](./gifs/pure.gif)

The concept of pure function is of the most importance in FP. My favorite example of one such function is arithmetic operator `+`:
1. It does not change it arguments. After evaluating `2+3`, both `2` and `3` are the very same, they were not changed by the function
1. It returns a value (always). Doesn't matter how many times the function is called, for the same arguments set the result is  the same. The order of calls with different sets of arguments is also does not change the results of each individual evaluation.

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

Another popular use case is calculations based on previous element(s), in other words state. Very thorough discussion of the state and it's use in FP can be found [here](https://typelevel.org/cats/datatypes/state.html), so I will be very brief. This is very counterintuitive part of FP: the state of the automaton should be an outside parameter. But it is against the rules to introduce mutable variable inside function: the function becomes non-pure. With mutable state the function has side-effect built-in. The solution is (as almost always) in the type that will be returned. The function will receive state and return result **and** new state:

```scala
type State = ???
type Result = ???
def foo(s: State): (State, Result) = ???
```

As you can see, within this implementation, `foo` is pure function, and given the same `State` will return essentially the same pair `(State, Res)`.

Random generator.





[to top][0]



## Immutability

![immutability](./gifs/immutability.gif)

Another cornerstone of FP is immutability. Think again of `+`:

```scala
def sum(x: Int, y: Int): Int = x + y
```

Neither `x`, nor y can't suddenly change to something else (same goes to the result). Each pure function returns new instance. Except for when it returns `object` (singleton) or `val`, pure function __always__ returns new immutable object. NOT!!! Arguments are __never__ mutated.



No to object transformation. 





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

[to top][0]



## Function - first class citizen

![function](./gifs/function-fcz.gif)

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

There are two sorts of types in FP: **Data Type** and **Type Class**

##### Data Type

![types](./gifs/data-type.gif)

Data type is a type to store data. With data type you describe measurement from sensor, payment, event or even some executable code (see [Function - is first class citizen](#function---is-first-class-citizen)). Usually `case class` is used to define this type. "Methods" for this type are defined either in [value class](https://docs.scala-lang.org/overviews/core/value-classes.html) or in [universal traits](https://docs.scala-lang.org/overviews/core/value-classes.html) (the trait doesn't have to extend `Any` but the methods should be [pure functions](#pure_functions)).

##### Type Class

![types](./gifs/types.gif)

Type class is an algebra for data type. Type class interface defines some operations available on some data type. Usually it is **Higher-Kinded Type** (which only means that the `trait` has type-parameter: `trait Monoid[M]`) so that instance of the type class defines operations for one specific data type.

So in most cases data type is just a way to pass some meaningful data, while type class holds all the know-how of how to handle this type.

What are the benefits of this separation, one might ask. One obvious advantage is the ability to generalize the algorithms over wider range of type than with mere inheritance (see thorough explanation [here](https://typelevel.org/cats/typeclasses.html#type-classes-vs-subtyping)). Less obvious benefit arises from the *constrains* that this approach enforce on the developer. It enforces to think of the data type as chunk of data and nothing more. The algorithms become type-independent - all type-related know-how is encapsulated in the type class. And so, by designing the model in accordance with the types-duality (type class & data type), the code is separated to two distinct areas of **what** (algorithms) and **how** (type classes).

So back to our example with `sum`, where we where with this implementation:

```scala
def sum[T](lst: List[T], m: Monoid[T]): T = lst.foldLeft(m.empty)(m.combine)
```

Here `m` is instance of class type for type `T` and contains needed operations for the algorithm (`empty` and `combine`). Obviously, `m` may be the same instance for all calls with the same type parameter `T`. So it's reasonable to move it to be implicit parameter:

```scala
def sum[T](lst: List[T])(implicit m: Monoid[T]): T = lst.foldLeft(m.empty)(m.combine)
```

which, with syntactic sugar, may be rewritten as:

```scala
def sum[T: Monoid](lst: List[T]): T = lst.foldLeft(Monoid[T].empty)(Monoid[T].combine)
```

More on this [here](https://typelevel.org/cats/typeclasses.html#a-note-on-syntax).

[to top][0]



## Postpone effects

![postpone](./gifs/postpone.gif)

As a result of the rules above, but also considered as one of the pillars of the FP is **postponing effects**. Think about it. When you use `Iterator[A].map`, you don't have even single value of type `A` yet, but still, you act as if you already have. What you loose here is that, after a `Iterator[_].map` you still have `Iterator`, or, in other words, you haven't left the context of the `Iterator`. In effect, you've built some computation for value(s) that doesn't exist yet (until somebody reads from the `Iterator`) and you don't know if you even going to get one (`Iterator` may be empty). And thus we have written some code, in context of the effect (iteration over the `Iterator`), that may or may not happen in some point in the future - **end of the world**.

[to top][0]



## Conclusion

![conclusion](./gifs/conclusion.gif)

I hope this article will help you to start understanding the concepts of FP, but as always, in order to become able to write functional code, or better yet to start thinking functional, you have to exercise (as with any other language), otherwise it will stay curios academic tricks not applicable to real-life problems.

[to top][0]

All images are taken from movie [Dr. Strangelove or: How I Learned to Stop Worrying and Love the Bomb](https://www.imdb.com/title/tt0057012/)

[0]: #how-i-learned-to-stop-worrying-and-love-fp-in-scala
[RP]: https://en.wikipedia.org/wiki/Referential_transparency
[Cats]: https://typelevel.org/cats/
[TailCall]: https://en.wikipedia.org/wiki/Tail_call
