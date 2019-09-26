
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
