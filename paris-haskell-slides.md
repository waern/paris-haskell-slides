# About me

- I'm Swedish
    * Sorry, talk in English!

- Working as a programmer at LexiFi
    * OCaml programming for the finance industry

- Haskeller since 2002
    * Struggled the first year
    * Then it suddenly clicked and I've been hooked ever since

- Co-maintainer of Haddock
    * Ported it to use the GHC API instead of its own frontend
    * Currently maintaining it together with Simon Hengel
    * Not as much time for Haddock nowadays (help wanted) 

# The point of Haskell

- What's the point of Haskell?

- Being pure!

    * Purity is the single best thing about Haskell

- In this short talk I'll try to argue why it's such a great thing

# A note for beginners

- Forget about fancy sounding things such as:
    * Monoids, Applicative Functors, Arrows, Iteratees, Zippers,
      GADTS, Type Families
    * Atleast if you're completely new to the language

- Instead, focus on the core concepts:
    * the syntax
    * types
    * the prelude
    * strive for simplicity (don't use the fancy stuff)
    * read early Haskell papers (parsing combinators, etc)
  
# Benefits of purity

- Purity enables equational reasoning:
    * "The equal sign really means equality!"
  
- This is useful in everyday programming when:
    * Convincing yourself that your code works
    * Refactoring without accidents

- It's also great for tooling:
    * Compiler optimizations
    * "lint"-tools (e.g. hlint)
    * Awesome future tools

- Treating effectful computations as first-class values

- Parallelism (I won't go into that)

# Everyday programming: example 1

You see the same expression in multiple places:

```
let x = 10 + a * z + f 3 in
do_something ();
let y = 10 + a * z + f 3 in
print (x + y)
```

You want to re-factor:

```
let x = 10 + a * z + f 3 in
do_something ();
print (x + x)
```

But what if `f` read from the database and `do_something` wrote to the
database, and you didn't check?

In Haskell you are not tempted to do the wrong refactoring:

```
b <- f 3
let x = 10 + a * z + b in
do_something ();
d <- f 3
let y = 10 + a * z + d in
print (x + y)
```

# Everyday programming: example 2 (a bit contrived)

```
time : (() -> ()) -> () -> ()
```

`time` records how much time a function takes to run
and records the running average somewhere.

We use it to time a callback function:

```
let f = time (fun () -> ... do something ... ) in
on_clicked button1 f;
on_clicked button2 f
```

Let's say we want the callback to do something with the button that
was clicked. We naively add a parameter to `f`:

```
let f but = time (fun () -> ... do something with but  ... ) in
on_clicked button1 (f button1);
on_clicked button2 (f button2)
```

But the function returned by `time` keeps track of the running average
in a closure, and now it is created twice, thus timing the invocations
of the function from one button separately from the other.

# In Haskell

```
time :: IO () -> IO (IO ())

f <- time (... do something ...)

on_changed button1 f
on_changed button2 f
```

`f` is not defined using `let`, so we can not simply add a
parameter. Easier to find the correct solution directly:

```
time :: (a -> IO ()) -> IO (a -> IO ()) 

f <- time (\but -> ... do something with but ...)
on_changed button1 (f button1)
on_changed button2 (f button2)
```

# Tooling example: HLint

"lint"-like tools such as HLint suggest changes to the source code
that makes it simpler, easier to read and more idiomatic. 

HLint can make such a suggestion:

```
Found
  f x = g () x
Why not
  f = g ()
```

In an impure language this may be a bad suggestion if `g` is defined like so:

```
let g () =
  let y = do_some_effect () in
  fun x -> x + y
```

because the effect would happen when `f` is defined rather than when it
is called.

In theory tools such as HLint should work better in a pure language.

# A bit more about HLint

Here are some more suggestions by HLint:

```
Found
  and (map isDigit $ take 14 d)
Why not
  all isDigit (take 14 d)
```

```
Found
  do x
Why not
  x
```

```
Found
  not (new == old)
Why not
  new /= old
```

(Note that these and many suggestions by HLint don't have anything to
do with purity)

HLint is a great tool. Highly recommended if you're starting out in
Haskell.
  * http://community.haskell.org/~ndm/hlint/

Run it as part of your build process!

# Tooling example: compiler optimizations

Purity is key to many built-in GHC optimizations.

Example, "let floating":

```
let x = expr + 1 in
case z of
  [] -> x*x
  (p::ps) -> 1

==>

case z of
  [] -> let x = expr + 1 in x*x
  (p::ps) -> 1
```

Avoids allocating for `x` when it's not needed.

# Tooling example: rewrite rules

Purity is also necessary to support GHC's rewrite rules:

Example, list fusion:

```
{-# RULES
  "map/map"    forall f g xs.  map f (map g xs) = map (f.g) xs
#-}
```

- Here we're telling GHC that it can rewrite occurrences of `map f (map
g xs)` to `map (f.g) xs`.

- Depends on `f` and `g` being pure (otherwise the order of their
  effects may change).

# Tooling: future possibilities

We have only started making use of purity in programming tools.
Imagine this future scenario: 

You write:

```
  if (flag && f x) || (not flag && not (f x)) then
```

The future IDE or vim/emacs-plugin (or HLint) suggests:

```
  if flag == f x then  
```

- It knows `f` has no side-effect. In fact it only suggests
transformations that don't alter the meaning of the program.

- Therefore, you decide to trust the tool and accept the suggested code
with a press of a button.

# Proofs and program calculation

- Equational reasoning can be used to

    * prove general properties and laws of programs (e.g type class laws)
    * prove a program semantically equal to some other simpler program
    * derive an equal but more efficient version of a program

- The latter use of equational reasoning is called "program
  calculation" and is a beatiful way of designing algorithms,
  advocated by Richard Bird:

    * http://www.cs.ox.ac.uk/richard.bird/
  
# Downsides of (Haskell's) purity

- "Monad creep"
- Duplicate functions (map/mapM)
