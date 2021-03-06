# The Fraustrating RegExp
The most tedious part of programming, is probably the regular expression, because it is just too complicated. One of the greatest features `Backbone` provides us, is the style of declarative routes & event handling. It abstracts away the complexity of regular expressions.

Firstly, as a reminder, when extending a router, routes can be defined in declarative syntax:

```js
var Router = new Backbone.Router.extend({

    // Each route can also be declared as string/function pair, e.g.
    // "search/:query/p:page": function(query, page) {  }
    routes: {
        "search/:query/p:page": "search",
        "file/*path": "file",
        "docs(/:chapter)": "docs"
    },

    search: function(query, page) {},
    file: function(path) {},
    docs: function(chapter) {}
});
```
For example, here are some urls can be matched:

- "search/books/p50"  - search("book", "50");
- "file/folder/router.js" - file("folder/router.js");
- "docs" - file("");

This illustrates that routes can have 3 types of parameters:

- `:param` matches a single **named parameter**;
- `*param` matches a parameter of any length, and is called a **splat parameter**;
- `(:param)` parameter wrapped in parentheses is **optional**.

An intuitive workflow of `Backbone.Router` instance is:

1. You specify declarative `routes` in the `option` object, either when extending the class as the above example did, or when `new`-ing an instance.
2. The constructor will convert each route to a RegExp object internally and register corresponding callback to that route. For example:

```js
  // you specify a route `book/:id`, with `book` callback
  routes: {
    "book/:id": "book"
  }
  // internally, it become something like this:
  var self = this;    // `this` is a Router instance here.
  Backbone.history.handlers.unshift({
    route: /book\/([^/?])/,   // converted to RegExp
    callback: function(fragment) {
      // Extract parameters from current fragment
      var args = router._extractParameters(route, fragment);
      self["book"].apply(self, args);
      // then trigger events
    }
  });
```
  I made up the code above just for comprehension. It is actually more complicated than that.

3. `Backbone.history` object watches the current path, based on different history state management schemes, listens to either `popstate` or `hashchange` event. Whenever the current fragment changes, `history` searches its handlers, if `handler.route` can match the current fragment, `handler.callback` will be called.

It doesn't matter if you cannot completely understand it at present. Feel free to read on.

You might need a call graph before you step into the source code:

![](assets/router.png)

## Constructor

Let's first take a look at the first few lines of the router module:

```js
var Router = Backbone.Router = function(options) {
  options || (options = {});
  if (options.routes) this.routes = options.routes;
  this._bindRoutes();   // Bind routes we specified.
  this.initialize.apply(this, arguments);
};

var optionalParam = /\((.*?)\)/g;
var namedParam    = /(\(\?)?:\w+/g;
var splatParam    = /\*\w+/g;
var escapeRegExp  = /[\-{}\[\]+?.,\\\^$|#\s]/g;
```

- **optionalParam**: that's straight forward, anything wrapped inside parentheses is optional.
- **namedParam**: something like "(?:namedParam", or ":namedParam".
- **splatParam**: something like "*param". Match any number of params
- **escapeRegExp**:
    This one is not so straight forward. It took me time to find out what it is. As its name suggest, this regx matches some meta characters in regular expression:
    `-, {, }, [, ], +, ., comma, \, ^, $, #, whitespace` will be matched by this regx. You might be wondering there is not a back slash among: `\]+?.,`. It's find, MDN says they lose their special meanings inside a character set. some characters wrapped in a pair of bracket form a character set. An answear on stackoverflow has mentioned it, [go check it.](https://stackoverflow.com/questions/399078/what-special-characters-must-be-escaped-in-regular-expressions)

## _bindRoutes & _routeToRegExp

`_bindRoutes` deals with all `routes`:

```js
_bindRoutes: function() {
  if (!this.routes) return;
  this.routes = _.result(this, 'routes');
  var route, routes = _.keys(this.routes);

  while ((route = routes.pop()) != null) {
    this.route(route, this.routes[route]);
  }
}
```

`route` method first convert each route to regx, by calling `_routeToRegExp`. Here's the `_routeToRegExp`:

