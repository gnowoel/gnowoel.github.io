---
layout: post
title:  "Types of Gulp tasks"
date:   2015-11-18 12:00:00 +0800
---

Gulp is an async task runner. It starts a task but never waits for it to
complete. A task should be defined in a way that Gulp would know where
the end is.

The obvious way is to use a callback to notify Gulp of the task
completion. But this is not the only possibility. Gulp also allows to
define a task with a function that either returns a promise, a stream, a
child process or an observable.

To illustrate the different types of Gulp tasks, we are going to create
the same task in all those possible ways. The requirement is simple:
read a file named `input.txt` and then write the contents to
`output.txt`.

We won't dive deep into each of those technologies. Just give you an
idea of what's available when it comes to defining a Gulp task.

## Using a callback

A callback is just a function being passed in as an argument of another
function. The other function runs asynchronously and once it's done, the
callback will be executed to notify the caller.

A Gulp task that uses a callback follows this pattern:

```js
function taskName(done) {
  // do stuff
  done();
}
```

The `done` argument is a callback function. It's passed by Gulp when invoking the function and used for signaling the end of the task.

Our task is to read file `input.txt` and then write the contents to `output.txt`. For reading and writing files, we could use two methods defined in the [`fs`] module of Node.js: `readFile()` and `writeFile()`.

[`fs`]: https://nodejs.org/api/fs.html

Both methods run asynchronously and take a callback as the last argument. To make sure writing comes after reading, executing `writeFile()` should happen in the callback passed to `readFile()`.

Here's our complete Gulpfile that defines a `copy` task:

```js
var gulp = require('gulp');
var fs = require('fs');

function copy(done) {
  fs.readFile('input.txt', 'utf-8', function(err, contents) {
    if (err) throw err;
    fs.writeFile('output.txt', contents, function(err) {
      if (err) throw err;
      done();
    });
  });
}

gulp.task('default', copy);
```

After we've done writing, the `done` callback will be executed to signal the end of the task.

Node.js also offers methods for reading and writing files
synchronously: `readFileSync()` and `writeFileSync()`. These
methods have similar signatures with the async equivalents, but they
don't take callback arguments.

By using the synchronous reading and writing methods, our Gulpfile can be written as below:

```js
var gulp = require('gulp');
var fs = require('fs');

function copy(done) {
  var contents = fs.readFileSync('input.txt', 'utf-8');
  fs.writeFileSync('output.txt', contents);
  done();
}

gulp.task('default', copy);
```

Generally speaking, the async version is preferred, as the synchronous methods would block the application while being executed.

## Returning a promise

Promises are a smarter way of managing callbacks. They allow to write async code in a synchronous fashion. The async functions are wrapped in the promise objects and then chained together with the `then()` method. The result of the current promise will be passed to the next one in the chain for further processing.

This is our promise version of the Gulp file:

```js
var gulp = require('gulp');
var Promise = require('promise');
var fs = require('fs');

var read = Promise.denodeify(fs.readFile);
var write = Promise.denodeify(fs.writeFile);

function copy() {
  return read('input.txt', 'utf-8')
    .then(function(contents) {
      return write('output.txt', contents)
    });
}

gulp.task('default', copy);
```

The `Promise.denodeify()` method is used for wrapping the native Node.js functions.

## Returning a stream

Node.js streams are similar to Unix pipeline. Multiple stream objects can be chained together with `pipe()` method to form a pipe chain. Data flows all the way through the first stream object to the last one in the pipe chain. Streams are a very efficient way for transferring data.

The stream version of our Gulpfile is shown as follows:

```js
var gulp = require('gulp');
var fs = require('fs');

function copy() {
  var read = fs.createReadStream('input.txt');
  var write = fs.createWriteStream('output.txt');

  return read.pipe(write);
}

gulp.task('default', copy);
```

Here we use `createReadStream()` and `createWriteStream()` from the `fs` module to create readable and writable streams.

## Returning a child process

A child process is used for forking the current process or running an external command. In our case, we could copy the files by issuing a Unix `cp` command.

Her is our updated task:

```js
var gulp = require('gulp');
var spawn = require('child_process').spawn;

function copy() {
  return spawn('cp', ['input.txt', 'output.txt']);
}

gulp.task('default', copy);
```

The `child_process.spawn()` method is what we need for running an external command.

## Returning an observable

Like promises, observables are a wrapper for managing async actions. They are even more generic, in the sense that different types of async objects can be unified with a single set of APIs.

Here's how to use observable for copying files:

```js
var gulp = require('gulp');
var fs = require('fs');
var rx = require('rx');

var read = rx.Observable.fromNodeCallback(fs.readFile);
var write = rx.Observable.fromNodeCallback(fs.writeFile);

function copy() {
  return read('input.txt', 'utf-8')
    .selectMany(function(contents) {
      return write('output.txt', contents);
    });
}

gulp.task('default', copy);
```

As with the promise version, we first wrap the `fs.readFile()` and `fs.writeFile()` methods, and then run them sequentially.
