# JavaScript: Promises

## Using Promises

A [`Promise`] is an object representing the eventual completion or failure of an asynchronous operation. Essentially a promise is a returned object to which you attach callbacks, instead of passing callbacks into a function.

**Example:**
Imagine a function, `createAudioFileAsync()`, which asynchronously generates a sound file given a configuration record and two callback functions: one called if the audio file is successfully created, and the other called is an error occurs.

Here's some code that uses `createAudioFileAsync()`:

```javascript
function successCallback(result) {
  console.log("Audio file ready at URL: " + result);
}

function failureCallback(error) {
  console.error("Error generating audio file: " + error);
}

createAudioFileAsync(audioSettings, successCallback, failureCallback);
```

...modern functions return a promise you can attach your callbacks to instead:

If `createAudioFileAsync()` were rewritten to return a promise, using it could be as simple as this:

```javascript
createAudioFileAsync(audioSettings).then(successCallback, failureCallback);
```

That's shorthand for:

```javascript
const promise = createAudioFileAsync(audioSettings);
promise.then(successCallback, failureCallback);
```

We call this an _asynchronous function call_. This convention has several advantages, each of which will be explored below.

## Guarantees

Unlike "old-style," _passed-in_ callbacks, a promise comes with some guarantees:

- Callbacks will never be called before the [completion of the current run] of the JavaScript event loop.
- Callbacks added with `then()` even _after_ the success or failure of the asynchronous operation, will be called, as above.
- Multiple callbacks may be added by calling `then()` several times. Each callback is executed one after another, in the order in which they were inserted.

## Chaining

A common need is to execute two or more asynchronous operations back to back where each subsequent operation starts when the previous operation succeeds, with the result from the previous step. We accomplish this by creating a **promise chain**.

The `then()` function returns a **new promise**, different from the original:

```javascript
const promise = doSomething();
const promise2 = promise.then(successCallback, failureCallback);
```

or

```javascript
const promise2 = doSomething().then(successCallback, failureCallback);
```

The second promise (`promise2`) represents the completion not just of `doSomething()`, but also of the `successCallback` or `failureCallback` you passed in, which can be other asynchronous functions returning a promise. When that's the case, any callbacks added to `promise2` get queued behind the promise returned by either `successCallback` or `failureCallback`.

Basically, each promise represents the completion of another asynchronous step in the chain.

In the old days, doing several asynchronous operations in a row would lead to the classic callback pyramid of doom:

```javascript
doSomething(function(result) {
  doSomethingElse(results, function(newResult) {
    doThirdThing(newResult, function(finalResult) {
      console.log("Got the final result: " + finalResult);
    }, failureCallback);
  }, failureCallback);
}, failureCallback);
```

With modern functions, we attach our callbacks to the returned promises, forming a promise chain:

```javascript
doSomething()
  .then(function(result) {
    return doSomethingElse(result);
  })
  .then(function(newResult) {
    return doThirdThing(newResult);
  })
  .then(function(finalResult) {
    console.log("Got the final result: " + finalResult);
  })
  .catch(failureCallback);
```

The arguments to `then` are optional, and `catch(failureCallback)` is short for `then(null, failureCallback)`. You might see this expressed with [arrow functions] instead:

```javascript
doSomething()
  .then(result => doSomethingElse(result))
  .then(newResult => doThirdThing(newResult))
  .then(finalResult => {
    console.log(`Got the final result: ${finalResult}`);
  })
  .catch(failureCallback);
```

**Important:** Always return results, otherwise callbacks won't catch the result of a previous promise (with arrow functions `() => x` is short for `() => {return x; }`).

### Chaining After a Catch

It's possible to chain _after_ a failure, i.e. a `catch`, which is useful to accomplish new actions even after an action failed on the chain. Just add another `then()` after the `catch`.

## Error Propagation

You might recall seeing `failureCallback` three times in the pyramid of doom earlier, compared to only once at the end of the promise chain:

```javascript
doSomething()
  .then(result => doSomethingElse(result))
  .then(newResult => doThirdThing(newResult))
  .then(finalResult => console.log(`Got the final result: ${finalResult}`))
  .catch(failureCallback);
```