```js
// Copy them here for you to reference them.
var optionalParam = /\((.*?)\)/g;
var namedParam    = /(\(\?)?:\w+/g;
var splatParam    = /\*\w+/g;
var escapeRegExp  = /[\-{}\[\]+?.,\\\^$|#\s]/g;

...
_routeToRegExp: function(route) {
                // escape meta character of RegExp.
  route = route.replace(escapeRegExp, '\\$&')
               .replace(optionalParam, '(?:$1)?')
               .replace(namedParam, function(match, optional) {
                 return optional ? match : '([^/?]+)';
               })
               .replace(splatParam, '([^?]*?)');
  return new RegExp('^' + route + '(?:\\?([\\s\\S]*))?$');
},
```
This is pretty fruastrating because I cannot completely understand it.
The first 2 `replace` calls can be understood. Here's what I found on MDN:

**Specifying a string as a parameter**

| Pattern | Inserts |
|---------|---------|
| $$ | Inserts a "$". |
| $& | Inserts the matched substring. |
| $` | Inserts the portion of the string that precedes the matched substring. |
| $n | Where n is a positive integer less than 100, inserts the nth parenthesized submatch string, provided the first argument was a RegExp object. Note that this is 1-indexed. |

To make life easier, assume that `history` knows we're using `pushState`, and we have a route

    `/employee[:name]/(id)`.

It might not be a good RESTful design, but whatever, it just serves as an example. After replaced by the first 2 strings, we got:

    /employee\[:name\]/(?:id)?

The 1st `replace` simply placed a backslash on meta character; the 2nd `replace` changed the optional param `(id)` into `(?:id)?`. This means if you got a url like "/employee[name]"

This is where it gets complex. `(?:id)` is a **non-capturing** group. If you're unfamiliar with **non-capturing** group, you need to check out [this post](https://stackoverflow.com/questions/3512471/what-is-a-non-capturing-group-what-does-do) before reading on.

After being processed by the 3rd `replace`, this is even more complex, because we have:

    employee\[([^/?]+)\]/(?:id)?

Why is this? because the part matched by `namedParam` is "[:name]", which has no group(If you're unfamiliar with the concept of group, go back to that link above to check it), thus the `optional` param passed to that replacement function is empty, & finally return `([^/?]+)`. If you have problem understanding `replace` method, you might need [this](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/replace#Specifying_a_function_as_a_parameter).

This is pretty delicate, but hard to understand or express, because there is no concept of "named parameter" mentioned on the official docs. A "named parameter" can contain any character except "/" and "?" of any number; An "optional parameter" can contain anything.

The last `replace` is pretty straight forward. A "splat parameter" can contain any character except "?" of any number.

The last step is to append something which I cannot really understand & return a RegExp. So far each `route` is a RegExp that can be used to map urls internally.

#### route & _extractParameters

Next thing to take a look is `route` method. After converting `route` to a RegExp, it then delegates to `history.route`.

`history.route` is pretty simple, `history` is pretty complicated though. `history.route` registers our callback to `history.handlers`. It does all those state management for us, but currently we're not into it because it just complicated.


```js
route: function(route, name, callback) {
  // Convert `route` to RegExp
  if (!_.isRegExp(route)) route = this._routeToRegExp(route);
  
  // Might have 2 styles.
  if (_.isFunction(name)) {
    callback = name;
    name = '';
  }
  if (!callback) callback = this[name];

  // Delegate to `history.route`
  var router = this;
  // For later reference, I added a name to this callback.
  Backbone.history.route(route, function routeHandler (fragment) {
    // Get the parameters from current fragment
    var args = router._extractParameters(route, fragment);
    // Execute our callback.
    if (router.execute(callback, args, name) !== false) {
      // Trigger events for the observer to react.
      router.trigger.apply(router, ['route:' + name].concat(args));
      router.trigger('route', name, args);
      Backbone.history.trigger('route', router, name, args);
    }
  });
  return this;
}

_extractParameters: function(route, fragment) {
  // extract parameters
  var params = route.exec(fragment).slice(1);
  return _.map(params, function(param, i) {
    
    // Don't decode the search params.
    if (i === params.length - 1) return param || null;
    return param ? decodeURIComponent(param) : null;
  });
}
```
To draw a conclusion, let me put the picture altogether.

1. Remember each route we specified will be converted to a RegExp object. This RegExp is used for matching **parameters**.

2. When the `popstate` event happens(assuming that `history` figured out that we should use `pushState`), `history` object figures out which route is currently active, and triggers that `routeHandler` function above.

3. `_extractParameters` use `route` to match a set of **parameters**, pass to the callback that we've registered for this route.
