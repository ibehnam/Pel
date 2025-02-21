# Pel (Pipeline Execution Language)

## Control Structures

### for

```
(for :coll :iterator :body)
```

`for` receives an iterable (like a literal list), an iterator (like a symbol `i`), and a body expression. `for` is a Closure, so it can be used in pipelines as well:

```
(def data [1 2 3 4 5 6 7])
data |> (for i (print i))
```

* `for` returns the last computed body value, therefore, the console prints `1 2 3 4 5 6 7 7`.

### if

```
(if :cond :then :else NIL)
```

By default, `if`'s `else` clause is NIL, so you can also use `if` with `then` clause only.

```
(if data |> (len) |> (gt 2)
  (print "len of data >= 2"))

; using in a pipeline
data |> (len) |> (gt 2) |> (if (print "data len >= 2"))

```

* Similar to `for`, `if` also returns the value of either `:then` or `:else`. If no `:else` branch is provided, `if` returns NIL.

### case

`case` is a more general version of `if` and is often the better choice, especially when dealing with conditions that are written in natural language! Here's how `case` looks like:

```
(case :scrut [
  <cond-1> <conseq-1>
  ...
  <cond-n> <conseq-n>
  #t       <default-consequ>
])
```

### case's features

* `case` pipes its scrutinee (`:scrut`) into each of the conditions. Therefore, conditions can be expressed as lambdas, waiting for input(s).

* Any string condition is treated as a __natural language condition__. Pel uses LLMs in the background to process truthy/falsy-ness of such conditions.

```
(case [1 2 3 4 5 6] [
  "is an ascending list"
  (print "list is increasing")
  #t
  (print "list is probably not ascending")
])
```

* Note that `case` still "pipes" the literal list `[1 2 ... 6]` into the string condition `"is an ascending list"`, but since piping into complete closures has no effect, the result will be the string condition itself. This is also true when you pipe something into `#t`, keys, etc.

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

* `case` is often superior to `if` because it pipes scrutinee into conditions, can have more than two conditions, and supports natural language conditions.

## Literal Lists

Literal lists use brackets `[...]` instead of parentheses. Unlike Python, Pel allows literal lists to be heterogeneous, that is, you can put items of various types in a list: `[1 "and" #t :c d (print "hello world") :e]`. Since Pel's parser automatically "pairs" keys with their immediate values, the previous example is internally represented as: `[... PelPair(key=PelKey(":c"), value=PelSymbol("d")) ...]`. This means dictionaries/maps/hash-tables are simply a special case of Pel literal lists in which all elements are Pel pairs. As the previous example shows, Pel allows for mixing dictionary-style values with others. When looking up a key in such a heterogeneous list, Pel only looks at elements that are indeed Pel pairs.

Unlike an expression list, literal lists evaluate all their items. We don't "lower" the result to Python lists (and frankly, it wouldn't make sense given our different take on what can go in literal lists); evaluated literal lists return literal lists but with all elements evaluated in the context (environment).

It might be a surprise that literal lists are actually implemented as Closures! That is, a literal list is a closure with three arguments that have, by default, been set to empty `NIL`. These arguments are: `:at`, `:from`, and `:to`. When a literal list is "called" without call arguments, it simply returns all the elements. But supplying one (or more) of its arguments results in list slicing. Pel has one of the most advanced list slicing mechanisms in the world!

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

* In (5), we have to quote the key because Pel never pairs a key with a key that immediately follows it. Without the quoting, Pel would interpret `:at :a` as `PelPair(PelKey(":at"),NIL)` and `PelPair(PelKey(":a"),NIL)`.

* As (7) shows, Pel automatically sees the literal list elements as key-value pairs, so internally this is just `[PelPair(PelKey(":a"), PelNum(1)), ...]`. That's why we can slice it just like any other literal list. Extracting element with `index=0` simply returns the first Pel pair with both key and value.

* Obviously, we don't have to always mention the named arguments. Here are some examples:

```
([5 6 7 8] 1) -> 6 (since `:at` is the first argument, this is implicitly saying `:at 1`)
([5 6 7 8] () 1 3) -> [6 7 8] (the first positional argument is NIL, followed by 1 and 3, which are `:from` and `:to`, respectively)
```

* Since Pel literal lists are Closures, we can pipe into them as well!

```
(def data [1 2 3 4 5 6 7 8 9]
(for [0 2 4 6] i
  i |> (data :at ^) |> (print))
```

which prints elements at `index=0,2,4,6` in `data`.
