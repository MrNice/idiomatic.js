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

Error handling is much much worse. Exceptions in JavaScript contain information about when they were created. With callbacks, that information is essentially garbage because of the polluted call stack. Futhermore, it's difficult to know where it is appropriate to catch an error and handle it. Which named or anonymous function should recover gracefully from this error? Is there any reason to continue with the execution? Will you simply `if (err) throw err;` everywhere, collecting all of the errors at the top and deal with them in a massive switch statement? How will you handle errant code containing a reference to your callback,a nd then calling it randomly 20 minutes later, and it crashes your server?

No.

Why?

Because you will stop writing callbacks *TODAY.*

## What is a promise?
As we've already covered, a promise is just a specialized object. Essentially, every promise is a placeholder for a value. When you do a blocking operation, the operating code returns to you a promise that it will eventually give you a result, or value.

Technically, a promise is always in one of three states. When there is no value, the promise is unresolved. If an error occurs, the promise becomes rejected, and fails. If the value is found without issue, the promise assumes the value.

Promises only ever transition FROM unresolved TO RESOLVED or REJECTED. You cannot restart a promise, but you can start a new promise chain should one fail. You can do this recursively, or with a loop, but you should instead rework your system so that your promises don't fail.

## How do you use promises?
Let's focus on bluebird, require'd as Promise.

To use promises, you must start with functions which return promises or use the Promise constructor.

In the simplest form, a Promise chain is a sequence of functions which return promises, finished with an error handler.

The syntax is: PromiseCreator().then(PromiseCreator).then(PromiseCreator).catch(errorHandler);

```js
GetUserFromIDAsync("12345")
  .then(function (user) {
    // Update user
    user.update_ad = Date.now();
    // Return a new promise to continue the chain
    console.log('The first promise resolved at: ' + Date.now());
    return user.saveAsync();
  }).catch(function (err) {
    // Handle the error...
  });

console.log('This will log out first.', Date.now());
```

The important thing to notice is that you're handling all of the asynchronous operations inside the promise chain. Once you leave the promise chain, everything continues synchronously.

One 'gotcha' is that, unlike Go, you cannot assign a variable to a Promise chain.

```go
c := make(chan int)  // Allocate a channel.
// Start the sort in a goroutine; when it completes, signal on the channel.
go func() {
    list.Sort()
    c <- 1  // Send a signal; value does not matter.
}()
doSomethingForAWhile()
a <-c
```

```js
// This doesn't work
var user = GetUserFromIDAsync("12345");
console.log(user); // Undefined
```

The reason is the same as it is for callbacks. The promise function doesn't block, and doesn't return, and therefore it doesn't have leave a defined value. There isn't a scheduler to handle running functions for you.

### More abstractions
So that covers the most basic use case, and the equivalent to the pyramid of doom.

Bluebird has a large toolchest of helper methods to make using promises simple and easy.

#### Generating Promises
First of all, let's talk about Promise generating functions. There are three ways to get your code to pass promises:

1. use `new Promise` to wrap around complex APIs
```js
return new Promise(function(accept, reject) {
  // Wrapping an odd order of arguments
  doSomething(function(data, err) {
    // Please don't if (err) return reject(err);
    if (err) {
      reject(err);
    } else {
      resolve(data);
    }
  });
});     
```
2. use Promise.promisifyAll() to promisify entire libraries and APIs
```js
// All DB and library stuff will be available by adding 'Async' to the end of the method calls
module.exports.db = Promise.promisifyAll(require('mongoose'));
```
3. use Promise.promisify() to replace certain functions

If all of your internal API's rely on promises to communicate, you'll never have any problems. And if you want your external API consumers to have a choice, you can use Promise.nodeify to turn a promise generating function into a callback accepting function. Although by then your spirit will be so broken you might just simply want to stop being a developer.

```js
// Ripped from bluebird docs
function getDataFor(input, callback) {
    return dataFromDataBase(input).nodeify(callback);
}

getDataFor("me").then(function(dataForMe) {
    console.log(dataForMe);
});

getDataFor("me", function(err, dataForMe) {
    if( err ) {
        console.error( err );
    }
    console.log(dataForMe);
});
```

### Making promises palatable
So .then() only works for returning a single promise (which may resolve to any value, including an array)
What if you have a use case more specific than that?

#### Core Chainables
- .then(fulfilledHandler[, rejectHandler]) handles a single value (which can be an object as well)
- .spread() is like then, but spreads an array of promises over spread's handler's arguments
- .catch() catches errors. Error handlers have a signature(error). Can be specialized to only handle specific errors
- .error() catches Promise.OperationErrors. Useful for recovering elegantly from failures.
- .bind() attaches statefulness to the promise chain. NEVER USE FOR RESOURCES

#### Core Methods
- Promise.join() accepts a specific number of promises
- Promise.try() converts synchronous exceptions into rejections on the promise
- Promise.method() wraps a method and turns exceptions into rejects and returns into resolves.
- Promise.resolve() converts unknown objects or values into promises. jQuery thenables mainly.
- Promise.bind() is sugar for Promise.resolve(undefined).bind(thisArg). A good enough way to start a chain

#### Collections
If you need a specific number of promises to resolve, such as when contacting two databases, you can use .all()

- .all() will fail if any of the promises fail
- .props() works on object properties
- .settle() waits for all promises to settle, that is, resolve or reject
- .any() waits for any promises to finish. useful for realtime racing.
- .some() is like any, but returns an array
- .each()   is like _.each but for async
- .map()    is like _.map but for async functions
- .reduce() is like _.reduce but for async
- .filter() is like _.filter but for async

#### Inspection
This is totally the most important thing for complicated edge cases, but I'm out of time

#### Resources
Promise.using() allows you to start a promise chain with a resource, and release that resource as soon as you are finished.

- .disposer()
Ripped from bluebird api:
```js
// This function doesn't return a promise but a Disposer
// so it's very hard to use it wrong (not passing it to `using`)
function getConnection() {
    return pool.getConnectionAsync().disposer(function(connection, promise) {
        connection.close();
    });
}
```

#### Timers
Sometimes you'll need to work with time.

Mainly, you'll use
```js
.timeout(int ms[, message])
```
To turn the promise into a cancellable promise. If the timeout is reached, the entire promise fails.


