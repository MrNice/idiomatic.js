Functional Toolchains

Why use a functional toolchain?

## Background

When we write node programs, we're mostly creating data pipelines. Node.js is insanely good at collecting data from sources (API producers, databases), transforming it, and then streaming (piping) to a drain (databases, API consumers). For our purposes, we usually collectAll, transformAll, and then sendAll, because the datasets are small enough to be processed by the server without stalling out other requests or hogging all of the available memory.

In data pipelines, we have sets of data which need to be collected, processed, and sent. Functional toolchains make the processing portion simple, readable, and elegant by allowing you to define a transformation as the composition of many smaller functions.

```js
var arr = [1,2,3,4,5,6,7,8,9];
// addOne to arr, losing the original array forever
for ( var i = 0, l = arr.length; i < l; i++ ) {
  arr[i] = ++arr[i];
}

// Same thing with each
_.each(arr, function(number, index, collection) {
  // Mutate same index in the collection
  collection[index] = ++number;
});
// You don't gain anything yet, except a small amount of readability
// The functional each version is slightly more understandable by non-programmers

// Using Map
arr2 = _.map(arr, function(num) {
  return ++num;
});
// arr is left untouched, and a new array is created.
// We haven't lost any information

// What happens when we create a named function?
var addOne = function(num) {
  return ++num;
};

// BOOM! Perfectly readable, as long as you know what a mapping is / does
arr2 = _.map(arr, addOne);

// We can even chain these functions
var addFiveToEach = function(numbers) {
  return _(numbers)
           .chain() // Comonadic wrapping because underscore is imperfect
           .map(addOne)
           .map(addOne)
           .map(addOne)
           .map(addOne)
           .map(addOne)
           .value(); // Retrieve the set of values from the comonad.
};
```

## The grandfather function(s)
There are two functions which can both be considered the foundation for functional pipelines (at least within JavaScript.) They are: each and Reduce.

`**Each**` takes a collection (a generic name for an ennumerable / iterable) and passes each item to a function. On its own, it does nothing else. Alone, it's very useful for mutating object properties or pieces of the array. You have already seen an example. Each is the grandfather function in Underscore, lodash, and Lazy.js, among others.

Here's map, filter, and reduce expressed through each (assuming each works on any collection):
```js
_.map = function(collection, mapper) {
  var mapped = [];
  _.each(collection, function(item, index, collection) {
    mapped.push(mapper(item, index, collection));
  });
  return mapped;
};

_.reduce = function(collection, reducer, memo) {
  _.each(collection, function(item, index, internalCollection) {
    memo = reducer(memo, item, index, internalCollection);
  });
  return memo;
};

_.filter = function(collection, predicate) {
  var arr = [];
  _.each(collection, function(item) {
    if (predicate(item)) {
      arr.push(item);
    }
  });
};

// If we use reduce, we don't have to lexically scope or abuse context
_.filter = function(collection, predicate) {
  return _.reduce(collection, function(result, item) {
    if (predicate(item)) {
      result.push(item);
    }
  });
};
```


`**Reduce**` (also known as foldLeft) take a collection and transforms it into a single value, by repeatedly applying a transformation to a collection. Reduce is different from each because the reducing function also passes a stateful 'memo' through the reduction chain, which allows reduce to be a *pure* grandfather function, as even each must rely on lexical scoping to simulate reduce. Let us see an example of using reduce as a map:

```js
var arr = [1, 2, 3, 4];

_.reduce(arr, function(result, x) {
    result.push(x + 1);
    return result;
}, []); // Starting memo
// -> [ 2, 3, 4, 5 ]
```
The reducer takes a function and a starting memo, in this case an array. It pushes, 1 by 1, each ++number into the memo array, before returning the memo as its single value.

Reduce is called reduce because it transforms a collection into a single value, but that single value can also be a collection.

Here's a more typical use of reduce:
```js
var sum = function(numbers) {
  return _.reduce(numbers, function(sum, num) {
    return sum + num;
  }, 0); // We don't need to supply this memo, it's implicit
};
sum([1, 2, 3, 4, 5]); // 15
```

