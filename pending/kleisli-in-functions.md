In my [previous blog post](dot-dot-dollar), I discussed the
similarities and connections between function application, composition
and the identity function and bind, Kleisli arrow and return.  There
was yet one more connection that I left unmentioned: Functions have a
monad instance as well.

All monads are functors as well and the `fmap` instance they have is
function composition.  Among other things this allows adding one more
entry to the list of possible definitions for function application:

```
($) = fmap id
```

Let's take a quest.  If Kleisli arrow is like function composition for
monads, then what does using Kleisli arrow on functions do, given that
functions are monads?  Like I mentioned, fmap for functions is
composition, but let's examine it a bit more.

```
fmap :: Functor f => (b -> c) -> f b -> f c

instance Functor ((->) a) where
    fmap :: (b -> c) -> (a -> b) -> a -> c

(.) :: (b -> c) -> (a -> b) -> a -> c
```

The first parameter of fmap is already the type you'd expect in
composition.  A function's type can be written in prefix notation as
`((->) a b)`.  Leaving one type variable out from that gives something
that has the kind `* -> *`, which is what Functor expects from its
parameter.  See for yourself with ghci if you like (you need to enable
a GHC extension for using the "forall" syntax).

```
> :set -XRankNTypes
> :k forall r. ((->) r)
forall r. ((->) r) :: * -> *
```

The functor instance ties the return type of a function, so writing `f
a` means it's a function that returns an "`a`".  The variable naming
would be a bit different in the standard defininition of fmap but I
renamed them to match with the ones composition has.  The actual
definition of fmap for the instance is pretty boring, once the types
have been sorted out: `fmap = (.)`.

Since functions are monads they are applicative functors as well.

```
(<*>) :: Applicative f => f (a -> b) -> f a -> f b

instance Applicative ((->) r) where
    pure :: a -> r -> a
    pure = const
    (<*>) :: (r -> a -> b) -> (r -> a) -> r -> b 
    (<*>) f g x = f x (g x)
```

The first parameter to `<*>` becomes a function with two parameters.
The "`a -> b`" comes from the usual type of `<*>` and the first
parameter `r` gets added from the function instance.  I've amended the
definitions with the types when specialized to the function instance.
One example of its use would be with the `note` function.

```
note :: a -> Maybe b -> Either a b
```

The semantics should be simple enough (besides the trivial `const
. Left`): Give out a Left with "a" if the second parameter is Nothing,
otherwise give Right.

```
(note <*>) :: (a -> Maybe b) -> a -> Either a b
```

This turns `note` into a function that can check on "a" and either
give some Right result based on it or give out the original value as a
Left.

This is a rare occasion of seeing a `<*>` in the wild without having
it preceded by an fmap first.  It works in this case since there's no
need to lift a function to some functor since all you need in this
case to get the applicatives engine primed is a function on its own.
Of course, you can use the style starting with fmap with function
functor as well.

```
data SomeData = SomeData
  { fieldOne :: String
  , fieldTwo :: Int
  , fieldThree :: ByteString
  }

buildTuple :: SomeData -> (String, Int, ByteString)
buildTuple = (,,)
  <$> fieldOne
  <*> fieldTwo
  <*> fieldThree
```

This uses `SomeData`'s accessor functions to get each value in turn.
The first field of the tuple can be filled with plain composition and
applicative gives access to the other fields.  The general rule of
thumb is that if you'd start with "`<$> id`" then you can omit that and
directly go to use `<*>`.

On to the monad instance for functions.  I've reversed the bind from
the standard definition to better match with the applicative instance
(and with function applications, where this all started).

```
(=<<) :: Monad m => (a -> m b) -> m a -> m b

instance Monad ((->) r) where
    (=<<) :: (a -> r -> b) -> (r -> a) -> r -> b
    (=<<) f g x = f (g x) x
```

Bind is almost like the sequential application from applicatives, but
now the insertion of `r` to it happens in the middle of the first
parameter, not on the outside of it.  Both the monad and applicative
instances take two functions and a value and use the value once and a
second time via the second function, but they vary the position where
it is done.

Using do notation with functions is just a way of defining multiple
functions which all take the same parameter.  `doThings 1` would
return `(5, "4", 4.0)`.

```
doThings :: Int -> (Int, String, Double)
doThings = do
    a <- (+1) . (*4)
    b <- show
    c <- fromIntegral
    return (a, b, c)
```

A key difference that sets applicatives apart from monads is that bind
allows using values through multiple binds but applicatives restrict
their use to one scope.  The way how that works with function functor
and monad, when you specialize them to `(<*>) f g x = f x (g x)` and
`(=<<) f g x = f (g x) x` becomes apparent when you chain more than
one of them together.

```
applicativeStuff :: (r -> a -> b -> c) -> (r -> a) -> (r -> b) -> r -> c
applicativeStuff f g h = f <*> g <*> h

monadicStuff :: (a -> r -> b) -> (b -> r -> c) -> (c -> r -> d) -> a -> r -> d
monadicStuff f g h x = h =<< g =<< f x
```

With the applicative function, `g` and `h` only ever see `r` and can't
do anything with `b` or `a`, respectively.  With the function using
binds, `g` could get `a` trivially if `b` was a tuple containing it
and likewise with `a` and `b` and the function `h`.

Or as another perspective, the equivalent functions without any of the
applicative or monad syntax would be the following:

```
applicativeStuff' f g h x = f x (g x) (h x)
monadicStuff' f g h y x = h (g (f y x) x) x
```

Monadic bind, join and the Kleisli arrows form a trinity where each
can be expressed with one of the others.  Join is the simple one.  If
`m a` is a function `r -> a` then `m (m a)` would be a function `r ->
r -> a`.  Thus the type of join, when specialized to the function
monad is `(r -> r -> a) -> r -> a`.  It could be defined as `join f x
= f x x`.

Finally, the Kleisli arrow, when specialized to function monad, has
the type and definition

```
(<=<) :: (b -> r -> c) -> (a -> r -> b) -> a -> r -> c
(<=<) f g x y = f (g x y) y
```

Compare it to the definition of composition:

```
(.) :: (b -> c) -> (a -> b) -> a -> c
f . g x = f (g x)
```

The answer, finally, is that Kleisli arrow for the function monad is
like composition, but every function in it gains an extra parameter
that gets the same value in each case.  One thing that makes it (even
more) impractical is that the parameter order is reversed from what
you'd usually expect from something like that, given the way currying
interacts with [parameter order](on-function-parameter-order).

Kleisli arrow on functions may be a bit too special to find practical
uses, but I could see them for all of the other functions, though they
may come as a surprise for the unaware.  If nothing else, the
instances exist because they work.  I didn't go through them but they
follow all the monadic, applicative and functor laws.

I'll finish this with an excercise.  Figure out what these functions
do:

```
fun1 f = (,) =<< f
fun2 f = (,) <*> f
fun3 f g = (,) =<< f =<< g
fun4 f g = (,,) <*> f <*> g
```
