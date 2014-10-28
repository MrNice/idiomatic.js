# What are promises?
A promise is an object, which represents a value, which is either unresolved, or (resolved or rejected). But this doesn't help you.
## What problems do promises solve?

If you use promises, you never have to use or write callbacks. Full stop.

## What about async?
Async is a great library, but it's significantly slower than bluebird (faster than Q), and it's odd declarative object syntax is often more difficult to read and reason about than bluebird's. If you have ever had to do an async.series or .waterfall inside of an async.parallel, you'll understand what I mean.

## What's so bad about callbacks?

First, let's examine what a callback *is*:

A callback is a function which is executed at some later time. It is a way of delaying the execution of code, and of turning an event loop into a simplified scheduler. Since functions are first class objects in JavaScript, you can pass callbacks around and Errors until something is capable of handling it.

A node-style callback is a callback with the signature (Error or Null[, arg1, arg2...]) passed as the final argument to an asynchronous function.

Callbacks are often anonymous. They are created during runtime as needed, and passed around. Once the function is invoked, no more references to it should remain, and it gets garbage collected.

You can compose entire programs out of callbacks, where anonymous functions are created within other anonymous functions and passed until a final callback `return` s and the the callback tree 'unwinds' until more logic is to be performed. In this way, any time a 'blocking' operation occurs, the execution can be paused and restarted once the blocking operation finishes. This is how JavaScript utilizes its event loop to perform operations in a highly concurrent manner.

The main strength of callbacks is they work everywhere, out-of-the-box.

The main problem with callbacks is that they're terrible in every single way.

## Give me examples already!

For one thing, they're ugly to compose and difficult to follow. When building a very simple processing chain, where you must read from a database AND then use that data to pull from an API, then send that data to a client, finally store the changed data in the database, you end up with a callback pyramid of doom:
```js
INSERT_DOOM_PYRAMID_HERE
```

What if instead you need to pull data from two seperate databases, the transform them together, then send it out? Obviously, you can do
```js
INSERT_DOOM_PYRAMID_HERE
```
But the problem is that there's really no reason to not make both requests at the same time. The second callback query will be the same no matter what - the first query has no impact on it. Therefore, you should be able to do them concurrently.

How do you do concurrent callbacks? Well... you don't. You'd need to keep track of the status of each callback and wait for them to resolve before contiuing. You can do this with events, where you emit a 'done' event along with the data as soon as you are finished, then wait for all of the events to fire (using an array or an object or some custom data structure to keep track of them all). But this evented code would be difficult to reason about and be frustration boilerplate to write (I've done it in tesselate).
```js
var tesselate = function(config, callback) {
  var loadedModules = {};
  var readyModules = 0;
  var modules = null;
  var alias = null;

  // TODO: add error checking for callback and config
  if (config.development === undefined) config.development = true;

  if (Array.isArray(config)) config.modules = objectify(config);

  devLog('Tesselating...');

  if (Object.keys(config.modules).length) {
    for (var tesselPort in config.modules) {
      if (tesselPort = 'E') break;
      module = config.modules[tesselPort][0];
      alias = config.modules[tesselPort][1];

      devLog('Loading "' + module + '" as "' + alias + '"...');

      ++readyModules;

      loadedModules[alias] = require(module).use(tessel.port[tesselPort]);

      loadedModules[alias].on('ready', function() {
        devLog(capitalize(config.modules[tesselPort][1]) + ' is ready.');
        // If no more modules to load, call the callback
        if (!--readyModules) {
          callback.length === 1 ? callback(loadedModules) : callback(tessel, loadedModules);
        }
      });
    }
  } else {
    devLog('No modules to load, continuing...');
    callback.length === 1 ? callback({}) : callback(tessel, {});
  }
};
```

But that's not nearly ideal, as it relies on lexical scoped variables to keep track of the callbacks resolving.

These are some fairly simple cases for callbacks. But it's easy to see how confusing they can become if you compose entire programs out of them.

There are other problems with callbacks as well. Debugging is fairly difficult, because when an error is thrown, the callstack is polluted with all of the calling functions, in the order in which they were called, and it's difficult to figure out what state is actually available to the current function, and why it's not what you're expecting.

Error handling is much much worse. Exceptions in JavaScript contain information about when they were created. With callbacks, that information is essentially garbage because of the polluted call stack. Futhermore, it's difficult to know where it is appropriate to catch an error and handle it. Which named or anonymous function should recover gracefully from this error? Is there any reason to continue with the execution? Will you simply `if (err) throw err;` everywhere?

No.

Why?

Because you will stop writing callbacks *TODAY.*

## What is a promise?
As we've already covered, a promise is just a specialized object. Essentially, every promise is a placeholder for a value. When you do a blocking operation, the operating code returns to you a promise that it will eventually give you a result, or value.

Technically, a promise is always in one of three states. When there is no value, the promise is unresolved. If an error occurs, the promise becomes rejected, and fails. If the value is found without issue, 
