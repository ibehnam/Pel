# Pel (Pipeline Execution Language)

<!--toc:start-->
- [Pel (Pipeline Execution Language)](#pel-pipeline-execution-language)
  - [Language Basics](#language-basics)
    - [Partial Lambdas](#partial-lambdas)
    - [Closure](#closure)
  - [Creating Lambdas](#creating-lambdas)
  - [`do` blocks](#do-blocks)
  - [Definitions](#definitions)
  - [Calling Lambdas](#calling-lambdas)
  - [Two Types of Lambdas](#two-types-of-lambdas)
  - [pipelines](#pipelines)
  - [basic data types](#basic-data-types)
    - [Booleans](#booleans)
    - [Nil (`#nil`)](#nil-nil)
    - [Numbers](#numbers)
    - [Strings](#strings)
    - [Symbols](#symbols)
    - [Keywords](#keywords)
    - [Literal Lists](#literal-lists)
      - [Slicing and Looking Up Keys in Literal Lists](#slicing-and-looking-up-keys-in-literal-lists)
  - [Control Structures](#control-structures)
    - [`for`](#for)
    - [`if`](#if)
    - [`case`](#case)
    - [`case`'s features](#cases-features)
<!--toc:end-->

## Language Basics

Pel interprets parentheses expressions (like `(foo 1 2 3)`) as `(operator operands...)` where "operator" is a lambda and "operands" are treated as its arguments. There are no exceptions to this rule; any time you see a parentheses expression, you are free to think about it in terms of calling a lambda with some arguments.

- "Functions" in Pel are just lambdas that are assigned to an identifier (a "Symbol"). That's why in this text we talk about lambdas and not functions. Anything discussed about lambdas also applies to functions.

### Partial Lambdas

Pel lets programmers easily create partial lambdas simply by __not__ providing all the required arguments. Say, a function `add` is defined with two arguments `:x` and `:y`. "Calling" `add` with only one argument would not cause any errors; it would just return a partially applied function that's awaiting one more argument to become "complete"/"full" and return a value.

```
(add 1) -> not an error; it's a partially applied function that receives an argument `:y` and adds it to 1.
((add 1) 2) -> returns 3
```

If any of the lambda arguments has a default value (set during lambda definition), then providing that argument is optional; if given, Pel substitutes the new value with the default value, otherwise it uses the default value for computing the lambda. For example, let's say `foo` has two arguments: `:name` and `:last_name #nil`. The second argument is given a default value (even though its value is `#nil`). Therefore, both `(foo :name "behnam")` and `(foo :name "behnam" :last_name "mohammadi")` are "complete" and return. That said, `(foo)` and `(foo :last_name "mohammadi")` are "incomplete" and return a partially applied function that awaits the argument `:name`.

### Closure

The fundamental internal building block of most Pel expressions is `Closure` which stores a lambda along with some meta data about it. This extra information is what lets Pel "know" if a lambda has become full and is ready to run. `Closure` also keeps track of the environment, always looking up symbols first in the child environment and then in parental environments if the lookup fails.

## Creating Lambdas

You can create a lambda as follows:

```
(lambda [:x :y 12 :z (add1 3)]
  (sum x y z))
```

Lambda arguments are written in brackets. Argument names are prefixed with a colon ":", which effectively makes them __keywords__ (see below for more details about Pel keywords). Each argument can have a default value. In the previous example, `:y` is set to 12 and `:z` is set to the result of the expression `(add1 3)`, which is 4.

- When referring to lambda arguments in the body, there's no need to write them as keywords!

## `do` blocks

To run a sequence of expressions after each other, you can use `do`, which is a function that runs each of its arguments and returns the value of the last one. This can be useful when creating lambdas, writing `for` loops, etc.

```
(lambda [:x :y]
  (do
    (print ["inputs are:" x y] " ")
    (print ["incrementing x:" (+ 1 x)] " ")
    (+ x y)
  )
  3 4)

-> "inputs are: 3 4"
   "incrementing x: 4"
   7
```

A few notes:

- Unlike Python, Pel's `print` function not only displays the result on screen, it also returns it.

- Pel values are __immutable__, therefore, `(+ 1 x)` simply returns `x + 1` and does not update `x` itself.

- In the example above, we immediately applied the lambda to two arguments `3` and `4`. When run, the lambda printed two strings and returned `7` (which we can use later).

## Definitions

Defining functions and constants (because Pel values are immutable) is done via `def`.

```
(def n 12) -> assigns symbol `n` to value 12
(def m (+ 3 4)) -> assigns symbol `m` to the expression `(+ 3 4)`
(def foo (lambda [:x :y 5]
  (print (+ x y)))) -> assigns symbol `foo` to a lambda expression which prints (and returns) the sum of its arguments

```

- Re-assigning the same symbol to a new value is not allowed.

```
(def n 12) -> ok
(def n 14) -> error
```

- `def` not only assigns a symbol to an expression, it also returns that expression as its result. This is a much more general version of Python's "walrus operator" `:=`:

```
(print (+ 2 (def n 12))) -> defines symbol `n` to value `12`, adds `2` to the result, and prints `14`
```

## Calling Lambdas

Pel allows positional arguments OR named arguments, but not a mix of both. That means you can call a lambda (or a function) as follows:

```
(foo :x 3 :y 4) -> named arguments; `:x` and `:y` are the names of the function arguments at the time of definitions
(foo 3 4) -> positional arguments
(foo :x 3 4) -> error
```

- Even Pel's control structures have named arguments. We will talk about them below, but just as an example, you can call a `for` loop in two ways:

```
(for [1 2 3 4] i (print i)) -> positional arguments
(for :coll [1 2 3 4] :ioterator i :body (print i)) -> named arguments
```

## Two Types of Lambdas

Pel lambdas (and functions) can be either strict or non-strict. Strict lambdas evaluate their arguments at the time of call. For example, calling `(print (+ 1 2))`, first the argument `(+ 1 2)` gets evaluated to `3`, and then `print` works with the evaluated value.

Non-strict lambdas, on the other hand, do no necessarily eagerly evaluate their arguments, which makes them "lazy". For example, `(def :name :value)` is a non-strict function that receives AST forms of its arguments, and later chooses to evaluate the `:value` only (because `:name` is a symbol that isn't bound yet).

Therefore, Pel provides an elegant and consistent approach to language design - one that does not special-case any forms (i.e., there are no special forms in Pel). This uniformity also allows for a new way of __meta programming__ without resorting to complex macros, which is on the roadmap for Pel.

## pipelines

pel deviates from lisp languages by providing a powerful pipeline syntax that "jumps" outside the parentheses. in most lisp implementations, one would expect something like this:

```clojure
(-> 5
    (inc)       ; 6
    (* 3)       ; 18
    (str " apples")) ; "18 apples"
```

which is fine as long as the user knows __in advance__ that they're going to pipe a value to a few functions, that is, when they have __planned__ the chain in advance. but users often make this decision __after__ they've already written some value code. if the user suddenly decides to call a function on the current value, he has to go all the way back to the opening parenthesis, write the "pipe" command, go back to the previous cursor position, write the function name, and repeat this for any

## basic data types

### Booleans

- `#t` and `#f` (true and false).

### Nil (`#nil`)

`#nil`, which can also be denoted by `()`, represents "no value", similar to `None` in Python. But unlike Python, Pel distinguishes between the absence of a value and it being false. Therefore, in Pel, `#nil` and `#f` are not equal.

```
(eq? #nil ()) -> #t
(eq? #nil #f) -> #f
```

### Numbers

- Integers or floats, e.g., `42` or `3.14`.

### Strings

- Text in double quotes: `"hello world"`.

- Single quotes are not allowed.

### Symbols

Identifiers such as `foo`, `do`, etc. Lookup in the current environment returns whatever value/binding is associated with that symbol.

### Keywords

- Symbols that start with ":" and evaluate to themselves, e.g., `:at`, `:x`, `:key`.

- Keys are commonly used as named function arguments or dictionary-like keys.

- Pel's parser automatically matches keywords in code with the expression that follows them, unless that expression is itself a keyword, a pipe, or a closing delimiter (`)` or `]`).

```
[:a 1 :b 2 :c 3 :d] -> Internally represented as [PelPair(key=PelKey(":a"), value=PelNum(1)), ..., PelPair(key=PelKey(":d"), value=#nil))]
```

### Literal Lists

Literal lists use brackets `[...]` instead of parentheses. Unlike Python, Pel allows literal lists to be heterogeneous, that is, you can put items of various types in a list: `[1 "and" #t :c d (print "hello world") :e]`. Since Pel's parser automatically "pairs" keys with their immediate values, dictionaries/maps/hash-tables are simply a special case of Pel literal lists in which all elements are Pel pairs. As the previous example shows, Pel allows for mixing dictionary-style values with others. When looking up a key in such a heterogeneous list, Pel only looks at elements that are indeed Pel pairs.

Unlike an expression list, literal lists evaluate all their items. We don't "lower" the result to Python lists (and frankly, it wouldn't make sense given our different take on what can go in literal lists); evaluated literal lists return literal lists but with all elements evaluated in the context (environment).

#### Slicing and Looking Up Keys in Literal Lists

It might be a surprise that literal lists are actually implemented as Closures! That is, a literal list is a closure with three arguments that have, by default, been set to empty `#nil`. These arguments are: `:at`, `:from`, and `:to`. When a literal list is "called" without call arguments, it simply returns all the elements. But supplying one (or more) of its arguments results in list slicing. Pel has one of the most advanced list slicing mechanisms in the world!

Some examples of Pel list slicing:

```
(1) ([5 6 7 8] :at 1) -> 6 (return element with index=1)
(1) ([5 6 7 8] :to 2) -> [5 6 7] (return all elements until (and including) index=2)
(2) ([5 6 7 8] :from 1) -> [6 7 8] (return all elements from (and including) index=1)
(3) ([5 6 7 8] :from 0 :to 2) -> [5 6 7] (self-explanatory)
(4) ([5 6 7 8] :at [0 3]) -> [5 8] (return multiple elements at indices=5,8; the output is a list)
(5) ([:a 1 :b 2 :c 3] :at ':a) -> 1 (lookup the value of a key ":a" in the literal list)
(6) ([:a 1 :b 2 :c 3] :at [:a :c]) -> [1 3] (lookup the values of multiple keys)
(7) ([:a 1 :b 2 :c 3]) :at [0 2]) -> [:a 1 :c 3] (return multiple elements at indices=0,2)
```

A few notes:

- In (5), we have to quote the key because Pel never pairs a key with a key that immediately follows it. Without the quoting, Pel would interpret `:at :a` as `PelPair(PelKey(":at"), #nil)` and `PelPair(PelKey(":a"), #nil)`.

- As (7) shows, Pel automatically sees the literal list elements as key-value pairs, so internally this is just `[PelPair(PelKey(":a"), PelNum(1)), ...]`. That's why we can slice it just like any other literal list. Extracting element with `index=0` simply returns the first Pel pair with both key and value.

- Obviously, we don't have to always mention the named arguments. Here are some examples:

```
([5 6 7 8] 1) -> 6 (since `:at` is the first argument, this is implicitly saying `:at 1`)
([5 6 7 8] () 1 3) -> [6 7 8] (the first positional argument is #nil, followed by 1 and 3, which are `:from` and `:to`, respectively)
```

- Since Pel literal lists are Closures, we can pipe into them as well!

```
(def data [1 2 3 4 5 6 7 8 9]
(for [0 2 4 6] i
  i |> (data :at ^) |> (print))
```

which prints elements at `index=0,2,4,6` in `data`.

## Control Structures

### `for`

```
(for :coll :iterator :body)
```

`for` receives an iterable (like a literal list), an iterator (like a symbol `i`), and a body expression. `for` is a Closure, so it can be used in pipelines as well:

```
(def data [1 2 3 4 5 6 7]) |> (for i (print i))
```

- `for` returns the last computed body value, therefore, the console prints `1 2 3 4 5 6 7 7`.

### `if`

```
(if :cond :then :else #nil)
```

By default, `if`'s `else` clause is `#nil`, so you can also use `if` with `then` clause only.

```
(if data |> (len) |> (gt 2)
  (print "len of data >= 2"))

; using in a pipeline
data |> (len) |> (gt 2) |> (if (print "data len >= 2"))

```

- Similar to `for`, `if` also returns the value of either `:then` or `:else`. If no `:else` branch is provided, `if` returns `#nil`.

### `case`

`case` is a more general version of `if` and is often the better choice, especially when dealing with conditions that are written in natural language! Here's how `case` looks like:

```
(case :scrut [
  <cond-1> <conseq-1>
  ...
  <cond-n> <conseq-n>
  #t       <default-consequ>
])
```

### `case`'s features

- `case` pipes its scrutinee (`:scrut`) into each of the conditions. Therefore, conditions can be expressed as lambdas, waiting for input(s).

- Any string condition is treated as a __natural language condition__. Pel uses LLMs in the background to process truthy/falsy-ness of such conditions.

```
(case [1 2 3 4 5 6] [
  "is an ascending list"
  (print "list is increasing")
  #t
  (print "list is probably not ascending")
])
```

- Note that `case` still "pipes" the literal list `[1 2 ... 6]` into the string condition `"is an ascending list"`, but since piping into complete closures has no effect, the result will be the string condition itself. This is also true when you pipe something into `#t`, keys, etc.

The previous `if` example can be written using `case` as well:

```
(case data [
  (len) |> (gt 2)
  (print "data len >= 2")

  #t
  #nil
])
```

Just like `if`, `case` returns the evaluated consequence value. An example in which `case` is used in a pipeline:

```
data |> (case [
  "len is greater than 2"
  ...
]) |> (print)
```

- `case` is often superior to `if` because it pipes scrutinee into conditions, can have more than two conditions, and supports natural language conditions.
