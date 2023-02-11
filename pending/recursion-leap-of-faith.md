A mathematical proof is often structured like a detective story.  You
have a list of suspects and facts you know about them.  You apply them
using logical rules and end up with butler being the culprit.  Except
when you don't.

Likewise, sometimes when writing a function it pays to stop and assess
what you know and what are the givens.  You know that there's a value
you want but it's not always the best thing to rush towards it or even
start evaluating what you are writing.  Let the computer worry about
that part for you.

Most introductions to recursion start by demonstrating how the
evaluation of recursion goes.  Pick some function that takes a natural
number and show how it gets values for some small numbers.  It may be
instructive to see it once (like [in the Haskell
wikibook](https://en.wikibooks.org/wiki/Haskell/Recursion)) but I'd
ask you, dear reader, to just forget about it once you've read about
it there.  It's not incorrect as such, but it really primes you to
focus on the wrong thing.

The question you should be asking yourself when you encounter
recursion is "what do I know?"  You have an idea about what you want
as an result, and you (should at least by now if you are reading about
recursion) know about things like arithmetic operations and pattern
matching.  Let's continue with the example with factorials.  It's a
function, that in mathematics is denoted by adding an exclamation mark
after a number or variable.  It amounts to multiplying every value
from 1 to n together.  The first few values of it are 1! = 1, 2! = 2,
3! = 6, 4! = 24 and so on.

A classic example of something you could do with recursion.  The type
of it's implementation in Haskell would be

```
factorial :: Int -> Int
```

Now, stop everything and start thinking of what everything you know.
Here's the key to understanding recursion: That line alone is
something you can add to your list of facts you have, already.  You
have just defined a function for generating factorials and you can use
it as is in what follows, without needing to complete it.

```
factorial 1 = 1
```

The classic base case, without it your program evaluation would
happily go on forever (or until your stack or memory runs out).

```
factorial n = n * factorial (n - 1)
```

Here's where you take the leap of faith: In the list of facts you know
about the world, you have already added factorials itself.  Stop
yourself from thinking about what the value of `factorial (n - 1)` if
you are doing it now.  Instead, take it as a fact you can use it
already.

Self-reference is a deep concept and there's just no way around it.
If it helps, you're in [good
company](https://www.scientificamerican.com/article/what-is-russells-paradox/)
if it's causing surprises.  In a computing context, it will ultimately
amount to some form of chasing the tail to some eventual end but
that's not nearly always the best way to think about it.  Instead,
you'll need to take it "as it is" and build a mental model of using it
like that.

For another perspective of the same principle in action, I recommend
reading about [mathematical
induction](https://www.mathsisfun.com/algebra/mathematical-induction.html).
It's about constructing proofs for all values instead of getting a
value from any value, so it's "building up" instead of "building down"
like recursion does, but the way you have to use self-referentiality
to tie the loop and make it work is based very much on the same idea.

If I were writing a full Haskell tutorial on my own, I'd try to move
covering recursion from basics to as far as I can to the advanced
section.  It may be tempting to introduce it early since it gives you
that nice and concrete chase the evaluation view of things, but I
suspect it does a disservice since that's not the mental model I'd
promote.  Moreover, in any practical Haskell code you'd more often be
writing folds instead of recursions.  I think it'd be easier to start
with just describing how `foldr1` can work over a collection by
operating pairwise on each element, and move from there to show how
`foldr` does the same but it can handle the empty case as well.

```
factorial = foldr (*) 1 . enumFromTo 1
```

I'd use something like that as the starting point.  Function
composition and higher order functions can be introduced well in
isolation and using a commutative operation will defer the whole topic
of fold direction until later.

I think you're all waiting for the obvious programmer joke by now: If
you want to read what I think of recursion, scroll to the start of the
article.
