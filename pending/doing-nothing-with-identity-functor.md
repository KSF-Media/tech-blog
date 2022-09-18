There's more than one way of doing nothing in Haskell.  The identity
function `id :: a -> a` is the simplest of them.  For example, you
could use it with `maybe` to build a function that adds a number if it
is given one.

```
addNumberIfJust :: Maybe Int -> Int -> Int
addNumberIfJust = maybe id (+)
```

The identity function is, on its own, naturally uninteresting but it
can have a use when doing pattern matching to fill in the blanks.

The `return` function can be used pretty similarly to `id`.  `return`
has structure of its own due to its monadic nature and has more uses
due to it, but it can still fit a similar role to `id`.  Consider this
example:

```
appendFileIfJust :: Maybe FilePath -> String -> IO String
appendFileIfJust = maybe return (\path x -> (x <>) <$> readFile path)
```

This does a bit more than sums a number but the basic structure is the
same.  `maybe return` is similar to `maybe id` in that they both
provide a filler to use in the base case.

There's a a third kind of identity in Haskell that has even more
structure to it and I'll present a real world example of its use, from
our article backend server.  `Data.Functor.Identity` is, as you'd
expect from the name, a functor with, as of this writing, some three
dozen other instances.  The trivial and uninteresting use for it is to
construct one with `Identity` and immediately undo it with
`runIdentity`.

```
newtype Identity a = Identity { runIdentity :: a }
```

Our article backend, Lettera, has a number of end points that return
lists of articles.  All of them offer JSON as a return type, with a
list of articles using our internal JSON representation but some of
them also offer RSS feeds.  For the feeds, they need a bit more
additional data besides of what's available in the articles itself,
since an RSS feed has some data about the feed itself and it's not
just a list of contents.  To help build that data, we use an
intermediate data type `Feed a`.

```
data Feed a = Feed
  { articles :: a
  , paper    :: Maybe PaperCode
  , feedTime :: Maybe UTCTime
  , category :: Maybe CategoryId
  }
  deriving (Functor, Foldable)

instance ToJSON a => ToJSON (Feed a) where
  toJSON = toJSON . articles

instance Applicative Feed where
  pure articles = Feed
    { articles = articles
    , paper = Nothing
    , feedTime = Nothing
    , category = Nothing
    }
  a <*> b = a { articles = articles a $ articles b }
```

When outputting the JSON format, all of the wrapping data is
discarded.  For those endpoints that don't offer RSS feeds, the
backend functions returing article lists will skip building a `Feed a`
and instead just directly return the list of articles.  So far so
good.

Another feature of Lettera is that currently it gets its articles from
two different system.  We're in the middle of migrating from one
publishing system to another one and are still sourcing the articles
from both.  Most of our endpoints get the articles from both systems
and use the following function to combine the results.

```
newtype OCAction     a = OCAction (Lettera a)
newtype AAction a = AAction (Lettera a)

combinedSourceList
  :: (Eq t, Ord t, Functor f, Applicative f, Foldable f)
  => AAction (f [t])
  -> OCAction (f [t])
  -> Lettera (f [t])
combinedSourceList (AAction aFetch) (OCAction ocFetch) =
```

I'll omit the implementation.  `OCAction` and `AAction` are
simple wrappers.  `Lettera` is the type alias for the reader monad we
use to keep the running environment.  Lettera's article type has `Eq`
and `Ord` instances which are used to deduplicate and sort the
combined results.  The function's not using concrete article types
since we have different article data types for the different API
generations we have.

Another aim with this function was to get to use it both for lists
that are wrapped inside a `Feed` and ones that are plain lists.
Substitute `Feed` for `f` and it's good to go for the RSS feed case.
But there's a stumbling block for using it for the plain list
endpoints.  There's nothing to put into that `f` slot in that case
since what you start with is just a `[t]`.

This is where the identity functor can be used.  The simple yet in
this case non-working way of using this function would be:

```
combinedSourceList (AAction f) (OCAction $ map toV4 <$> g)
```

where `f` and `g` are some actions which return lists of articles and
`toV4` converts articles to the current generation's data type.  To
get this to work, these can be wrapped inside identity:

```
combinedSourceList
  (AAction $ Identity <$> f)
  (OCAction $ Identity . map toV4 <$> g)
```

The final result needed from this all is a plain list, so the call
needs one more step:

```
runIdentity <$> combinedSourceList
  (AAction $ Identity <$> f)
  (OCAction $ Identity . map toV4 <$> g)
```

This works since the identity functor has also instances for
Applicative and Foldable.

A bit more involved than the toy examples with `maybe` that this
started with, but the general pattern was the same: There was a slot
that needed to be filled with something for a base case, and identity
was the piece that could be used.
