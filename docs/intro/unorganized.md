### Functor

Functor type class

```
class Functor f where
  map : (a -> b) -> f a -> f b
```

### Applicative
```
class (Functor f) => Applicative where
  apply : f (a -> b) -> f a -> f b
  pure : a -> f a
```
### Monad

```
class Monad where
  bind : m a -> (a -> b) -> m b
  pure : a -> m a
```

### List comprehensions

List comprehensions are syntactical conveniences to generate a new list from a source list. Consider that we would like to take some list `[1,2,3,4,5]` and multiply each element by two. We can, of course, map it:

```
μ  map (*2) [1,2,3,4,5]
[2,4,6,8,10]
```
Or, with list comprehension syntax, we can do the following:

```
μ  [x * 2 | x <- [1,2,3,4,5]]
[2,4,6,8,10]
```

Is this easier? Here, no. But that's not where the story ends. List comprehensions also allow the user to filter the source list according to a *predicate*. Let's assume we want to do the same as before, but only on odd numbers, and let's declare that `myList = [1,2,3,4,5]`. Traditionally, we might do this:
```
μ  map (*2) <| filter (\x -> x % 2 != 0) myList
[2,6,10]
```
List comprehensions clean that up a bit:
```
μ  [x * 2 | x <- myList, x % 2 != 0]
[2,6,10]
```
Want another predicate? Add another predicate. Everyone gets a predicate. One is the loneliest number, so kick him out:
```
μ  [x * 2 | x <- myList, x % 2 != 0, x != 1]
Result: [6,10]
```
The story continues. List comprehensions are designed to represent nondeterministic computation. Instead of explaining that $5 term, let's explore it with an example:
```
μ  write n a = glue "" [n, " is ", a, "."]

μ  nouns = ["Chase", "Dresden"]
μ  adjs  = ["fat", "happy", "old"]

μ  [write n a | n <- nouns, a <- adjs]

[ "Chase is fat."
, "Chase is happy."
, "Chase is old."
, "Dresden is fat."
, "Dresden is happy."
, "Dresden is old."
]
```

Dresden is my dog. I'm actually not fat, but he sure is, so let's fix that with a predicate:

```
μ  [write n a | n <- nouns, a <- adjs, n != "Chase" || a != "fat"]

[ "Chase is happy."
, "Chase is old."
, "Dresden is fat."
, "Dresden is happy."
, "Dresden is old."
]
```
Sorry, Dresden.

That one-liner above would be a quite bit longer without list comprehensions, and the source lists and predicates can go on arbitrarily. Though they can absolutely be abused, they're great for writing terse, readable code to manipulate lists.
