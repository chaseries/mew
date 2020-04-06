# Folds

## From first principles

A fold is a way of reducing a larger data structure into a value. Since lists are the obvious thing to use, a first-pass example of writing a fold might look something like this:

```
fold f [] = empty
fold f (x:xs) = f x (fold f xs)
```
The mysterious `empty` above is a real Mew value exported by the default module `Basics`. We'll define it in a second; for now, just assume it's some value that works for this scenario.

The `fold` takes a function of two arguments, `f`, and a list, `l`. In the base case it returns  `empty`, but in general it takes the head of the list and passes it as the first argument to our binary function, and passes as the second argument the value we get from recursing over the rest of the list.

Simple? Well, maybe not, if you're not used to this sort of thing. Let's try making it concrete on `Integer` values, where `add = (+)`. We could also pass `(+)` directly (and idiomatically, we would), but for the sake of example, name passing might feel more familiar. Anyway:

```
 fold add [1,2,3]
   = add 1 (fold add [2,3])
   = add 1 (add 2 (fold add [3]))
   = add 1 (add 2 (add 3 (fold add [])))
   = add 1 (add 2 (add 3 empty)))
   = add 1 (add 2 (add 3 0)))
   = add 1 (add 2 3)
   = add 1 5
   = 6
```

Ah, there's `empty` again. Did you see it morph into `0`? For an `Integer` under addition, that's what it should be. In fact, for every type with an `empty`, `empty` represents that value which, when combined in a certain way with a second value, returns the second value unchanged. For a `String` it's empty string, or `""`, because "adding" a string with an empty string does nothing.

But wait: that means that there's a bug in our code. We defined `empty` as `0` because we were adding. What happens if we're multiplying? In that case, `empty` would logically be `1`, not `0`. Given the definition above, `fold multiply someList` will always return `0`, and that's almost certainly not what we want.

## Folding the right way

There are actually two ways around this. The first is to specify that we're working with summable integers, traditionally by wrapping `Integer` in a type called `Sum`, but to explore this method would require a bit of a digression from folding.

The second way is to introduce another parameter to our `fold` function. While it turns out that in many cases, as with strings, there's a sensible default for `empty`, in the case of any given set of integers, we actually need to hand-pick this value because our folding callback could reasonably be one of several things. Thus a more general `fold` would look like the following:

```
fold f a [] = a
fold f a (x:xs) = f a (fold f a xs)
```

Now we can do this:

```
μ fold (+) 0 [1,2,3,4,5]
15
μ fold (*) 1 [1,2,3,4,5]
120
```

In fact, replacing `empty` with such an argument, often called an *accumulator*, also affords us more flexibility. We could just as well write `fold (+) 10 [1,2,3,4,5]` if we're interested in seeing the result of summing the list with 10 rather than 0. (The output, of course, is `25`.)

Why spill so much ink on `empty`, then? For one, hopefully it helped build up an idea about the behavior and constraints of folding from a naïve approach. More importantly, `empty` is actually super useful, and you're going to see it a lot if you write Mew code. It's a real name exported by the default module `Basics`. In polymorphic functions, it's a stand-in for `""`, `0`, `1`, and any other identity element of any other type that supports it. These types are collectively called *monoids*, and indeed there are a fair amount of useful ones in *any* programming language, even if they aren't always identified as such. In Mew, they're part of the typeclass `Monoid`. To see it in action, open up the Mew REPL and enter `empty : String`. The output will be `""`. As a bonus, `import Data.Monoid` and check the output of `empty : Sum Integer` and `empty : Product Integer`.

## Folding the left way

If you're quick, you may have noticed there's an alternate way of handling this whole enterprise.

```
fold f a [] = a
fold f a (x:xs) = fold f (f a x) xs
```
There is a difference between these functions, and for certain considerations it can be quite meaningful. In particular, non-commutative operands won't be accumulated in the same way, and there are time and space differences as well.

In fact, they're both in common use for different applications, so they have different names in the standard library. Our definition for `fold` in the previous section is actually known as a *right fold*, while the one immediately above is a *left fold*. In Mew these are `foldRight` and `foldLeft`, respectively, and they're exported by the default module `Basics`.

To get an intuition for the naming, consider the full sequence of operations that a left fold performs:

```

foldLeft add 0 [1,2,3] 
  = foldLeft (add 0 1) [2,3]
  = foldLeft add (add (add 0 1) 2) [3]
  = foldLeft add (add (add (add 1 0) 2) 3) []
  = foldLeft add (add (add 1 2) 3) []
  = foldLeft add (add 3 3) []
  = foldLeft add 6 []
  = 6
```

In particular, the fourth line above contrasts with that of a right fold. Pull the guts out of both and examine them, and we see this:

```
add 1 (add 2 (add 3 0))     -- Right fold
add (add (add 0 1) 2) 3     -- Left fold

-- Or, equivalently:
1 + (2 + (3 + 0))           -- Right fold
((0 + 1) + 2) + 3           -- Left fold
```
The parenthesizing is exactly the opposite between them, so the order of evaluation is the opposite. For right folds, naturally, the rightmost part of the list is evaluated first (after recursion bottoms out). For left folds, the leftmost part of the list is evaluated first, so recursion doesn't actually have to end by the time we start producing values.
