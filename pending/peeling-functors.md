# Peeling functors

This blog post is about PureScript (and React), but here's short cheat
sheet for the differences to Haskell in this post if that's more
familiar to you: In PureScript, `fmap` is simply `map`.  Function
composition is `<<<`, `Unit` stands for `()` and `/\` is an infix
operator for 2-tuples.  And `forall` is always explicit, there's no
implicit `forall`.

If you've had some experience with functional programming with Haskell
or PureScript or some other similar language, one concept you've
surely encountered is functor.  They're generally characterized as
some sort of containers.  `Maybe a` contains one or no value and
likewise an array or list may contain any number of `a`'s.  But that
doesn't really begin to describe how to think about what their
defining function `map` is about.

```
map :: forall f a b. Functor f => (a -> b) -> f a -> f b
```

In this post, I'll show a few uses of `map` in some [real world
code](https://github.com/KSF-Media/affresco/blob/de21d6d043012880bf4bdca96aef0c5d1121927e/packages/components/src/Search.purs)
and tell about how I've thought about them when using them.  The
scenario is that there's a list of customers and in some cases they
might be defined in one system but not in all of them.  If that's the
case then we would like to offer a button that opens a form where the
missing data can be inputted and submitted to create the missing
account.

To represent the possible form, we're using
[useState](https://pursuit.purescript.org/packages/purescript-react-basic-hooks/7.0.0/docs/React.Basic.Hooks#v:useState)
to create a value and a function to alter that value.

```
(accountData :: Maybe (Tuple Cusno (AsyncWrapper.Progress User.NewCusnoUser)))
  /\ (setAccountData :: (Maybe (Tuple Cusno (AsyncWrapper.Progress User.NewCusnoUser))
                         -> Maybe (Tuple Cusno (AsyncWrapper.Progress User.NewCusnoUser)))
			-> Effect Unit) <- useState Nothing
```

To follow what this type represents, think of three layers.  The
topmost is `Maybe a`, which means that the value may be missing.
That's the default state, before we have opened any form for editing.
The second layer is a `Tuple Cusno a`.  It represents how we use an
identifier to place the form in the correct location in the list.  And
finally `AsyncWrapper.Progress a`, which is our [custom
wrapper](https://github.com/KSF-Media/affresco/blob/de21d6d043012880bf4bdca96aef0c5d1121927e/packages/components/src/AsyncWrapper.purs)
for the form data.  Basically it's like a `Maybe` but multiple values
to represent other states.  `User.NewCusnoUser` is used for storing
the form data.  Finally, `setAccountData` is a function of form `(a ->
a) -> Effect Unit`, meaning that it modifies a value of type `a` to
something other of the same type and does it as an effectful action.

When using this to render the search form, we've hidden all this
detail and just use a variable `createAccountForm` to pass the
possible form to our list renderer.

```
createAccountForm :: Maybe (Tuple Cusno JSX)
```

JSX is an opaque data type that represents a section of React DOM
tree.  The rendering function can take it and append it to the list
without having to (or, in this case, getting to) consider what it took
to build it.  Maybeness is retained for this type to represent that
there may not even be a form to render and it uses the Tuple to decide
where to render it.

Let's look at the type of the function that generates that `JSX`.

```
renderEditNewUser
  :: (User.NewCusnoUser -> Effect Unit)
  -> Effect Unit
  -> ((User.NewCusnoUser -> User.NewCusnoUser) -> Effect Unit)
  -> AsyncWrapper.Progress User.NewCusnoUser
  -> JSX
```

The first two variables are for the submit action and cancel action.
Third one is for updating the form state as an action and the fourth
is the form's state, along with the `AsyncWrapper` used to represent
the different stages of submitting the data to the server.  The
reasoning for these types is that all the operations this function
does to manipulate the form data only concern the form's values and we
don't want to set the editing process itself to another state by using
the function.  The only way that `renderEditNewUser` can initiate
those actions is by using the first two variables to cause them.  This
makes the function easier to reason about since it limits the ways how
the state can change.

The reason why the last parameter is of type `AsyncWrapper.Progress
User.NewCusnoUser` and not of `User.NewCusnoUser` is that the function
wants to display it if the form has been submitted and what the result
was.  If it had been restricted to just `User.NewCusnoUser`, all it
could ever do would be to show the form.

Finally, the part where `renderEditNewUser` is used.  Recall that we
started with something of type `Maybe (Tuple Cusno
(AsyncWrapper.Progress User.NewCusnoUser)))` and want to turn it into
a `Maybe (Tuple Cusno JSX)`.

```
createAccountForm = (map <<< map)
                    (renderEditNewUser submitCreateAccount (setAccountData (const Nothing)) $
                    setAccountData <<< map <<< map <<< map) accountData
```

This certainly looks dense, but let's look at it piecewise.  The first
two arguments for `renderEditNewUser` are defined in the middle and
they're straightforward enough.  Third one uses `setAccountData` and
uses `map` thrice with function composition.  Hold on.  I'll make a
brief detour to Haskell to talk about what's going on with this.

```
(. fmap) :: Functor f => ((f a -> f b) -> c) -> ((a -> b) -> c)
```


What that type tells is that it takes a function that manipulates a
value wrapped inside a functor and returns another function that
manipulates something that's lost its wrapper.  I added extra
parentheses to the type for emphasis.  Normally that would have been
written as just `Functor f => ((f a -> f b) -> c) -> (a -> b) -> c`
but every function that takes multiple parameters in Haskell (and
PureScript as well) returns, in reality, another function that takes
one fewer parameter.  So that's a useful perspective to this: it
really needs to be thought of as a function returning another function
to make better sense of it, instead of thinking how `(. fmap)` is a
function that takes two parameters.  That arrow isn't there just to
denote "the next parameter starts here".

To return to what all this functor wrapping entails: Peel away the
maybeness since we'll only do this anyway if there's a value.  Peel
away the tuple since that's for bookkeeping that doesn't concern
rendering the form.  Peel away the async wrapper since we don't want
to do anything but change the form state with this function.

Finally, the definition starts with a `(map <<< map)` and ends with
`accountData`.  They're used to build the fourth parameter for
`renderEditNewUser`, which doesn't even get explicitly mentioned but
is handled by this call.  It uses one map to strip the maybe (as a
side effect short circuiting the whole process and not even calling
renderEditNewUser if it happens to be a `Nothing`) and second one to
tuck away the Tuple.  This time we stop here since we want to display
other async wrapper states besides just the editing phase.

Or here's another, perhaps simpler, perspective to `(map <<< map)`:
Ignore the first parameter at first and just look at the two topmost
layers of its second parameter: `Maybe (Tuple Cusno a)`.  The thing
about mapping is that we already know that anything the first argument
will do, will preserve that outer structure as is.  Knowing that we
can look at `renderEditNewUser`'s return type `JSX` and we'll know
from that that the final type is going to be `Maybe (Tuple Cusno
JSX)`.

That's it.  This certainly isn't the easiest example to show about
functors, but I wanted to try to give a sense of how they get used in
some real world code, and how the thinking process with them goes.
It's all about concerns: Is this something that I want to manipulate
or touch with my function?  If not, then lift the thing with a `map`
and get that out of your view.  "Containers" is a lacking description
for functors to grasp that notion.
