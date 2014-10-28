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

What if instead you need to pull data from two seperate databases, the transform them together, then send it out? Obviously, you can 