If there's an exception, the browser will look down the chain for `.catch()` handlers or `onRejected`. This is very much modeled after how synchronous code works:

```javascript
try {
  const result = syncDoSomething();
  const newResult = syncDoSomethingElse(result);
  const finalResult = syncDoThirdThing(newResult);
  console.log(`Got the final result: ${finalResult}`);
} catch(error) {
  failureCallback(error);
}
```

This symmetry with asynchronous code culminates in the [`async`/`await`] syntactic sugar in ECMAScript 2017:

```javascript
async function funk() {
  try {
    const result = await doSomething();
    const newResult = await doSomethingElse(result);
    const finalResult = await doThirdThing(newResult);
    console.log(`Got the final result: ${finalResult}`);
  } catch(error) {
    failureCallback(error);
  }
}
```

It builds on promises, e.g. `doSomething()` is the same function as before. Read more about this syntax [here](https://developers.google.com/web/fundamentals/getting-started/primers/async-functions).

Promises solve a fundamental flaw with the callback pyramid of doom, by catching errors, even thrown exceptions and programming errors. This is essential for functional composition of asynchronous operations.

## Promise Rejection Events

Whenever a promise is rejected, one of two events is sent to the global scope (generally, this is either the [`window`] or, if being used in a web worker, it's the [`Worker`] or other worker-based interface). The two events are:

[`rejectionhandled`]<br>
Sent when a promise is rejected, after that rejection has been handled by the executor's `reject` function.

[`unhandledrejection`]<br>
Sent when a promise is rejected but there is no rejection handler available.

In both cases, the event (of type [`PromiseRejectionEvent`]) has as members a [`promise` property] indicating the promise was rejected, and a [`reason` property] that provides the reason given for the promise to be rejected.

These make it possible to offer fallback error handling promises, as well as to help debug issues with your promise management. These handlers are global per context, so all errors will go to the same event handlers, regardless of source.

One case of special usefulness: when writing code for Node.js, it's common that modules you include in your project may have unhandled rejected promises. These get logged to the console by the Node runtime. You can capture these for analysis and handling by your code—or just to avoid having them cluttering up your output—by adding a handler for the [`unhandledrejection`] event, like this:

```javascript
window.addEventListener("unhandledrejection", event => {
  /* You might start here by adding code to examine the promise specified by event.promise and the reason in event.reason */

  event.preventDefault();
}, false);
```

By calling the event's [`preventDefault()`] method, you tell the JavaScript runtime not to do its default action when rejected promises go unhandled. That default action usually involves logging the error to the console, and this is indeed the case for Node.

Ideally, of course, you should examine the rejected promises to make sure none of them are actual code bugs before just discarding these events.

## Creating a `Promise` Around an Old Callback API

A [`Promise`] can be created from scratch using its constructor. This should be needed only to wrap old APIs.

In an ideal world, all asynchronous functions would already return promises. Unfortunately, some APIs still expect success and/or failure callbacks to be passed in the old way. The most obvious example is the [`setTimeout()`] function:

```javascript
setTimeout(() => saySomething("10 seconds passed"), 10*1000);
```

Mixing old-style callbacks and promises is problematic. If `saySomething()` fails or contains a programming error, nothing catches it. `setTimeout()` is to blame for this.

Luckily we can wrap `setTimeout` in a promise. Best practice is to wrap problematic functions at the lowest possible level, and the never call them directly again:

```javascript
const wait = ms => new Promise(resolve => setTimeout(resolve, ms));

wait(10*1000).then(() => saySomething("10 second")).catch(failureCallback);
```

Basically, the promise constructor takes an executor function that lets us resolve or reject a promise manually. Since `setTimeout()` doesn't really fail, we left out reject in this case.

## Composition

[`Promise.resolve()`] and [`Promise.reject()`] are shortcuts to manually create an already resolved or rejected promise respectively. This can be useful at times.

[`Promise.all()`] and [`Promise.race()`] are two composition tools for running asynchronous operations in parallel.

We can start operations in parallel and wait for them to finish like this:

```javascript
Promise.all([func1(), func2(), func3()])
  .then(([result1, result2, result3]) => { /* use result1, result2 and result3 */ });
```

Sequential composition is possible using some clever JavaScript:

```javascript
[func1, func2, func3].reduce((p, f) => p.then(f), Promise.resolve())
  .then(result3 => { /* use result3 */ });
```

Basically, we reduce an array of asynchronous functions down to a promise chain equivalent to: `Promise.resolve().then(func1).then(func2).then(func3);`

This can be made into a reusable compose function, which is common in functional programming:

```javascript
const applyAsync = (acc, val) => acc.then(val);
const composeAsync = (...funcs) => x => funcs.reduce(applyAsync, Promise.resolve(x));
```

The `composeAsync()` function will accept any number of functions as arguments, and will return a new function that accepts an initial value to be passed through the composition pipeline:

```javascript
const transformData = composeAsync(func1, func2, func3);
const result3 = transformData(data);
```

In ECMAScript 2017, sequential composition can be done more simply with async/await:

```javascript
let result;
for (const f of [func1, func2, func3]) {
  result = await f(result);
}
/* use last result (i.e. result3) */
```

## Timing

To avoid surprises, functions passed to [`then()`] will never be called synchronously, even with an already-resolved promise:

```javascript
Promise.resolve().then(() => console.log(2));
console.log(1); // 1, 2
```

Instead of running immediately, the passed-in function is put on a microtask queue, which means it runs later when the queue is emptied at the end of the current run of the JavaScript event loop, i.e. pretty soon:

```javascript
const wait = ms => new Promise(resolve => setTimeout(resolve, ms));

wait().then(() => console.log(4));
Promise.resolve().then(() => console.log(2)).then(() => console.log(3));
console.log(1); // 1, 2, 3, 4
```

## Nesting

Simple promise chains are best kept flat without nesting, as nesting can be a result of careless composition.

Nesting is a control structure to limit the scope of `catch` statements. Specifically, a nested `catch` only catches failures in its scope and below, not errors higher up the chain outside the nested scope. When used correctly, the gives greater precision in error recovery:

```javascript
doSomethingCritical()
  .then(result => doSomethingOptional(result)
    .then(optionalResult => doSomethingExtraNice(optionalResult))
    .catch(e => {})) // Ignore is optional stuff fails; proceed.
  .then(() => moreCriticalStuff())
  .catch(e => console.error("Critical failure: " + e.message));
```

Note that the optional steps here are nested, not from the indentation, but from the placements of the outer parentheses around them.

The inner neutralizing `catch` statement only catches failures from `doSomethingOptional()` and `doSomethingExtraNice()`, after which the code resumes with `moreCriticalStuff()`. Importantly, if `doSomethingCritical()` fails, its error is caught by the final (outer) `catch` only.

## When Promises and Tasks Collide

If you run into situations in which you have promises and tasks (such as events or callbacks) which are firing in unpredictable orders, it's possible you may benefit from using a microtask to check status or balance out your promises when promises are created conditionally.

One situation in which microtasks can be used to ensure that the ordering of execution is always consistent is when promises are used in one clause of an `if...else` statement (or other conditional statement), but not the other clause. Conside code such as this:

```javascript
customElement.prototype.getData = url => {
  if (this.cache[url]) {
    this.data = this.cache[url];
    this.dispatchEvent(new Event("load"));
  } else {
    fetch(url).then(result => result.arrayBuffer()).then(data => {
      this.cache[url] = data;
      this.data = data;
      this.dispatchEvent(new Event("load"));
    });
  }
};
```

The problem introduced here is that by using a task in one branch of the `if...else` statement (in the case in which the image is available in the cache) but having promises involved in the `else` clause, we have a situation in which the order of operations can vary; for example as seen below.

```javascript
element.addEventListener("load", () => console.log("Loaded data"));
console.log("Fetching data...");
element.getData();
console.log("Data fetched");
```

Executing this code twice in a row gives the results shown in the table below:

**Results when data isn't cached (left) vs. when it is found in the cache (right)**

| **Data isn't cached** | **Data is cached** |
| --- | --- |
| Fetching data | Fetching data |
| Data fetched | Loaded data |
| Loaded data | Data fetched |

Even worse, sometimes the element's `data` property will be set and other times it won't be by the time this code finishes running.

We can ensure consistent ordering of these operations by using a microtask in the `if` clause to balance the two clauses:

```javascript
customElement.prototype.getData = url => {
  if (this.cache[url]) {
    queueMicrotask(() => {
      this.data = this.cache[url];
      this.dispatchEvent(new Event("load"));
    });
  } else {
    fetch(url).then(result => result.arrayBuffer()).then(data => {
      this.cache[url] = data;
      this.data = data;
      this.dispatchEvent(new Event("load"));
    });
  }
};
```

This balances the clauses by having both situations handles the setting of `data` and firing of the `load` event within a microtask (using `queueMicrotask()` in the `if` clause and using the promises used by [`fetch()`] in the `else` clause).

If you think microtasks may help solve this problem, see the [microtask guide] to learn more about how to use [`queueMicrotask()`] to enqueue a function as a microtask.

## References

[Using Promises (MDN)](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Using_promises)<br>
[The `Promise` Object (MDN)](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise)<br>
[JavaScript Promises (Google Web Fundamentals)](https://developers.google.com/web/fundamentals/primers/promises#whats-all-the-fuss-about)<br>
[Promise, aysnc/await (Javascript.info)](https://javascript.info/promise-basics)

<!-- Reference Links -->
[arrow functions]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/Arrow_functions "Arrow function expressions"
[`async`/`await`]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/async_function "Async function"
[completion of the current run]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/EventLoop#Run-to-completion "Run-to-completion"
[`fetch()`]: https://developer.mozilla.org/en-US/docs/Web/API/WindowOrWorkerGlobalScope/fetch "WindowOrWorkerGlobalScope.fetch()"
[microtask guide]: https://developer.mozilla.org/en-US/docs/Web/API/HTML_DOM_API/Microtask_guide "Using microtasks in JavaScript with queueMicrotask()"
[`preventDefault()`]: https://developer.mozilla.org/en-US/docs/Web/API/Event/preventDefault "Event.preventDefault()"
[`Promise`]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise "Promise"
[`Promise.all()`]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/all "Promise.all()"
[`Promise.race()`]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/race "Promise.race()"
[`Promise.reject()`]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/reject "Promise.reject()"
[`Promise.resolve()`]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/resolve "Promise.resolve()"
[`PromiseRejectionEvent`]: https://developer.mozilla.org/en-US/docs/Web/API/PromiseRejectionEvent "PromiseRejectionEvent"
[`promise` property]: https://developer.mozilla.org/en-US/docs/Web/API/PromiseRejectionEvent/promise "PromiseRejectionEvent.promise"
[`queueMicrotask()`]: https://developer.mozilla.org/en-US/docs/Web/API/WindowOrWorkerGlobalScope/queueMicrotask "WindowOrWorkerGlobalScope.queueMicrotask()"
[`reason` property]: https://developer.mozilla.org/en-US/docs/Web/API/PromiseRejectionEvent/reason "PromiseRejectionEvent.reason"
[`rejectionhandled`]: https://developer.mozilla.org/en-US/docs/Web/API/Window/rejectionhandled_event "Window: rejectionhandled event"
[`setTimeout()`]: https://developer.mozilla.org/en-US/docs/Web/API/WindowTimers/setTimeout "WindowOrWorkerGlobalScope.setTimeout()"
[`then()`]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/then "Promise.prototype.then()"
[`window`]: https://developer.mozilla.org/en-US/docs/Web/API/Window "Window"
[`Worker`]: https://developer.mozilla.org/en-US/docs/Web/API/Worker "Worker"
[`unhandledrejection`]: https://developer.mozilla.org/en-US/docs/Web/API/Window/unhandledrejection_event "Window: unhandledrejection event"
