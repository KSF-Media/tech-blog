Every Haskeller will eventually write at least one monad tutorial.
Some of them deny being monad tutorials while still doing it.  This is
not a monad tutorial.  Monads come with a bind operator and a return,
which allow composing and building monads, respectively.  Like a
stranger in the shadows, there is a third, silent member of that cast.
This is about it.

Consider the type class definition of Monad:

```
class Monad m where
  (>>=)  :: m a -> (  a -> m b) -> m b
  return ::   a                 -> m a
```

It gives you enough to build them and to combine them, but there is
one function conspicuous in its absence: `extract :: m a -> a`.

Let's start by fantasizing a bit.  Suppose they used Haskell in R'lyeh
and they had a madness inducing definition of type classes that
included it.  With this insanity, there would be a default definition
of bind available, with

```
a >>= f = return (f (extract a))
```

Even if a monad defined a non-default bind, this above function could
still be used.  No matter what monad you gave to it, you could get
equal results from them by using `extract` with `return`.  Monads
would lose all meaning and all of them would be in effect equal.

Enough of that, the cultists may be disappointed but luckily we don't
live in that world.  But there is still a need for that silent, third
partner, to make use of monads.  Without it you can only make them,
eternal and pure, but nothing more.  You need to add the breath of
life that gives motion to the golem, that something special that
distinguishes one monad from another.

The third part comes under different names but a common one is run.

```
newtype State s a = State { runState :: s -> (s, a) }
```

Taking the state monad as an example, the one line type definition
creates an accessor function `State s a -> s -> (s, a)`.  That takes a
state monad and breathes in the initial state `s` to use and returns
the final state along with a value.

```
newtype Identity a = Identity { runIdentity :: a }
```

The trivial identity monad provides another example with the accessor
`runIdentity :: Identity a -> a`.  There are more that fit the same
pattern, like `runCont :: Cont r a -> (a -> r) -> r`.  I'm not going
into the particulars of any of their details in the scope of this
article.

That's one pattern.  There's a number of monads that have a particular
set of operations available for each `m a` (like `get` and `put` for
the state monad) and one run function to put into motion whatever they
do to get a value.

Running a monad is the moment it gets its true semantics, but until
that moment they are little but lists of operations.  They are like an
abstract cup that has walls and a bottom but it's left to be chosen
what liquid to pour into them.  As an aside, free monads are a
formalisation of that concept.  They allow reifying a monad in that
suspended state they are in, to be given meaning at a later moment and
not necessarily with only one possible interpretation.  They make the
liquid concrete, so to speak.

Lists are another monad, but you don't talk about running them to get
a value.  Instead you reduce them, or fold them.  That is another
aspect of the silent third member, there is not necessarily just one
way for it to manifest but possibly multiple.  `foldr`, `foldl` or any
other specialized variation, even simple pattern matching on the
list's internal components, they are all ways of collapsing the
quantum of monad into a value.

Tutorials on monads readily delve in the mechanics of binding with
examples, but they may lead astray.  Binding is not about execution,
state or burritos or whatever else you have.  Instead, bind is just a
bind.  Running them is the part which gives it all those possible
interpretations.  A bind takes a monad and gives you a monad, running
it (in all its various forms) takes a monad and gives you a value.

As the final act, standing alone, is IO, impenetrably capsulated, but
ultimately there is an act for running it as well.  The substrate for
it is the run time system and the vulgar way to call it is to execute
a program.  Your program is a fold.
