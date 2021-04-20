# Prim.Row and purescript-record for extending functions

In PureScript, the most basic way of expressing a missing value in a
type safe way is to use `Maybe a`.  If something is not needed or not
relevant, just leave it as `Nothing`.

```
type Params =
  { alwaysNeeded :: String
  , optionalParam :: Maybe String
  }

someCall :: Params -> Effect Unit
```

But that can quickly get cumbersome if a record type starts to
accumulate fields and adding new fields would involve changing every
call of `someCall` to include the new parameter, even if it would be
`Nothing` for all other cases.  PureScript's extensible records aren't
really useful for trying to manage this either, as they would allow
extending `someCall` to accept records which had more fields but in a
sense what we would like to do is the opposite of that.

Enter `Prim.Row` and `purescript-records` package.  The type classes
and functions defined in them may seem quite abstract when looking at
them the first time.  Let's focus on [two of the type
classes](https://pursuit.purescript.org/builtins/docs/Prim.Row)
defined in `Prim.Row`:

```
import Prim.Row

class Union (left :: Row k) (right :: Row k) (union :: Row k)
  | left right -> union, right union -> left, union left -> right

class Nub (original :: Row k) (nubbed :: Row k) | original -> nubbed
```

`Union` is used to express that combining `left` and `right` would
result a `union`.  This gives no regard to whether `left` and `right`
have duplicates, so `Nub` can be used to strip those.  We're going to
go from abstract record manipulation functions to more concrete uses,
so what we're doing is that we're using these type classes but we'll
fill in the type parameter slots with concrete types that our program
uses.

Let's add a new field to our `Params` but can we make it so that the
existing calls to someCall can still use the same `Params` type
without having to define new fields?

```
type NewParams =
  ( alwaysNeeded :: String
  , optionalParam :: Maybe String
  , newOptionalParam :: Maybe Int
  )

type DefaultParams =
  ( optionalParam :: Maybe String
  , newOptionalParam :: Maybe Int
  )
```

One necessary change with using `Prim.Row`'s type classes was to
switch from using record types to row types.  [Recall that curly
braces are syntactic
sugar](https://github.com/purescript/documentation/blob/master/language/Records.md)
for `Record ...`.  `NewParams` is the extended set of parameters that
we'd like to use and `DefaultProps` is for values that may have
default values.  The updated `someCall`'s type looks like this.

```
someCall
  :: forall attrs union. Union attrs DefaultProps union
  => Nub union NewParams
  => Record attrs -> Effect Unit
```

Using `DefaultProps` in this position means that `attrs` is free to
leave `optionalParam` and `newOptionalParam` out but `alwaysNeeded` is
still mandatory and adding anything besides those fields would be an
error.  The extra `union` class is just for bookkeeping the fact that
we may have duplicates in between but they'll get erased and what we
get is a simple `NewParams`.  Internally, `someCall` uses
`Record.merge` for actually doing that.  Note that the underlying
JavaScript representation has no notion of duplicate fields but types
have no qualms about being able to express things that have no
inhabitants.  As long as the end result is something concrete then
it's fine.

```
import Record as Record

someCall userOpts = ...
  where
    defaultOpts =
      { optionalParam: Nothing
      , newOptionalParam: Nothing
      }
    opts = Record.merge userOpts defaultOpts
```

There, `someCall` got a new field and any old code using it won't need
to care about it.

This still leaves something to be desired: Why is `newOptionalParam`'s
type `Maybe Int` and not just `Int`?  Can't we express the absence of
value by simply leaving it out?  It can be done though it gets a bit
more involved.  I'll just end this by showing an example of how it
could look like.

```
import Record.Unsafe (unsafeHas)
import Type.Proxy

type NewParams =
  ( alwaysNeeded :: String
  , optionalParam :: Maybe String
  , newOptionalParam :: Int
  )

type DefaultParams =
  ( optionalParam :: Maybe String
  , newOptionalParam :: Int
  )

type SomeParams =
  { alwaysNeeded :: String
  , optionalParam :: Maybe String
  , newOptionalParam :: Maybe Int
  }

someCall userOpts = ...
  where
    defaultOpts =
      { optionalParam: Nothing
      , newOptionalParam: 0
      }
    opts = Record.modify (Proxy :: Proxy "newOptionalParam")
           (\r -> if unsafeHas "newOptionalParam" userOpts then Just r else Nothing) $
	   Record.merge userOpts defaultOpts
```

The internal `SomeParams` type can be expressed with curly braces
again as there's no need to access its row kind anymore.  Mind you,
this comes with a cost: It's not as simple to have a list of params
where some fields are missing and others not.
