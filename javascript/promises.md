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

[`rejectionhandled`]
Sent when a promise is rejected, after that rejection has been handled by the executor's `reject` function.

[`unhandledrejection`]
Sent when a promise is rejected but there is no rejection handler available.

## Creating a `Promise` Around an Old Callback API

## Composition

## Timing

## Nesting

## Common Mistakes

## When Promises and Tasks Collide

## References

[Using Promises (MDN)](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Using_promises)<br>
[The `Promise` Object (MDN)](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise)<br>
[JavaScript Promises (Google Web Fundamentals)](https://developers.google.com/web/fundamentals/primers/promises#whats-all-the-fuss-about)

<!-- Reference Links -->
[arrow functions]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/Arrow_functions "Arrow function expressions"
[`async`/`await`]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/async_function "Async function"
[`await`]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/async_function "Async function"
[completion of the current run]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/EventLoop#Run-to-completion "Run-to-completion"
[`fetch()`]: https://developer.mozilla.org/en-US/docs/Web/API/WindowOrWorkerGlobalScope/fetch "WindowOrWorkerGlobalScope.fetch()"
[`preventDefault()`]: https://developer.mozilla.org/en-US/docs/Web/API/Event/preventDefault "Event.preventDefault()"
[`Promise`]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise "Promise"
[`Promise.all()`]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/all "Promise.all()"
[`Promise.race()`]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/race "Promise.race()"
[`Promise.reject()`]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/reject "Promise.reject()"
[`Promise.resolve()`]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/resolve "Promise.resolve()"
[`PromiseRejectionEvent`]: https://developer.mozilla.org/en-US/docs/Web/API/PromiseRejectionEvent "PromiseRejectionEvent"
[promise constructor anti-pattern]: https://stackoverflow.com/questions/23803743/what-is-the-explicit-promise-construction-antipattern-and-how-do-i-avoid-it "What is the explicit promise construction antipattern and how do I avoid it?"
[`promise` property]: https://developer.mozilla.org/en-US/docs/Web/API/PromiseRejectionEvent/promise "PromiseRejectionEvent.promise"
[`reason` property]: https://developer.mozilla.org/en-US/docs/Web/API/PromiseRejectionEvent/reason "PromiseRejectionEvent.reason"
[`rejectionhandled`]: https://developer.mozilla.org/en-US/docs/Web/API/Window/rejectionhandled_event "Window: rejectionhandled event"
[`setTimeout()`]: https://developer.mozilla.org/en-US/docs/Web/API/WindowTimers/setTimeout "WindowOrWorkerGlobalScope.setTimeout()"
[`then()`]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/then "Promise.prototype.then()"
[`window`]: https://developer.mozilla.org/en-US/docs/Web/API/Window "Window"
[`Worker`]: https://developer.mozilla.org/en-US/docs/Web/API/Worker "Worker"
[`unhandledrejection`]: https://developer.mozilla.org/en-US/docs/Web/API/Window/unhandledrejection_event "Window: unhandledrejection event"
