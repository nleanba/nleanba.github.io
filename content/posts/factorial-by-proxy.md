+++
title = "Factorial by Proxy"
date = "2024-04-16T22:45:49+02:00"
author = "nleanba"
tags = ["js", "programming"]
keywords = []
description = "Using proxy objects to provide an array of all the factorials (also works for fibonacci numbers or other recursively defined sequences)"
showFullContent = false
readingTime = true
hideComments = false
color = "" #color from the theme settings
draft = false
math = false
+++

A cool snippet came up with for calculating recursively defined sequences with built-in memoization.

```js
const factorial = new Proxy([1], {
    get(f, n) {
      return f[n] ?? (f[n] = this.get(f, n-1) * n);
    }
});
```

Use like this:

```js
factorial[0] // 1
factorial[1] // 1
factorial[2] // 2
factorial[3] // 6
             // ...
```

## Explanation:

The proxy allows us to overwrite property access on the array using a [custom get function,](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy/Proxy/get) which we ~~abuse~~ use here to get a seemingly infinite array containig the entire sequence.

Within it, we can _set_ and _get_ properties of the underlying array normally: `f[k] = 3` and `return f[n]` work as expected (the array is available as `f`).
However, for recursion to work we want to use our new get, so we use `this.get(f, k)` to call the our new getter recursively,
calculating unknown (`undefined`) values along the way.

Our get function does the follwing: if `f[n]` is already set—i.e. not undefied (or false)—it is returned instantly.
Otherwise, we calculate it recursively, assign the calculated value to `f[n]` and return that value.

Note that we don’t overwrite anything to do with iterators, so `factorial.forEach(...)` will only iterate over previously calulated values.
If you want to be fancy, you could make it so that iterating works, that the underlying array cannot be modified other than from eithin the getter and maybe some handling for very large inputs or inputs &lt; 0.

Judgin by the [browser compatibility for handler.get() listed on mdn,](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy/Proxy/get#browser_compatibility) this should work everywhere in browsers updated since 2016.

## Other sequences:

Here’s how it looks for the fibonacci numbers:
```js
const fibonacci = new Proxy([0,1], {
    get(f, n) {
      return f[n] ?? (f[n] = this.get(f, n-2) + this.get(f, n-1));
    }
});

fibonacci[0] // 1
fibonacci[1] // 1
fibonacci[2] // 2
fibonacci[3] // 5
             // ...
```

Build your own:
```js
const yourSequence = new Proxy([/* array containing the first elements of the sequence */], {
    get(f, n) {
      return f[n] ?? (f[n] = /* formula to caclulate the n-th element,
                                use `this.get(f, k)` instead of `f[k]` or `yourSequence[k]` */);
    }
});
```