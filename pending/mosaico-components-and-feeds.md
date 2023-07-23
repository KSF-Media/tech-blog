Like with most news sites, a part of our content is meant only for
paying subscribers.  As such, Mosaico needs to handle user logins and
state management related to paywalled articles.  In this second part
I'll cover how that's done and expand on its Feeds and cover what
Routes do.

# User component

On a high level, the user may be in one of three states: Pending, not
logged in or logged in.  At application start up, the state is pending
but it'll never return to that state after initialization is done.
Depending on how the data we get from the authentication services,
it'll resolve to either unlogged or logged in state.  Logins and
logouts will just transition between those latter two states.

This state is kept in the User component.  It's the only component
that's added to the JSX prior to the main render function and in the
usual state it doesn't add any content to DOM.  If requested via
User.Props, it can show a login modal dialog.  The main render and
subcomponents will naturally want to know if the user state changes
and it's communicated from the User component via a setter function it
gets in User.Props.

## Vetrina

Vetrina is our paywall implementation.  It has also the capability of
logging an user in, possibly also with creating an account.  User
component doesn't have that functionality, it only does logins.

## Log out

The User component doesn't do log outs either but other parts of the
site have UI elements for doing that.  Invoking those elements'
handlers will make User component set user to not logged in state in
its internal state.

# Article component

Article component handles article rendering as well as loading them.
If the initial page is an article the server will send the article as
JSON in the window variables as well.  For non-premium articles this
is pretty much the whole story but for premium articles there's
further interaction with the User state.

## ArticleStub to Article

The Article component will always get either Article (in initial
render) or ArticleStub.  If the application later returns to the
initial article it'll get ArticleStub and it doesn't use the initial
Article anymore.  If, for any reason, there's a link to an article
within the site and there's no ArticleStub available, it'll have to go
via server render to it.  Most likely with a 404 response.

All rendered article lists will naturally have ArticleStub data
available since it's used for the render.  This also applies for
prerendered pages: Lettera parses the prerendered content on its end
and offers an endpoint for loading the list of ArticleStubs that are
included in the prerendered content.  So, loading a prerendered Feed
will involve two accesses to Lettera: one to get the prerendered data
and another to get the corresponding list of articles.

## Paywall and previews

Mosaico doesn't (for the most part) actually care whether an account
has active subscriptions.  It'll send the authentication tokens, if
available, to Lettera regardless, and Lettera will check if any
condition matches for the user to view the content.  If they are
entitled to read the content, they'll get the full article from
Lettera.  If not, Lettera will respond with an error status but the
response will still contain the article data, including the initial
part of the body content as well.

When this happens, the paywall is appended after the preview content.
The Article component receives the paywall component as a plain JSX
and therefore it doesn't have any way of interacting with it by
itself, but instead it's Props have been primed by the main component.

The Article component will need to be aware of user interaction with
the paywall component and this is done via Article.Props by passing a
counter variable in it.  Its only use is to make the `useEffect` hook
in it register a change and it'll then check if any condition is
fulfilled which would require reloading the article, to attempt the
transition from a preview to the full article.  It also monitors user
log ins and log outs with it but that's not enough by itself since the
user may complete a purchase within the paywall component while
already logged in.

## Advertorials

One more detail that the Article component takes care of:
Advertorials.  A part of our ad business is selling advertorial
article space.  It's a list of articles that are published along our
normal content and we show them among our usual content.  Of course
labelling it clearly as being that kind of content.  One place where
they are used is in our articles, where we show one of them after
every article.  For every article view, we make a random selection
about which one to show.  That state is held within the article
component as well.

# Routing

Routing is the glue that binds this all together.  All the links to
articles, tags, categories and such are regular `a` tags with `href`
attributes in them, but the Handlers Mosaico sets up will catch all
those events and instead will tell the router to change the path to
the corresponding URL without triggering full server load.  They'll
also possible set more state related to navigation, like setting the
ArticleStub to match with the article page URL.  The route listener
will catch the change in address and will set the route in the State
to match with a sum type that represents all the possible pages the
user may access (including a catch all 404 page for the rest).

The same route parser is used on the server side as well and it's used
to populate all the page specific data into Props and State.

Articles are (of course) represented in the `MosaicoPage` data type
and they correspond to article pages.  Another major group of them has
to do with Feed specific pages and it has representations for
categories (one of which is the main page), tag lists and search page
results.

# Feeds and the Cache module

Feeds have been mentioned a number of times already but a full
discussion about them will require covering the Cache module as well.
Feeds are basically lists of articles, possibly coming with a
prerendered representation of where they are used.  They can be
divided into two categories: Common ones that are used for all renders
and ones that have routes specific to them.  Advertorials is also a
feed but it's a bit of a special case.

Common ones are most read articles, latest articles and the breaking
news banner.  Route specific ones are categories (determined by the
category structure data), tags and search results.  Basically any page
that may show a list of articles (either as a rendered list or as
prerendered content) has one.

Feeds are maintained and requested via the Cache module, which is used
both in client and server contexts, but it operates differently in
each.  Cache keeps track of the Feeds data and it's staleness, and if
the content is still fresh enough it'll just reuse the old data.

## Server Cache

On the server side, the Cache module works more proactively.  For the
main categories and the common feeds, it'll download new data from the
server asynchronously even when there's been no requests for it,
whenever they get stale.  That way it'll always have fresh enough
copies of the data available for any requests and for the Feeds it
doesn't actively maintain (and have a fresh enough copy regardless)
it'll require at most one new request to Lettera.

It'd be a topic for another article but it also has the capability for
receiving a cache invalidation notification from Lettera.  That way
when the prerendered front page is updated Mosaico will know to
discard the old data and also notify our Varnish cache about it.

In server context, getting Feeds from Cache is rather straightforward.
It's just done as a single call that returns an Aff that contains
them.  It returns an array with all the common Feeds with a single
page specific feed, if applicable.

## Client Cache

On the client side, the Cache operates much more on a per request
basis.  There are no actively updated Feeds but instead when the user
changes routes it will fire up.

The way data flows from Cache to the application is different from the
server context.  It doesn't get an Aff containing the Feeds anymore,
but instead the application will submit a setter function to the Cache
at program start up time, once.  Loading a page involves issuing a
request to the cache which, by itself, has a return type `Aff Unit`.
The application will show a loading spinner while Cache is loading it
via having no entry in the `Map` it uses in its internal state and
once the load is done, Cache will call the setter function to set it.

Issuing a request to a page specific Feed will also trigger updates to
the common Feeds, if needed.

## Feeds' data types

Feeds have two representations in application code.  One is
serialization oriented one: An array of Tuples with Feed type and Feed
content as its values.  This is also closer to how they are
represented internally in Cache module.  In this form they are
serialized and send in window variables during the server render.

Another one is more application oriented, where there's a Map that
contains the page specific Feeds and the common feeds are split into
record fields for easier access.  The type also makes it clear that
they are always available and aren't duplicated for any reason.
There's a conversion function from the array Feeds representation to
the record one.

## Advertorials

Advertorials are also handled via the Feeds mechanism, though their
handling is a bit different.  On the server side they are maintained
just like the common Feeds.  On the client side, they are loaded from
the serialized Feeds data at start up time, but they are held as fixed
for the duration of the application.
