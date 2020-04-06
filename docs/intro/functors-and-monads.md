## Functors

If you come from the imperative world and know a modern dynamic language like JavaScript or Python, you're probably familiar with the `map` function, but you may not be aware of its broader context.

Mapping is actually a very general idea that is not restricted to sequences like arrays or lists. It can happen over many other data structures that serve as containers for underlying values. This makes sense: things like trees, options, and tuples all contain some more primitive values wrapped up in their structure, and it is often these contents that we care about manipulating the most. Such "mappable" data structures are known as *functors* in the functional programming world.

As a rule, `map` is always a function of the generalized type `(a -> b) -> f a -> f b`. This type signature specifies the following things about `map`:

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
[Just 'a',Just 'b',Just 'c']
```
That output type is `List (Maybe Char)`, and we're going to talk about it presently.

### Map Me Maybe

*If you are familiar with optionals like `Option` or `Maybe`, you can safely skip to the next subheading.*

There is something simpler than lists that can be mapped over, and it's analogous to a list that can only have 0 or 1 element. It's a type called `Maybe`, and its terms are used to represent failure or success with a payload. The definition of `Maybe` is:

```
type Maybe a = Nothing | Just a
```
Functions in Mew that evaluate to a `Maybe` are roughly equivalent to those functions in imperative languages that can either return some value, like a character from an ASCII table (`Just char`), or fail to produce the requested value and return a null type (`Nothing`). ASCII is pretty limited, after all. At most it has has 256 elements indexed from `0`; thus `toChar 256` is undefined and a reasonable state to call `Nothing`.

How would one map over a value of type `Maybe`? Very easily, which is why it makes a great example:

```
mapMaybe : (a -> b) -> Maybe a -> Maybe b
mapMaybe f Nothing  = Nothing
mapMaybe f (Just a) = Just (f a)
```

This definition means we have a result that never demands that we type-check its contents for success or failure. Any function that's valid for mapping over `Just a` is also valid for mapping over `Nothing`; there are no nasty runtime bugs of this class to be had. What's more, the Mew compiler will insist that pattern matches and other control flow over any type `Maybe a` account for both possible inhabitant terms, `Just a` and `Nothing`.

### Bringing it together

Of course, `Maybe` is a functor, as are `List`, `Set`, the various tree types, and many more.

TODO: Add summary.

### Polymorphizing map

You may have noticed in your explorations of Mew that we don't have to use `mapMaybe` on values of type `Maybe` or `mapList` on values of type `List`. For all mappable types (that is, functors), `map` suffices. That's because `Functor` is an interface that types can be a part of. The interface's only requirement is that those types implementing it define `map` in their instance declaration.

There's nothing magical going on here. The `Functor` interface, like the `Monad` interface we'll soon see, is plain old Mew, and it's actually pretty simple:

```
interface Functor f
  map : (a -> b) -> f a -> f b

instance Maybe of Functor
  map f (Just a) = Just (f a)
  map f Nothing = Nothing

instance List of Functor
  map f [] = []
  map f x++xs = f x ++ map f xs
```

Giving the same function a different definition based on the type it is passed is called *function overloading*, and it's a form of polymorphism. It tidies up code in the real world, emphasizing that the same conceptual operation is being applied to different data types, even if the exact implementation varies.

It's important to note that a polymorphic `map` is the normal, nigh exclusive, way to write Mew in real life, but for pedagogical purposes we may return to using more specific mapping functions like `mapMaybe` throughout this guide.

## Monads

Let's recall the type of `map` on any functor:

```
map : (a -> b) -> f a -> f b
```

Easy enough, right? If you want to make your type a functor, implement a `map` for it.  That's the rule specified in the interface. You're done. The conception of a functor, then, is just a container type that defines a `map` function so that its contents may be altered. The implementation varies and the type signature of `map` is specialized based on the functor in question, but it follows the same general formula. When used on `Maybe`, the type specializes to `(a -> b) -> Maybe a -> Maybe b` and the function argument is applied to the contents of `Just` (with a no-op on `Nothing`). When used on a `List`, the type specializes to `(a -> b) -> List a -> List b`, and the function argument is iteratively applied to each element in a list (with a no-op on empty lists).

If this all feels belabored, it is to hopefully demonstrate that `Functor` is actually pretty easy. It turns out that its sister, `Monad`, is structurally identical. In fact, all `Monad` instances are, by definition, `Functor` instances. They can (and commonly do) use `map`. So what changes? Only one critical thing: we add another, very-similar-looking mapping function.

So let's play around with the type of `map` and see what we come up with. Below is the type signature of our original `map`, aliased to `functorMap` for clarity, and a very similar signature for a new function, `monadMap`.

```
functorMap : (a -> b)   -> f a -> f b
monadMap   : (a -> f b) -> f a -> f b
```

The parameter `(a -> f b)` of `monadMap` is the only difference between it and `functorMap`, and it's what all the hand-wringing over monads is about. It turns out that there are some powerful implications hidden in that subtle change. In particular, since you, the user, return a custom `f b`, you control not only the inner values, but also the structure of the data you're operating on. Let's revisit `Maybe`, in particular where each interface's version of the map function matches on a `Just a`:

```
functorMap : (a -> b) -> Maybe a -> Maybe b
functorMap f Nothing  = Nothing
functorMap f (Just a) = Just (f a) -- Pay attention here...