## Caveats
No matter what the library promises, it will never be as fast as a custom, hard coded for-loop in javascript. v8 optimizes each function independently, so a composition of 6 functions won't be simplified.

For example:
function addOne(number) { return ++number };
var a = addOne(addOne(addOne(addOne(1)))); // a == 5
// addFive is a contrived function created by functional composition
function addFive(number) { return addOne(addOne(addOne(addone(number)))) }

var a = addFive(1); // a == 5

What we'd like is for addFive to boil down to: 
  return number + 5;
or, perhaps, for anywhere we call addFive to be replaced with 
  number + 5;

But that simply cannot happen, because the optimizer cannot generically compose functions into their transformations. Because we have mutables and side effects, it's impossible to 'prove' that the transformation is optimizable into inline or into a for-loop.

So why do we write functional compositions when they are much, much slower than manual for-loops?

With functional composition we get maintainability through testability and readability. In general, our programs do not perform trivial transformations, as from addOne ( x -> x+1) or addFive (x -> x+5). The function calls and the lack of optimization abilities across functions aren't very important in our cases. It's infinitely more important to be able to reason about the system quickly and effectively.

Forget about 'fastest.' Care only about 'fast enough.'

One order of magnitude difference between two different functions is generally insignificant -- your performance problems stem from somewhere else.

## Which to use?

### TL;DR
Just use Lo-dash. It's very fast for most operations, mostly interoperable with underscore (the most popular functional toolchain employed today), and has a ton of helpful aliases and utilities, including many predefined predicates you can just drop in.

### Still reading?
There are two small problems with using lodash, however. One is that it's been written to rely on arrays, which has a wide array of implications. A _.map() over an object gives you.... an array!
```js
var obj = { a: 1, b: 2, c: 3}
_.map(obj, function(num) { return num }) // [1, 2, 3]
```

Another problem this causes is that lodash does not seperate the transformation from the array. A map takes an iterable and returns an array. If you want to chain, you must use the .chain() or rely on the implicit chaining that comes from _(obj).<function> syntax in lodash, before ending with value() in order to transform the wrapper object back into an array.

Lodash also usually evaluates optimistically, so even though, in general, the objects are all transformed independently, you must finish every transform on every item before you can see the results of ANY item.

The other huge problem with lodash is that you cannot create composed functions declaratively, and you cannot curry / partially apply.

For example, in our earlier addFiveToEach:
```js
var addFiveToEach = function(numbers) {
  return _(numbers)
           .chain() // Comonadic wrapping because underscore is imperfect
           .map(addOne)
           .map(addOne)
           .map(addOne)
           .map(addOne)
           .map(addOne)
           .value(); // Retrieve the set of values from the comonad.
};
```

We were forced to wrap our functional chain in an anonymous function, AND make the function accept arrays. We should be able to disconnect the transform which is being applied from the underlying data structure which is holding the data, and we should be able to write our functions as partially applied transformations

Let's define our transform:
```js
var addTwo = function(num) { return addOne(addOne(num)) };
// Why can't we simply say:
var addTwo = compose(addOne, addOne);

// Where compose is the following function:
function compose() {
  var funcs = Array.prototype.slice.call(arguments);
  return function(r) {
    var value = r;
    for(var i=funcs.length-1; i>=0; i--) {
      value = funcs[i](value);
    }
    return value;
  }
}

function compose() {
  var arg = [].slice.call(arguments);
  return function(input) {
    _.reduceRight(arguments, function() {

    });
  }
}

// Then we can do:
var seven = compose(addOne, addOne)(5);
```

Transducers js is a functional composition library which focuses on the act of transforming data. The underlying data structure is unimportant*. This allows you to use arrays, object, trees, graphs, channels, streams, whatever as your source and as your destination, as long as there is a way to build up the final data structure.

For very specific, processing intensive worker jobs which need to be comprehensible but fast, transducers.js is a great choice for building the functional pipeline to handle the problem. However, most of our transforms would be better expressed in Go, once we've proven that they work and do what we want them to do.

*In order to use transducers on the data structure, you must simply implement the ES6 iterator protocal and specify what data structure you want back. Transducers lets you implement toArray, toObj, toIter(able), seq, into, transduce. 
