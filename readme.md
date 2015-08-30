# thread.js

Javascript thread library. Uses web workers when run in browsers and child processes
when run by node.js. Also supports browsers which do not support web workers.

- Convenience API
- For client and server use
- Use different APIs (web worker, node child_process) transparently
- Thread pools
- example use cases
- ES6 and backwards-compatible
- Planned features: Shared workers


## How To

### Basic use

```javascript
import { spawn } from 'thread.js';
// ES5 syntax: var spawn = require('thread.js').spawn;

const thread = spawn(function(input, done) {
  // Everything we do here will be run in parallel in another execution context.
  // Remember that this function will be executed in the thread's context,
  // so you cannot reference any value of the surrounding code.
  done({ string : input.string, integer : parseInt(input.string) });
});

thread
  .send({ string : '123' })
  .on('message', function(response) {
    console.log('123 * 2 = ', response.integer * 2);
    thread.kill();
  })
  .on('error', function(error) {
    console.error('Worker errored:', error);
  })
  .on('exit', function() {
    console.log('Worker has been terminated.');
  });
```


### Thread code in separate files

You don't have to write the thread's code inline. The file is expected to be a
commonjs module (so something that uses `module.exports = ...`), for node and
browser.

```javascript
import { config, spawn } from 'thread.js';
// ES5 syntax: var config = require('thread.js').config, spawn = require('thread.js').spawn;

// Set base paths to thread scripts
config.set({
  basepath : {
    browser : 'http://myserver.local/thread-scripts',
    node    : __dirname + '/../thread-scripts'
  }
});

const thread = spawn('worker.js');

thread
  .send({ do : 'Something awesome!' })
  .on('message', function(message) {
    console.log('Worker sent:', message);
  });
```


### Thread Pool

You can also create a thread pool that spawns a fixed no. of workers. Pass jobs
to the thread pool which it will queue and pass to the next idle worker.

```javascript
import { Pool } from 'thread.js';
// ES5 syntax: var Pool = require('thread.js').Pool;

const pool = new Pool();
const jobA = pool.run('/path/to/worker').send({ do : 'something' });
const jobB = pool.run(
  function(string, done) {
    const hash = md5(string);
    done(hash);
  }, {
    // dependencies; resolved using node's require() or the web workers importScript()
    md5 : 'js-md5'
  }
).send('Hash this string!');

jobA
  .on('done', function(message) {
    console.log('Job A done:', message);
  })
  .on('error', function(error) {
    console.error('Job A errored:', error);
  });

pool
  .on('done', function(job, message) {
    console.log('Job done:', job);
  })
  .on('error', function(job, error) {
    console.error('Job errored:', job);
  })
  .on('spawn', function(worker, job) {
    console.log('Thread pool spawned a new worker:', worker);
  })
  .on('kill', function(worker) {
    console.log('Thread pool killed a worker:', worker);
  })
  .on('finished', function() {
    console.log('Everything done, shutting down the thread pool.');
    pool.destroy();
  })
  .on('destroy', function() {
    console.log('Thread pool destroyed.');
  });
```

### Streaming

You can also spawn a thread for streaming purposes. The following example shows
a very simple use case where you keep feeding numbers to the background task
and it will return the minimum and maximum of all values you ever passed.

```javascript
thread
  .run(function minmax(int, done) {
    if (typeof this.min === 'undefined') {
      this.min = int;
      this.max = int;
    } else {
      this.min = Math.min(this.min, int);
      this.max = Math.max(this.max, int);
    }
    done({ min : this.min, max : this.max }});
  })
  .send(2)
  .send(3)
  .send(4)
  .send(1)
  .send(5)
  .on('message', function(minmax) {
    console.log('min:', minmax.min, ', max:', minmax.max);
  })
  .on('done', function() {
    thread.kill();
  });
```

### Transferable objects

You can also use transferable objects to improve performance when passing large
buffers (in browser). Add script files you want to run using importScripts()
(if in browser) as second parameter to thread.run().
See [Transferable Objects: Lightning Fast!](http://updates.html5rocks.com/2011/12/Transferable-Objects-Lightning-Fast).

```javascript
const largeArrayBuffer = new Uint8Array(1024*1024*32); // 32MB
const data = { label : 'huge thing', buffer: largeArrayBuffer.buffer };

thread
  .run(function(input, done) {
    // do something cool with input.label, input.buffer
    done();
  }, [
    // this file will be run in the thread using importScripts() if in browser
    '/dependencies-bundle.js'
  ])
  // pass the buffers to transfer into thread context as 2nd parameter to send()
  .send(data, [ largeArrayBuffer.buffer ]);
```

### Use external dependencies

TODO
-> gulp task to bundle deps using browserify and expose all of them -> dependency bundle
-> dependency bundle can be imported by importScripts()
-> code can just call `var axios = require('axios');`, no matter if browser or node.js


## API

You can find the API documentation in the [wiki](https://github.com/andywer/thread.js/wiki).


## License

This library is published under the MIT license. See [LICENSE](https://raw.githubusercontent.com/andywer/thread.js/master/LICENSE) for details.
