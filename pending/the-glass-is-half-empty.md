Less is more, as the saying goes.  In programming, it's common routine
to do clean ups like removing unused function parameters or module
imports.  But there is a lot more to the principle than just that.

First off, some preliminaries.  Let's have a simple sum type to
demonstrate a few concepts.

```
data Color = Red | Green | Blue
```

Let's consider the humble identity function.

```
id :: a -> a
id x = x
```

Can't get any more minimal than that.  It has a very curious property:
Given the type `a -> a`, it's already possible to know from it alone
that there is just one possible definition for such functions (up to
isomorphism).  With nothing else known about `a`, if it is given one
then the only thing it can do with it is to return it, as is.  Let's
see what happens if we start making changes to the identity function.

```
anythingToColor :: a -> Color
```

This function still knows nothing about `a`, but now it's required to
produce a number.  Since it can't use `a` for anything, the only thing
it can do is to return a color.  Since we're talking about pure
functions it has to be constant.  That means that there are three
possible `anythingToColor` definitions.

```
colorToAnything :: Color -> a
```

The other way around, we're in trouble.  The function is given a
`Color` but is required to produce an `a`.  Since it doesn't know
anything about `a`, the `Color` it has is useless for it.  There are
zero possible `colorToAnything` functions.

The general principle that can be drawn from this example is that
making the return value more restricted expands the space for possible
implementations and restricting inputs limits the space.  The examples
I used so far are downright banal, but the same applies in general as
well.  Designing programs straddles between these two forces: You want
to be able to express things but putting limits on what can be done is
benefical, since the fewer possible combinations there are for the
pieces available in any situation are the more likely it is to have
the correct one.  It's rare to have such luck to have only one
possible solution.

More formally, the different roles that the output and input of the
identity function have are called *covariance* and *contravariance*,
respectively.  `id` is covariant in its return type and contravariant
in its input type.  The general rule of thumb is that covariance has
to do with outputs and contravariance with inputs.

A bit more involved example:

```
foldr :: Foldable t => (a -> b -> b) -> b -> t a -> b
```

I'll omit the definition.  In higher order functions, the notions of
covariance and contravariance get flipped each time a function is used
as a parameter.  The function parameter `a -> b -> b` is covariant in
its parameter `a -> b` and its return value `b` is contravariant.  In
that sense, the parameters to the function in `foldr`'s first
arguments are outputs for `foldr` and its return value is input for
`foldr`.  Substituting `+` and `-` minus for the parameters gives

```
(+ -> + -> -) -> - -> - -> +
```

Having even higher orders would flip the variance again.  One
consequence of this all is that everything labeled with a minus is
something that you can use when you use `foldr` and everything with a
plus is a requirement.

The substitutions I've done for the examples so far aren't actually
valid for Haskell, since the same type variable `a` is used for both
sides for `a -> a`.  I can introduce new functions that look similar
but they are all new functions and can not specialization of `id`
itself.  The way how `a` is used in `id` makes it *invariant*, since
it's used in both contravariant and covariant positions.  Likewise,
`b` is invariant for `foldr`.

How do type classes fit in with this?  Like with previous
substitutions of `a` with `Color`, going from `a` to `Foldable t => t
a` is going to a more specific thing.  If you look at it this way,
having a function's return type to be `IO Color` instead of `Color`
tells you that the function is going to be able to do a lot more.
Likewise, `IO Color` in a parameter means that there's less that a
function can do since it's going to be constrained by what `IO`
requires in order to use the input variable.  If the return type has
no `IO` in it you'd know that the parameter can never be used for
anything.

```
($) :: (a -> b) -> a -> b
```

Function application is another function where there is only one
possible implementation.  The only way for it to get a `b` is to use
apply the second parameter to the first parameter, that is: `f $ a = f
a`. Let's consider the `a -> b` in the first parameter and see what it
means for implementing `($)` if we replaced either side of it.  `a` in
this place is contravariant, which would mean that replacing it with
`Color` should make it possible to make more apply-like functions.

```
($&) :: (Color -> b) -> a -> b
```

This function can no longer do anything with `a` but it's not an
obstacle since there are three possible definitions for it.

```
f $& _ = f Red
f $& _ = f Green
f $& _ = f Blue
```

Likewise, `b` in this context is covariant and restricting it should
limit the possibilities.  It's easy to see that

```
($&$) :: (a -> Color) -> a -> b
```

has no possible definitions.

Of course, examples are not proofs and the ones I had in this were
demonstrably silly and in any non-trivial program it's unlikely to get
to enumerate the possible implementations for a function of a given
type.

The crux of this is that it's not always that straightforward matter
to see what's the input and output for something, and how they matter
when you think about how to implement a function or reason about them.
Moving to a more specific type in the output type, either via type
variable substitution or by adding a type class constraint, is
actually adding things to the input side.
