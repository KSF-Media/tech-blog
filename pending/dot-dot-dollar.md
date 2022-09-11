The more you look at things the samer they look.  A fundamental
function in Haskell is function composition:

```
(.) :: (b -> c) -> (a -> b) -> a -> c
f . g x = f (g x)
```

This type definition follows the standard way of denoting function
parameters, but an alternate but isomorphic form may emphasize better
what it is about:

```
(.) :: (b -> c) -> (a -> b) -> (a -> c)
```

The usual way of thinking about function composition is that it takes
two functions and returns a function.  The standard way of writing out
the type reads out as "take three parameters, where the two first are
functions and the third is a value."

```
\f g x -> (f . g) x
```

This is function composition, with the three parameters it uses placed
explicitly as variables into a lambda.  Since `x` is dangling on the
right side on both sides of the arrow, it can be eta reduced into:

```
\f g -> f . g
```

This is isomorphic to the first version, but if you use the mundane
understanding of amount of variables in a lambda representing the
equalling the number of arguments to a function, it has lost one in
the process.  Of course it didn't really, but this is just
interpreting the composition as a function that returns a function.

Identity is another fundamental function:

```
id :: a -> a
id a = a
```

It does nothing.  Really.  But using it with composition gives you
another function that should also be familiar to you:

```
($) :: (a -> b) -> a -> b
($) = (id .)
```

Using identity as an argument to composition gives you the application
function.  But this is not all.  The following is a second
non-standard definition of function application:

```
($) = id
```

Really.  If you have ever looked at function application and had a
feeling that it isn't really doing anything, you were not wrong.  This
has to do with the alternate way of writing application's type as `($)
:: (a -> b) -> (a -> b)`.  If you think of it as a function that takes
a function and returns an identical function, then you can see that
`id` is something that can fit the role.  Of course, the usual
definition of function application is just plain `f $ x = f x`.

The first place where you'd see applications used in any Haskell code
is to replace parentheses with them.  Like this:

```
foo x = show $ (+1) $ (*2) x
```

This could equivalently be written as:

```
foo x = show ((+1) ((*2) x))
```

There's a curious connection between composition and application.  A
third, isomorphic form for writing this function could swap the first
application with a composition instead.  Look, nothing up my sleeve:

```
foo x = show . (+1) $ (*2) x
```

This bears taking a closer look.  Both `$` and `.` are right
associative, but `$` has a lower precedence, meaning that "show
. (+1)" gets resolved first, resulting in a function that takes one
parameter, which I'll call "f" (type `Num a => a -> String`) for the
rest of this section.  Substituting it to foo gives `f $ (*2) x`.  The
rest follows with doubling x and by giving that result to `f`.
Function composition is an associative operation, which means that it
doesn't matter which one you do first in something like `f . g . h`.

With the version that only uses application, the substitutions start
from the end to make values to give to each function in turn.  Using
composition made a short cut to only use application to the function
built with compositions.

I made the distinction to not call this evaluation.  If Haskell used
strict evaluation, the order I described would most likely be coupled
with evaluation order but Haskell is a lazy language which means that
the compiler is free to decide on the evaluation order on its own.
Haskell is, strictly speaking (sorry for the pun), a [non-strict
language](https://wiki.haskell.org/Lazy_vs._non-strict).

Trailing variables like in the above are really asking to be eta
reduced.  Remove the "x" from both sides of "foo" and change
applications to compositions and you'll get what you'd usually
consider the most idiosyncratic code for foo:

```
foo = show . (+1) . (*2)
```

## Chains of applications and compositions

In summary, chains of function applications can have any number
applications replaced with compositions starting from the left end,
including the final one if there's a variable that can be eta reduced.

Let's try to build a bit of intuition of what happens if you have
chains that mix compositions and applications.  Recall that function
composition has a higher precedence that application.  If you have a
mix, read it like every section with just compositions can be
collapsed into a single function.  Something like

```
f . f' $ g . g' . g'' $ h . h'
```

could be written like

```
f_ $ g_ $ h_
  where
    f_ = f . f'
    g_ = g . g' . g''
    h_ = h . h'
```

Another key to understanding it if you see code that uses a mix of
compositions and applications is that functions can take functions as
parameters.  If you are using compositions then you are working with
functions and if you have some on the right hand side of an
application dollar sign, it just means that the left side is something
that takes a function as its first parameter.  It's not called
functional programming for nothing.

## Taking it to monads

The connections don't stop there.  Let's put the types of the
function's we've examined so far together once more:

```
id  ::  a -> a
($) :: (a -> b) ->  a -> b
(.) :: (b -> c) -> (a -> b) -> a -> c
```

There's a quintuplet of Haskell functions that should look familiar:

```
return :: Monad m   =>      a  -> m a
join   :: Monad m   => m (m a) -> m a
fmap   :: Functor f =>     (a  ->   b) ->  f a -> f b
(=<<)  :: Monad m   =>     (a  -> m b) ->  m a -> m b
(<=<)  :: Monad m   =>     (b  -> m c) -> (  a -> m b) -> a -> m c
```

The parallel with the identity function bifurcates at this point to
return and join.  Of them, return is closer to the spirit of identity
since it can't really do anything since it needs to conjure up a new
monad out of nothing, but it's anybody's guess what would transpire
with `join` when you have a monad that someone has used already for
who knows what.

I'm not discussing `fmap` further in this context, it's only included
for completeness' sake.  It's not a coincidence that it's infix symbol
`<$>` is so close to function application's `$`.

`=<<` is called bind and `<=<` is Kleisli arrow.  Both have reversed
versions as well but I'm using these to match the order that
application and composition use.  Lifting things into monads adds more
to the definitions but the general pattern stays the same.  This even
carries over to allow a possible definition of bind with using
identity and the Kleisli arrow: `(=<<) = (<=< id)`.  This works
because using `id` substitutes `a` in the Kleisli arrow with `m b` and
the rest follows from there.  In a sense, using identity function is
telling Haskell that these types are equal.

Another curious thing happens if you use bind with identity:

```
join :: Monad m => m (m a) -> m a
join = (id =<<)
```

Doing the corresponding definition with application like `id' = (id
$)` is isomorphic to plain identity function but doing the same in the
monadic world gives you `join` instead.  Furthermore, combine this
with the definition of bind from above and you'll get this curiosity:

```
join = id <=< id
```

If you did the same things but substituted application and composition
in place, you'd only get two isomorphic definitions of identity.

Like with function application and composition, bind can sometimes be
changed to Kleisli arrow.

```
doStuff x = doThis =<< doThat x
doStuff'  = doThis <=< doThat
```

As a final example, return and Kleisli arrow act symmetrically to
identity and composition:

```
id2 :: a -> a
id2 = id . id

return2 :: Monad m => a -> m a
return2 = return <=< return
```

For any monad that follows the monad laws, `return2` is isomorphic to
the usual return.
