### Notes on convention

Mew tries, among other things, to be consistent with its conventions. Notably, all names are either `camelCase` or `PascalCase`. This extends to those names that include initialisms; otherwise, there's no way to delimit (and easily read) those initialisms in context. Thus, unlike Haskell, the type of effect-causing computation is `Io`,  not `IO`.

### Basic I/O operations

The behavior and types of some basic effect-causing I/O operations look like the following:

#### Input

**`gets : Io String`**

Reads a line from standard input as a Mew `String`.

#### Output

##### Strings
**`puts : String -> Io ()`**

Causes a `String` to be written to standard output.

**`putsLine : String -> Io ()`**

Causes a `String` and newline to be written to standard output.

##### Other structures

More generally, one can use the following functions, which are defined against any type that implements the type class `Show`:

**`put : a -> Io ()`**

Causes a type implementing class `Show` to be output. Defined simply as `put = show ~> puts`<sup>†</sup>.

**`putLine : a -> Io ()`**

Causes a type implementing class `Show` to be output with a newline, if none exists. Defined simply as `putLine = show ~> putsLine`<sup>†</sup>.

---
<sup>†</sup>The operator `~>` represents forward function composition. That is, `~>` takes two functions, each with a single parameter, and makes a new function out of them by glueing the output of the left-hand side function to the input of the right-hand side function.

To illustrate with absolute clarity, one can review the following sequence of identically-behaving definitions:
```
put x = puts (show x) 
put x = (show ~> puts) x
put = (show ~> puts)
```

### Comments

Comments in Mew look like this:

```
-- I am a single-line comment.

-_ I am an inline comment _-

-_
  I am a multi-line comment.
  Look at all the lines that I am.
  It is three.
_-
```

### Type signatures

Type signatures in Mew use `:` (as opposed to Haskell's `::`).

```
methuselah : Integer
methuselah = 969
```

### Types and data

To create a data type, Mew uses the keyword `type`:

```
type Magi = Melchior
          | Caspar
          | Balthazar
```

To create a type synonym, Mew uses the keyword `alias`:
```
alias Email = String
```

### Lists

List types in Mew are denoted like this:

```
list : List Integer
list = [1,2,3]
```

### Records

Records look a lot like object literals in JavaScript:

```

person = { name : String
         , age : Int
         }
```

### Type aliases

In Haskell,  PureScript, and other functional languages, type synonyms are denoted using the following syntax:

```
type Email = String
```

The idea is the same in Mew, but the syntax is different, and attempts to be less ambiguous:

```
alias Email = String
```
