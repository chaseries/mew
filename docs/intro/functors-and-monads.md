## Functors

If you come from the imperative world and know a modern dynamic language like JavaScript or Python, you're probably familiar with the `map` function, but you may not be aware of its broader context.

Mapping is actually a very general idea that is not restricted to sequences like arrays or lists. It can happen over many other data structures that serve as containers for underlying values. This makes sense: things like trees, options, and validators all contain some more primitive values wrapped up in their structure, and it is often these contents that we care about manipulating the most. Such "mappable" data structures are known as *functors* in the functional programming world.

As a rule, `map` is always a function of type `(a -> b) -> f a -> f b`. This type signature specifies the following things about `map`:

1. Its first parameter is a function that takes some value of type `a` and evaluates to some value of type `b`, denoted above as `(a -> b)`
2. Its second parameter is a data structure of type `f` that holds an inner value of type `a`, denoted above as `f a`
3. It evaluates to a value of type of `f b`—that is, the same type of data structure, but now with a new inner value of type `b`

Naturally, the `f b` was produced by `map` by somehow applying `(a -> b)` to `f a`. This is a data structure-specific definition; in Mew, mapping over lists and arrays could be formulated recursively as such:

```
mapList : (a -> b) -> List a -> List b
mapList f []    = []
mapList f x++xs = f x ++ (mapList f xs)
```

It's worth noting that your function `a -> b` does not *necessarily* output a different type than it accepts. You could pass in a `plusOne` function of type `Integer -> Integer` and it would be a perfectly valid concretization of `a -> b`. In idiomatic Mew, that would look like this:

```
μ  map (+1) [1,2,3]
[2,3,4] 
```
Of course, an actual `a -> b` transformation is also acceptable:

```
μ  map toChar [97,98,99]
[<'a'>,<'b'>,<'c'>]
```
That output type is `List (Maybe Char)`, and we're going to talk about it presently.

### Map Me Maybe

*If you are familiar with optionals like `Option` or `Maybe`, you can safely skip to the next subheading.*

There is something simpler than lists that can be mapped over, and it's analogous to a list that can only have 0 or 1 elements. It's a type called `Maybe`, and its terms are used to represent failure or success with a payload. The definition of `Maybe` is:

```
type Maybe a = Nothing | Just a
```
Functions in Mew that evaluate to a `Maybe` are roughly equivalent to those functions in imperative languages that can either return some value, like a user ID (`Just id`), or fail to produce the requested value and return a null type (`Nothing`).

How would one map over a value of type `Maybe`? Very easily, which is why it makes a great example:

```
mapMaybe : (a -> b) -> Maybe a -> Maybe b
mapMaybe f Nothing  = Nothing
mapMaybe f (Just a) = Just (f a)
```

This definition means we have a type that never demands that we type-check its contents for success or failure. Any function that's valid for mapping over `Just a` is also valid for mapping over `Nothing`; there are no nasty runtime bugs of this class to be had. What's more, the Mew compiler will insist that pattern matches and other control flow over any type `Maybe a` account for both possible inhabitant terms, `Just a` and `Nothing`.

### Bringing it together

Of course, `Maybe` is a functor, as are `List`, `Set`, the various tree types, and many more.

### Polymorphizing map

You may have noticed in your explorations of Mew that we don't have to use `mapMaybe` on values of type `Maybe` or `mapList` on values of type `List`. For all mappable types (that is, functors), `map` suffices. That's because `Functor` is an interface that types can be a part of. The interface's only requirement is that those types implementing it define `map` in their instance declaration.

There's nothing magical going on here. The `Functor` interface, like the `Monad` interface we'll soon see, is plain old Mew, and it's actually pretty simple:

```
interface Functor f
  map : (a -> b) -> f a -> f b

instance Maybe of Functor
  map f (Just a) = Just (f a)
  map f Nothing = Nothing
```

## Monads

Let's recall the type of `map` on any functor:
```
map : (a -> b) -> f a -> f b
```

