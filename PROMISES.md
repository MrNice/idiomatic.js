# What are promises?
A promise is an object, which represents a value, which is either unresolved, or (resolved or rejected). But this doesn't help you.
## What problems do promises solve?
If you use promises, you never have to use callbacks. Full stop.
A callback is a function which is executed at some later time. It is a way of delaying the execution of code, and of turning an event loop into a simplified scheduler. Since functions are first class objects in JavaScript, you can pass callbacks around and Errors until something is capable of handling it.

Callbacks are often anonymous. They are created during runtime as needed, and passed around. Once the function is invoked, no more references to it should remain, and it gets garbage collected.

You can compose entire programs out of callbacks, where anonymous functions are created within other anonymous functions and passed until a final callback `return` s and 

The main problem with callbacks is that they're terrible in every single way.
