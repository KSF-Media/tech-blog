In functional programming, it is natural to take one function and turn
it into another function.  That's what functions being first class
values means.  You take them, you pass them along, you transform them
and finally apply values to them to get more values.  The basic form
of such functions transforming functions is:

```
f :: (a -> b) -> a -> b
```

It's the function application, yet again.  I'll take a concrete
example of this form:

```
increaseAfter :: (String -> Int) -> String -> Int
increaseAfter f n = (+1) $ f n
```

Taking `increaseAfter read "1"` has value 2 as you'd expect.  Taking
it a step further, `(increaseAfter . increaseAfter) read "1"` has
value 3.  So far so good, but if you look at the function's
definition, it is pretty formulaic and repetitive with the variables.
There is a thing you can do about that.  Functions are functors and
using them as functors means using `fmap` to manipulate their return
value.  Using that, you could write the definition like this:

```
increaseAfter = fmap (+1)
```

No more variables and even the function application went away.  On its
own this may look like a parlor trick, but one way to think of this is
that using the function functor is a bit like using the `snd` function
on a 2-tuple.  In fact, 2-tuples are functors themselves and using
`fmap` on them means manipulating the second element.

On to "`fst`" for functions, but things will have to take a different
turn since now it's about input and no longer about output.  Reverse
polarity and trek on.

```
increaseBefore :: (Int -> String) -> Int -> String
increaseBefore f n = f $ n + 1
```

For example, `increaseBefore show 1` gives you "2" and so on.  Again,
the same question.  There's repetition and variables that are used in
a seemingly trivial manner.  What's the abstraction this time?  It
can't be functor, anymore, since it would touch the "`String`" part
and the aim is to do something about the `Int`.

The answer is functor, but a contravariant one, but to work on this
example, we have to jump all the way to profunctors as well, though
that's enforced more by Haskell's type system than function's
properties in the abstract.

```
fmap :: Functor f => (a -> b) -> f a -> f b
contramap :: Contravariant f => (b -> a) -> f a -> f b
```

Functions don't implement `contramap` in Haskell, but that has mostly
to do with how `(->) r` syntax doesn't really allow for having the
parameter in the position it'd require.  It'd be a theoretical can of
worms how to even possibly allow that but it can be sidestepped by
using the profunctor instance instead.

```
dimap :: Profunctor p => (a -> b) -> (c -> d) -> p b c -> p a d
rmap :: Profunctor p => (b -> c) -> p a b -> p a c
lmap :: Profunctor p => (a -> b) -> p b c -> p a c

rmap = dimap id
lmap = flip dimap id 
```

Profunctor is a covariant and contravariant functor together, with
`lmap` corresponding to `contramap` and `rmap` to `fmap`.  If `dimap`
is defined then the default definitions for `rmap` and `lmap` are
available.  The function instance for `dimap` is simple enough:

```
dimap f h g = f . g . h
```

I'm not going to examine profunctors further at this time, but it's
sufficient to get to grab `lmap` from it and continue the example with
it.  So, on to how to manipulate the "`Int`".

```
increaseBefore = lmap (+1)
```

Success.  I'll take a few words to motivate this all.  It's not code
golf to get to write a bit less.  Functors, in either variety, are
structure preserving.  When used on functions like that, you can tell
just from seeing them in use that we are preserving other parts of
them but there's going to be one part that's going to be transformed.
You get to remove all of that from the view and instead focus on just
writing the `a -> b` part of it.  Or in these examples, that was even
simpler since it used an endofunction but it wouldn't have to be.

I'll finish this with an example that uses the concepts I introduced
together.  `IORef` is a mutable container of values.

```
modifyIORef :: IORef a -> (a -> a) -> IO ()
```

This function takes an `IORef`, a function that modifies it in place
and finally an `IO ()` to denote that it's doing it just for the side
effect's sake.  Let's assume that we're working on a fixed `IORef`
with a fixed reference, so we can simplify things a bit:

```
data Thing = Thing
  { flag :: Bool
  , warning :: Bool
  }

ref :: IORef Thing

modifyIO :: (Thing -> Thing) -> IO ()
```

Let's make a rule: If `flag` was `True` then we require that `warning`
is always set `True` as well.  How to write a definition for
`modifyIO` that ensures that condition?

```
enforce :: Thing -> Thing
enforce x = x { warning = warning x || flag x }

modifyIO :: (Thing -> Thing) -> IO ()
modifyIO f = modifyIORef ref $ enforce . f
```

In this case, it's not too bad even as is, but let's still apply what
we know to see how it goes.

```
modifyIO = (lmap . rmap) enforce $ modifyIORef ref
```

Start with `lmap` to focus on the input and then `rmap` to apply
`enforce` to the output.  This needs some attention since the
parameter is an endofunction so `lmap . lmap` would pass type check as
well but it wouldn't have the correct semantics.  To wrap this up,
recall that all functions in Haskell take one parameter, some of them
just return new functions.  Using that, we can even write a
`modifyIOAnyThingRef` that enforces the rule we set for any `Thing`s.

```
modifyIOAnyThingRef :: IORef Thing -> (Thing -> Thing) -> IO ()
modifyIOAnyThingRef = (rmap . lmap . rmap) enforce modifyIORef
```

Using `rmap` first on the function `modifyIORef` lets us modify the
output of it, namely the `(Thing -> Thing) -> IO ()` part.  The rest
follows like in the first function.