monadMap : (a -> Maybe a) -> Maybe a -> Maybe b
monadMap f Nothing  = Nothing
monadMap f (Just a) = f a  -- ...and here.
```

In the functor version, the structure of the term that is output is embedded into the definition of the mapping function itself. You change only the inner value; the wrapping `Just` and `Nothing` terms are inaccessible to you. In the monadic version, the resulting structure is supplied by you via `f`.

You may or may not have noticed before that `functorMap myFunction` does not let you transform the structure of the term you're operating on. You could run a million `functorMap myFunction` applications over `Just a` and, while you may change it to `Just b`, you can never produce `Nothing`. It is a structure-preserving operation.

But `monadMap` discards the structure entirely in each application, forcing you to resupply it. Thus the outer structure of any term that you chose to produce via `f` can depend on the inner value of the `Just a` that you're mapping over. You can change the whole kit and caboodle for any reason you want. Here's a trivial example function:

```
μ  myFun v = if v < 10 then Nothing else Just v
μ  monadMap myFun (Just 11)
Just 11
μ  monadMap myFun (Just 9)
Nothing
```

The implication may still be opaque until you reflect back onto the definition of `monadMap`. Even though you supply the structure, there are still rules embedded into the mapping function itself, and they too can change the outcome. Recall that monadic mapping `f` over `Just a` can yield `Just b` or `Nothing`, while mapping any `f` over `Nothing` always yields `Nothing`.  So in a nested sequence of `monadMap` functions, any function that produces a `Nothing` will change the result of every function applied after it to `Nothing`. That is, we can choose to short-circuit any nesting of `monadMap` functions based on the inner value of some intermediary `Just`.

Let's see this in action:
```
μ  divideBy 0 x = Nothing
μ  divideBy a x = Just (x / a)

μ  divideBy0 = divideBy 0
μ  divideBy1 = divideBy 1
μ  divideBy2 = divideBy 2
μ  divideBy3 = divideBy 3

μ  monadMap divideBy3 (monadMap divideBy2 (monadMap divideBy1 (Just 60)))
Just (10)
```
OK, success so far. Now let's divide by 0.
```
μ  monadMap divideBy2 (monadMap divideBy1 (monadMap divideBy0 (Just 60)))
Nothing
```
Since we got a `Nothing` somewhere in our chain of operations, `Nothing` is what we get as the result. You've created an automatic effect system, almost as though the type itself represents a sub-language within Mew. This simply isn't possible with your run-of-the-mill `functorMap`, because you can't run the control flow that would allow you to do this. The changes to the structure of the containing type are what makes monads so useful, because the structure itself, combined with a map, represent the added "effect" that monads promise.

Let's try another well-worn example within the world of monad tutorials: `Writer`.  `Writer` has some peculiarities that serve well to demonstrate the concept above. It is a type that serves as a wrapper for both another type `a` and a logging type `w`. Here's what it looks like concretely:

```
type Writer w a = Writer a w
```

We're interested in performing computations on type `a`, but we also want to write a log of what we're doing, and we store that log in `w`. For now, just assume that `w` will be `List String`, so that for every function we run over our `Writer`, we can add another entry to the list. 

---------------
† In general, though, `w` will be *any* type that can run `++` on. That's an "appending" operation from another interface called `Foldable`, and more types than `List` implement it. `String` does too, but concatenating logs to a single, giant string is pretty ugly.

-------------

 How would we write `functorMap` for `Writer`? Like this:

```
functorMap f (Writer a w) = Writer (f a) w
```
All at once the limits of functors become manifest. We simply cannot change our log `w` with `f`; `f` is of type `a -> b`. It can only operate on *one* of the contained types, and we've already chosen `a`. If, on the other hand, we'd run an `a -> f b`, we could. In that case, `f b` would be a fully-formed `Writer w a` that could produce a new log based on `a`.

It turns out that `w` is part of the structure of the value `Writer`. Recall that within the type signature of any map there is an `f a`—that is, a functor type containing some value. The `f` of `f a` here is actually not `Writer`. It's `Writer w`. The thing that is *changing*, that we are mapping on, is the `a` of `Writer`. There can only be one such type of value in any functor. In a `type` declaration, it's always the last type listed before the `=` sign. Notice how `w a` and `a w` are reversed on either side of the declaration of `Writer` above. It's an arbitrary (and heuristic) choice that we've selected `a` to be first on the right-hand side, since it's the "important" bit that we're operating on. On the left-hand side, however, it's necessary that the type that represents the value we want to map over be last, because everything that comes before it will form part of the structure of the functor.

Luckily, we now know that we can embed "effects" within a monad by changing this very structure via `monadMap`. What does `monadMap` look like for `Writer w a`?

```
monadMap f (Writer a w) = Writer a' (w ++ w')
  where Writer a' w' = f a
