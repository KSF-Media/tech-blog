As the old programmer joke goes, there are two difficult things in
computing.  One of them is cache invalidation.  There are a lot of
things going on with a cache.  They're needed to make things go fast
or often enough, to be able to even run a service at all.  But they
add complexity to programs since you'll have a whole new kind of
potential bugs with having stale caches.  You want to have them update
but not too often, either according to a validity period or by
request.

Of course, there's no single caching solution or architecture that'd
fit every scenario.  In this article, I'll discuss what fit our needs
with serving newspapers' web sites.  We have a limited number of
categories that we want to keep cached, like "Startsidan" or
"Nyheter".  The main ones we want to keep available at all times but
the subcategories can be served by need, but if we fetch them once
then it won't harm to reuse the data while it's still considered
valid.  We have two kinds of cached content, one is prerendered HTML
and the other is a list of articles.  Our web site can render either
but the prerendered one is the preferred form to use for the front
page since it gives our journalists a free hand at laying the content
how they want.

In terms of software, these requirements set the shape for our cache.
We won't cache just one kind of data, so it'd be useful to have a
generic caching data type.  The main categories are pretty much fixed
for the duration of our back end software, so we can build a fixed
maps structures to reflect that.  We want to keep using the cache
simple, so we'll hide the whole logic of fetching data and updating it
inside the cache module and just use the access functions from the
outside to get the data.  When using multiple cached resources to
generate one response, we want to use set the combined data's validity
period to follow the data that's going to expire the soonest.

With this all said, let's look at the code.

```
data Stamped a = Stamped
  { validUntil :: DateTime
  , content :: a
  }

instance functorStamped :: Functor Stamped where
  map f (Stamped s@{ content }) = Stamped $ s { content = f content }

instance applyStamped :: Apply Stamped where
  apply
    (Stamped { validUntil: t1, content: f})
    (Stamped { validUntil: t2, content: x}) =
    Stamped { validUntil: min t1 t2, content: f x }
```

`Stamped` is the basic type for cached resources.  It's container-like
so you won't be surprised to see it to have a `Functor` instance.  The
`Apply` instance is used to combine cached resources, to handle the
bookkeeping of using the minimum of the validity times.  We'll just
sidestep the question about what the default validity should be by not
providing a `pure` for `Stamped`.

```
data Controls a b = Controls
  { reset :: Aff Unit
  , set :: (LetteraResponse a) -> Effect Unit
  , content :: Aff (Stamped b)
  }

instance profunctorControls :: Profunctor Controls where
  dimap f g (Controls c@{ content}) = Controls $ c
    { content = (map <<< map) g content
    , set = \x -> c.set $ f <$> x
    }
```

`Controls` is used to represent the actions available to manipulate
the cached object.  `reset` should be self-explanatory about what it
does.  `content` uses `Aff` to represent that accessing the data may
involve waiting for the data to arrive.  `set` is used in case the
cache receives a new resource as off band data.  In this case, we use
Google's pubsub messaging queue to get an updated front page as soon
as possible from our publishing system to the cache.
`LetteraResponse` is a container type that has the message body and
some data from the response headers, mainly the cache max age data.

`Controls` has two type variables since the type of the input it gets
may differ from its output type.  In its basic form these types would
match but when a `Controls` is mapped those can be changed
indepentently.  This needs to implement a `Profunctor` instance since
reinterpreting `set` is going to alter the input it gets and not the
output, which requires contravariance.

The following function, whose implementation details I'll skip, is the
one that sets up all of this.  It uses a bunch of `AVar`s to set up
locks and synchronization for keeping the data we're following up to
date.  If it encounters an error it'll clear the data it's holding and
will load a fresh copy the next time someone makes the request via
`Control`'s `content` function.

```
startUpdates
  :: forall a. Aff (LetteraResponse a)
  -> Effect (Controls a (Maybe a))
```

The argument it gets is an action which gets the resource.  It uses a
type variable to abstract over what kind of data we're holding, so it
can work with both HTML and lists.  The `LetteraResponse` container it
uses will let `startUpdates` know about errors and the `Controls`
value it returns will have `Maybe` in its output slot to denote that
on error, we'll be having `Nothing` as the value.

```
type UpdateWatch a = Controls a a

type Cache =
  { prerendered :: HashMap CategoryLabel (Controls String (Maybe String))
  , mainCategoryFeed :: HashMap CategoryLabel (UpdateWatch (Array ArticleStub))
  , mostRead :: Aff (Stamped (Array ArticleStub))
  , latest :: Aff (Stamped (Array ArticleStub))
  , subCategoryFeed :: AVar (HashMap CategoryLabel (Stamped (Array ArticleStub)))
  , byTag :: AVar (HashMap Tag (Stamped (Array ArticleStub)))
  , subPrerendered :: AVar (HashMap CategoryLabel (Stamped String))
  }
```

`Cache` is the container for all our various cached resources and the
way it holds each of them reflects their properties.  `prerendered`
and `mainCategoryFeed` both hold data that concerns a fixed set of
keys and which we expect to have cached at all times.  For each of the
types that contain lists, the data fetch will just default to using an
empty list if any errors occurred.  `mainCategoryFeed` is the
exception since we'll use the `Nothing` state outside of the cache
module to use the list layout instead as the default.  `mostRead` and
`latest` are both simple lists shared by most pages on the web site.

`subCategoryFeed`, `byTag` and `subPrerendered` are all items that are
only updated by need, and there's no fixed set of them that we'd
expect at any time.  Hence they are the only ones that have the
containing map data structures inside mutable variables, nor do they
have any `Controls` associated with them.

Finally, the initialization function for the cache handle:

```
initCache :: Paper -> Array Category -> Effect Cache
```

Again, I'll skip most of the implementation details.  The
initialization of `mainCategoryFeed` is perhaps the most interesting
one of them:

```
initCache paper categoryStructure = do
  mainCategoryFeed <-
    (map <<< map <<< rmap) (join <<< fromFoldable)
    HashMap.fromArray
    <$> traverse (withCat $ startUpdates <<< Lettera.getFrontpage paper <<< Just) categoryStructure
```

`join <<< fromFoldable` is used to squash any `Nothing` results in
`startUpdates` to just represent it as an empty list instead.  What
follows it is the call to `startUpdates` and some scaffolding to use
it for each value in `categoryStructure` and to build the kind of data
type we're going to hold in the cache handle.  `map <<< map <<< rmap`
has one map for reaching inside an `Effect` and another for doing the
same to the `HashMap` but the final `rmap` is different since
`Controls` is a profunctor and it's making a map on the right hand
(output) type it has only.

Another design option would have been to pass `startUpdates` another
function as a parameter to decide what to do with errors.  But since
what it returned was a functor, we could use a map to instead decide
how to handle them.  The way this is set up, that function is used
every time the cache gets updated and not just once, but that
distinction was transparent for the humble map function.

The full implementation of the cache is available in our public
repository (insert link once the pubsub branch has been merged) and
examples of how the cache is accessed can be found in our main server
side module (insert link).