Easy enough, right? If you want to make your type a functor, implement a `map` that works over it.  That's the rule specified in the interface. You're done. The conception of a functor, then, is just a container type that defines a `map` function so that its contents may be altered. The implementation varies and the type signature of `map` is specialized based on the functor in question, but it follows the same general formula. When used on `Maybe`, the type specializes to `(a -> b) -> Maybe a -> Maybe b` and the function argument is applied to the contents of `Just` (with a no-op on `Nothing`). When used on a `List`, the type specializes to `(a -> b) -> List a -> List b`, and the function argument is iteratively applied to each element in a list (with a no-op on empty lists).

If this all feels belabored, it is to hopefully demonstrate that `Functor` is actually pretty easy. It turns out that its sister, `Monad`, is structurally nearly identical.

So let's play around with the type of `map`. Below is the type signature of our original `map`, renamed `functorMap` for clarity, and a very similar signature for a new function, `monadMap`.

```
functorMap : (a -> b)   -> f a -> f b
monadMap   : (a -> f b) -> f a -> f b
```

The parameter `(a -> f b)` of `monadMap` is the only difference between it and `functorMap`, and it's what all the hand-wringing over monads is over. It turns out that there are some meaningful implications to that subtle change. One is that since you, the user, return a custom `f b`, you control not only the inner values, but also the structure of the data you're operating on. Let's revisit `Maybe`, in particular where `monadMap` receives a `Just a`:

```
functorMapMaybe : (a -> b) -> Maybe a -> Maybe b
functorMapMaybe f Nothing  = Nothing
functorMapMaybe f (Just a) = Just (f a) -- Pay attention here...

monadMapMaybe : (a -> Maybe a) -> Maybe a -> Maybe b
monadMapMaybe f Nothing  = Nothing
monadMapMaybe f (Just a) = f a  -- ...and here.
```

In the monadic version, the resulting `Maybe a` term is supplied by you via your function parameter `f`: `monadMapMaybe f (Just a) = f a`. In the functor version, it is embedded into the definition of the mapping function itself: `functorMapMaybe f (Just a) = Just (f a)`. 

You may or may not have noticed before that `functorMap` does not let you transform the structure of the term you're operating on. You could run a million `functorMap` applications over `Just a` and, while you may change it to `Just b`, you can never produce `Nothing`. It is a structure-preserving operation.

But `monadMap` discards the structure entirely in each application, forcing you to resupply it. Thus the outer structure of any term that you chose to produce via `f` can depend on the inner value of the `Just a` that you're mapping over. Here's an example function:

```
μ  exampleFunction v = if v < 10 then Nothing else Just v
μ  monadMap exampleFunction (Just 9)
Nothing
```

The implication may still be opaque until you reflect back onto the definition of `monadMap`. Monad mapping `f : (a -> f b)` over `Just a` can yield `Just b` or `Nothing`, but mapping over `Nothing` always yields `Nothing`. In a nested sequence of `monadMapMaybe` functions, any function that produces a `Nothing` will change the result of every function applied after it to `Nothing`. That is, you can short-circuit any nesting of `monadMapMaybe` functions.

You've created an automatic effect system, almost as though the type itself represents a sub-language within Mew. Take this example:
```
μ  divideBy 0 x = Nothing
μ  divideBy a x = Just (x / a)
μ  monadMap (divideBy 2) (monadMap (divideBy 1) (monadMap (divideBy 0) (Just 1)))
Nothing
```
That's pretty hairy to look at. Let's try redefining `monadMap` as an operator for convenience's sake:

```
(>>=) : f a -> (a -> f b) -> f b
(>>=) = flip monadMap

Just 1 >>= divideBy 0 >>= divideBy 1 >>= divideBy 2
```
The syntax may be novel, but this is the exact same thing as before. First `divideBy 0` is mapped onto `Just 1` (producing, of course, `Nothing`). Then `divideBy 1` is mapped to this result, followed finally `divideBy 2`.

Since we got a `Nothing` somewhere in our chain of operations, `Nothing` is what we get as the result. The effect looks kind of like a boolean `AND` with a payload, where terms that evaluate to `Just` are `True` and those that evaluate to `Nothing` are `False`.
