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
and then, having the last function we can do this:
```scala
implicit val fld = Foldable[List].compose[Option]
sum(List(Some(1), Some(2), None, Some(4))) // 7
```
