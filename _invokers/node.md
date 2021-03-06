---
name: node
title: "Node Invoker"
author: projectriff
href: https://github.com/projectriff/node-function-invoker
---

*by [{{page.author}}]({{ page.href }})*

The node function invoker provides a Docker base layer for a function consisting of a single NodeJS module.
It accepts gRPC requests, invokes the command for each request in the input stream, and sends the command's output to the stream of gRPC responses.

## Functions

At runtime, the node function invoker will `require()` the target function module.
This module must export the function to invoke.

```js
// square
module.exports = x => x ** 2;
```

The first argument is the triggering message's payload and the returned value is the resulting message's payload.

### Async

Asynchronous work can be completed by defining either an `async function` or by returning a `Promise`.

```js
// async
module.exports = async x => x ** 2;

// promise
module.exports = x => Promise.resolve(x ** 2);
```

### Streams (experimental)

Streaming functions can be created by setting the `$interactionModel` property on the function to `node-streams`.
The function will then be invoked with two arguments, an `input` [Readable Stream](https://nodejs.org/dist/latest-v8.x/docs/api/stream.html#stream_class_stream_readable) and an `output` [Writeable Stream](https://nodejs.org/dist/latest-v8.x/docs/api/stream.html#stream_class_stream_writable).
Both streams are object streams. Any value returned by the function is ignored, new messages must be written to the output stream.

```js
// echo.js
module.exports = (input, output) => {
    input.pipe(output);
};
module.exports.$interactionModel = 'node-streams';
```

Any npm package that works with Node Streams can be used.

```js
// upperCase.js
const miss = require('mississippi');

const upperCaser = miss.through.obj((chunk, enc, cb) => {
    cb(null, chunk.toUpperCase());
});

module.exports = (input, output) => {
    input.pipe(upperCaser).pipe(output);
};
module.exports.$interactionModel = 'node-streams';
```

The `Content-Type` for output messages can be set with the `$defaultContentType` property. By default, `text/plain` is used. For request-reply function, the `Accept` header is used, however, there is no Accept header in a stream.

```js
// greeter.js
const miss = require('mississippi');

const greeter = miss.through.obj((chunk, enc, cb) => {
    cb(null, {
        greeting: `Hello ${chunk}!`
    });
});

module.exports = (input, output) => {
    input.pipe(greeter).pipe(output);
};
module.exports.$interactionModel = 'node-streams';
module.exports.$defaultContentType = 'application/json';
```

### Lifecycle

Functions that communicate with external services, like a database, can use the `$init` and `$destroy` lifecycle hooks on the function.
These methods are invoked once per function invoker instance, whereas the target function may be invoked multiple times within a single function invoker instance.

The `$init` method is guarenteed to finish before the main function is invoked.
The `$destroy` method is guarenteed to be invoked after all of the main functions are finsished.

```js
let client;

// function
module.exports = async ({key, amount}) => {
    return await client.incrby(key, amount);
};

// setup
module.exports.$init = async () => {
    const Redis = require('redis-promise');
    client = new Redis();
    await client.connect();
};

// cleanup
module.exports.$destroy = async () => {
    await client.quit();
};
```

The lifecycle methods are optional, and should only be implemented when needed.
The hooks may be either traditional or async functions.
Lifecycle functions have up to 10 seconds to complete their work, or the function invoker will abort.
