One design choice that you may not always even think of much is the
order of arguments to a function.  There seldom are any hard and fast
rules about what to follow and they can typically be reordered if need
be.  Some rules of thumb may be that you'd expect a non-commutative
binary operation to always use the same order.  In Haskell or
PureScript, `flip` is a pretty standard way of reordering parameters.
In functional programming, partial application and currying set out
some guidelines.  Ordering them so that typical use would see the
least amount of "flip" operations is often a good rule.

The monadic bind operation is one place where following a certain
pattern makes things flow around naturally.  I'll use Haskell's MVars
as an example:

```
newMVar :: a -> IO (MVar a)
takeMVar :: MVar a -> IO a
putMVar :: MVar a -> a -> IO ()
```

These are the basic functions but if you look at the library, you'll
see that all of them follow the pattern where they take `MVar a` as
the first parameter.  The reason for this comes apparent if you do any
operations which chain them using the bind operation.

```
main = do
  -- Typically done in program initialization
  mvar <- newMVar @Int 0

  -- ...
  let askAndUpdate n = do
    putStrLn $ "The number is now " <> show n
    putStrLn "Enter a new number:"
    read <$> getLine
  putMVar mvar =<< askAndUpdate =<< takeMVar mvar
```

In this example, mvar is something that stays the same for the
duration of the program.  If the MVar libary functions used a
different convention, they would require using `flip` and none of this
would read as naturally.  Partial application is the guiding principle
here: Recall that, under the hood, all functions in Haskell take one
parameter and return a value.  Talking about multiple parameters is
just a short hand lie we tell ourselves, in reality all functions in
Haskell return a value.  Some of those values just may be functions as
well.  `putMVar mvar` returns a new function that gets used much more
often than what `flip putMVar 1` would.  That is a valid function but
just not as useful as the other one since you'd typically have just a
few mvars and it's the values they contain that vary more.

`lookup` in Prelude is another one that uses the order it has for a
reason.

```
lookup :: Eq a => a -> [(a, b)] -> Maybe a
```

The first argument is the key to use and the second one has the
haystack.  The list could be something like the query parameters to a
GET call and key could be a certain variable we want to know from it.
The variable name is more likely to be fixed for the program and the
arguments vary according to user input.

This gives the sense that the further a function argument is to the
left, the more fixed it is for the flow of program and the ones to the
right are more variable, input related and ephemeral.  This is not a
hard and fast rule but this is about how the program flow usually goes
and what's the usual way of using the functions.

Partial application has further implications for the natural order of
parameters: Any partial application returns a new function.  This
being functional programming, a natural thing is to give a function to
another function.  I'll use an example from our own software:

```
getFrontpageHtml :: Paper -> String -> Maybe String -> Aff (LetteraResponse String)
```

For this discussion it's enough to note about the return type that
this is a monadic action.  This function is used by our front ends and
we're using the same code to run it for each of the three newspapers
we have, hence the first parameter is only going to have one value for
the duration of the program, which justifies having it as the first
parameter.  The second parameter is for the category we're getting.
We have a few of these (like "Startsidan" or "Nyheter") but they are
mostly fixed as well.  On program init, we use partial application on
this function with these parameters, one being fixed and the other
getting a few values according to a value we download from a server.
This is where we stop at this time: For the actual run time actions we
do to fetch refreshed front page contents, we'll keep a map of `Maybe
String -> Aff (LetteraResponse String)` functions (or values, if you
will).  The final parameter is a timestamp token we add when we do the
actual request.

This is a further dimension to the ordering of parameters: The ones on
the right end may be more ephemeral in the sense of getting more
variable values but they may also be offset in time as well, possibly
getting assigned at more varied times as well than the more "fixed"
ones.

This also expands on a mental model that you may be holding about
functions in Haskell: The last parameter of a function can often be
removed by using function composition, bind or the Kleisli arrow.  It
can sometimes also be done in other positions, which will give you a
function as the result.