```
Clearly we're not just running `f` on `a` and returning the result as we did for `monadMap f (Just a) = f a`. We *do* run `f` on `a`, but as an intermediary result in line 2. We do this so that we can concatenate the old logging list `w`  with the new one, `w'`. Then we wrap our `a'` and our log concatenation back up in a fresh `Writer` and use this as the result. To be clear, this added machinery still matches the correct type signature of any monadic map. What we compute inside a monadic function—how we change the resulting structure, and therefore what kind of effects we produce—is largely up to us, as long as it obeys the general rules.

What can we do with it? Let's define a function to try on it:

```
multAndLog m x = Writer res [entry]
  where res = x * m
        entry = "Multiplied " ++ show x ++ " by " ++ show m ++ ". Result is " ++ res ++ "."
```
And then give it a whirl:
```
μ  monadMap (multAndLog 13) (Writer 7 ["Starting state. Result is 7."])
Writer 91 ["Starting state. Result is 7.", "Multiplied 7 by 13. Result is 91."]
```
It turns out it logs. Who knew?

## Operator, please

There's good news and there's bad news.

The bad news is that Mew doesn't actually have a function called `monadMap`. It's unwieldy. The good news is that it does have an equivalent function called `>>=`, or, in human speech, *bind*. Double good news: don't have to throw out anything we know about `monadMap`, because `>>=` is just `monadMap` with its parameters reversed. The semantics are otherwise identical, and the syntax from here on out will be much more convenient. For reference, the type signatures are:
```
monadMap : (a -> f b) -> f a -> f b
(>>=)    : f a -> (a -> f b) -> f b
```
Remember that ugly nest of divisions on `Maybe Int` above? Compare it below to its equivalent using `>>=`:

```
μ  monadMap divideBy2 (monadMap divideBy1 (monadMap divideBy0 (Just 1)))
Nothing
μ  Just 1 >>= divideBy 0 >>= divideBy 1 >>= divideBy 2
Nothing
```

Much nicer, no? In fact, the only reason we created functions like `divideBy0` is because the parentheses necessitated by using a curried `divideBy 0` in the `monadMap` version would make for painful parsing by humans. Did you notice they're replaced in the second version?

Heck, it even looks like a pipeline of some sort—or a boolean `AND` with a payload. Remember, novel operator aside, this is the exact same thing as before. First `divideBy 0` is "monadically mapped" onto `Just 1` (producing, of course, `Nothing`). Then `divideBy 1` is mapped to this result, followed finally `divideBy 2`.

Now do `Writer`:

```
μ  Writer 7 ["Starting state. Result is 7."] >>= multAndLog 13
Writer 91 ["Starting state. Result is 7.", "Multiplied 7 by 13. Result is 91."]
```
That's a little better, but we're probably not only going to want to log one thing. Let's clean that up and pack even more operations in there:
```
μ  init = Writer 1 ["Starting state. Result is 1."]
μ  init >>= multAndLog 13 >>= multAndLog 42 >>= multAndLog 90
```
I've moved the result **here** because it's quite long. But the getting there is nice. The parenthesizing required for `monadMap` would make you pull your hair out, but this is terse and readable. It's dense, yes—*very* dense, even—but we're not scared, because we understand the rules of how `>>=` maps its arguments.

## Toward a general monadic map

We've seen the `Functor` interface implemented, and we've mentioned that there's one for `Monad`, too. To this point, we've neglected a tiny detail about `Monad` for the sake of focusing on the bigger picture. It's *another* function we have to provide to `Monad` instances, and we're going to define it now:

```
interface Monad f where
  (>>=) : (f a) -> (a -> f b) -> f b
```
