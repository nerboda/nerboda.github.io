---
layout: post
title: 'Broken Promises: Building JavaScript Promises from scratch'
date: 2018-05-03 23:00 -0600
---

The Broken Promises part of the title is actually referring to the weekend plans
I bailed on in order to stay home and dig into the internals of Javascript
Promises. What you see below is a limited but functioning implementation of
Javascript Promises, along with an imitation fetch API (named `get`) to prove it
works.

First, a quick refresher on what a Promise is and what it does.

### Promises

In the simplest terms, a Promise allows you to run some code asynchronously
(without blocking the execution of the rest of your program) and then run other
code after it completes and dependent on the result (whether it failed or
succeeded).

More importantly, it does this in a way that avoids what JS developers have
referred to over the years as “Callback Hell”.

#### Terminology

I used the words “complete”, “failed” and “succeeded” above, but those aren’t
the terms actually used by Promises. Instead, the words “settled”, “resolved”
and “rejected” are used, respectively. Prior to a promise “settling”, it’s said
to be “pending”.

#### The Promise Constructor

Promises are created using the constructor pattern. The Promise constructor
function takes 1 argument, a Function with the following signature:

`function(resolveCallback, rejectCallback)`

* resolveCallback — a function to be called if the promise resolves.
* rejectCallback — a function to be called if the promise is rejected.

*NOTE: see the **Using Promises** section below for details on building the
body of the function passed to the constructor.*

The constructor returns a Promise object which keeps track of it’s state
(status), and can only be settled once. This means that the resolver and
rejector functions together can only receive one call, and the status can’t
change from resolved to rejected or vice versa.

#### The Interface

Promises implement the **Thenable** interface (actually I believe they’re the
originator of the term), which simply means you can call `then` on them to run
code after their completion.

`then` **: Function**

* Takes a callback to be invoked on resolve.
* The callback should take one argument, the value returned from the resolver.
* Returns itself so methods can be chained.

They also expose a `catch` method, which is like `then` except that it runs only
if the promise rejects.

`catch` **: Function**

* Takes a callback to be invoked on reject.
* The callback should take one argument, the value returned from the rejector.
* Returns itself so methods can be chained.

Lastly, they expose a `finally` method, which is called after the promise
settles and the `then` or `catch` callbacks are invoked, regardless of whether
it rejects or resolves.

`finally` **: Function**

* Waits for the promise to settle and for the `then` or `catch` callbacks to run.
* Takes a callback to be invoked no matter what.

#### My Implementation

```javascript
function MyPromise(action) {
  this.status = 'pending';
  this.value = undefined;

  this.thenCallbacks = [];
  this.onCatch = undefined;
  this.onFinally = undefined;

  this.then = function(callback) {
    this.thenCallbacks.push(callback);
    return this;
  };

  this.catch = function(callback) {
    this.onCatch = callback;
    return this;
  };

  this.finally = function(callback) {
    this.onFinally = callback;
    return this;
  };

  // Do the thing
  action(resolver.bind(this), rejector.bind(this));

  // Private
  function resolver(value) {
    this.status = 'resolved';
    this.value = value;
    this.thenCallbacks.forEach(function(func) {
      func(this.value);
    }, this);

    if (typeof this.finallyCallback === 'function') {
      this.finallyCallback(this.value);
    }
  }

  function rejector(value) {
    this.status = 'rejected';
    this.value = value;

    if (typeof this.catchCallback === 'function') {
      this.catchCallback(this.value);
    }

    if (typeof this.finallyCallback === 'function') {
      this.finallyCallback(this.value);
    }
  }
}
```

#### Using Promises

If you’re a javascript developer, you’re aware of Promises and you’ve almost
surely benefited from them while using something like the `fetch` API. What’s
less likely however is that you’ve created one directly. Let’s change that.

As mentioned above, the Promise constructor takes a single argument, a function
which itself takes 2 callback functions. The internal logic of that function is
the important part, that logic being: when should the `resolve` callback be
called and when should the `reject` callback be called.

Let’s take a look at how to implement this using `MyPromise` (this will be no
different than using the native Promise provided by Javascript).

```javascript
function get(url) {
  return new MyPromise(function(resolve, reject) {
    var request = new XMLHttpRequest();
    request.open('GET', url);

    request.onload = function() {
      if (request.status === 200) {
        resolve(request.response);
      }
      else {
        reject(Error(request.statusText));
      }
    };

    request.onerror = function() {
      reject(Error('Network Error'));
    };

    request.send();
  });
}
```

It should be pretty straightforward to understand what I’m doing here: I define
a XMLHttpRequest object and attach a couple event listeners to it. If the
response has a status of 200, I pass it to the `resolve` callback, otherwise
it’s status is passed inside an Error object to `reject`. If the request itself
throws an error, I pass an Error with the message ‘Network Error’ to `reject`.

Let’s test it out.

```javascript
get('https://nerboda.github.io')
  .then(function(response) {
    console.log(response);
  })
  .catch(function(error) {
    console.log(error);
  })
  .then(function(result) {
    console.log(result);
  });
```

Look at that, the thing works!

I wrote this blog post primarily as a way to improve my own understanding of the
internals of JS Promises, but I hope it’s been helpful to you too. If I missed
anything important or used terminology that’s misleading, please let me know.

Thanks for reading!
