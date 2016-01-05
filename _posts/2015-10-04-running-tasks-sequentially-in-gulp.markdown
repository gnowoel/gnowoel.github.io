---
layout: post
title:  "Running tasks sequentially in Gulp"
date:   2015-10-04 12:00:00
---

Gulp tasks can be run in sequence. To specify the dependencies of a task, we just list their names in an array. Suppose we have task `a`, `b` and `c`, and our `default` task gets start only when all of those are complete. The relationship can be structured as follows:

``` javascript
var gulp = require('gulp');

gulp.task('a', function(cb) {
  setTimeout(cb, 1000);
});

gulp.task('b', function(cb) {
  setTimeout(cb, 1000);
});

gulp.task('c', function(cb) {
  setTimeout(cb, 1000);
});

gulp.task('default', ['a', 'b', 'c']);
```

Task `a`, `b` and `c` are specified as dependencies of task `default`. From the output of executing this script, we can verify that the `default` task doesn't run until others are finished:

    [21:17:19] Starting 'a'...
    [21:17:19] Starting 'b'...
    [21:17:19] Starting 'c'...
    [21:17:20] Finished 'a' after 1 s
    [21:17:20] Finished 'b' after 1 s
    [21:17:20] Finished 'c' after 1 s
    [21:17:20] Starting 'default'...
    [21:17:20] Finished 'default' after 13 μs

However, there's one thing we should be aware of. Though the `default` task comes last, there's no particular order of execution among the dependent tasks themselves. In our case, we see the overlapped running times of task `a`, `b` and `c`. They all start in parallel and none waits for others to complete.

This behavior of Gulp is by design, which allows to run the tasks with maximum concurrency. But what if we want to run the tasks one after the other, like `c`, `b`, `a` and then `default`?

There are several ways to achieve this. We can find a simple but somewhat cumbersome solution in the official docs. Or we can instead use the [run-sequence](https://www.npmjs.com/package/run-sequence) plugin, which solves the problem beautifully. In the beta release of Gulp 4, there's even a built-in `series()` method just for overcoming this shortcoming.

## The simple solution

In the official wiki page, *[Running tasks in series](https://github.com/gulpjs/gulp/blob/master/docs/recipes/running-tasks-in-series.md)*, we can learn a simple way to run multiple tasks one after another. The idea is always specifying a single dependency for any given task. If there's only one element in the dependency array, no competition would ever happen.

For example, we could make task `a` depend on `b`, and `b` depend on `c`. In task `default`, we just specify `a` as the single dependency. To apply this trick to our code, we get this listing:

```js
var gulp = require('gulp');

gulp.task('a', ['b'], function(cb) {
  setTimeout(cb, 1000);
});

gulp.task('b', ['c'], function(cb) {
  setTimeout(cb, 1000);
});

gulp.task('c', function(cb) {
  setTimeout(cb, 1000);
});

gulp.task('default', ['a']);
```

Let's check the result:

    [22:09:23] Starting 'c'...
    [22:09:24] Finished 'c' after 1 s
    [22:09:24] Starting 'b'...
    [22:09:25] Finished 'b' after 1 s
    [22:09:25] Starting 'a'...
    [22:09:26] Finished 'a' after 1 s
    [22:09:26] Starting 'default'...
    [22:09:26] Finished 'default' after 12 μs

Comparing to the previous output, where it took about 1 second to complete all tasks, now it runs the tasks in predefined order and takes about 3 seconds in total.

## The run-sequence plugin

In version 3, the stable release as of this writing, Gulp is an instance of [Orchestrator](https://www.npmjs.com/package/orchestrator). Orchestrator is a task system, which is responsible for defining and running tasks, as well as managing task dependencies. In turn, Orchestrator is an instance of Node.js built-in [EventEmitter](https://nodejs.org/api/events.html) class.

By "instance", we mean "subclass". Subclassing can be idiomatically implemented with Node.js [`util.inherits()`](https://nodejs.org/api/util.html#util_util_inherits_constructor_superconstructor) method, as illustrated with the following code:

```js
function Gulp() {
  Orchestrator.call(this);
}
util.inherits(Gulp, Orchestrator);
```

This snippet is extracted from Gulp [source](https://github.com/gulpjs/gulp/blob/47623606afb698f66a4085ad6f73bc7270ad1654/index.js#L9-L12). We can find similar code in Orchestrator [source](https://github.com/orchestrator/orchestrator/blob/fa11e5e2cbbf735f321d8c19f29c00b8d46058c4/index.js#L10-L17).

From Orchestrator, Gulp inherits the task managing methods. From EventEmitter, it gains the ability to emit and handle events. Under the hood, Gulp emits a `task_stop` event on completion of a task. When running some task with dependencies, it will be delayed until all the `task_stop` events from the dependencies have been received.

The [`run-sequence`](https://www.npmjs.com/package/run-sequence) plugin provides a clean solution to serializing tasks by intercepting the `task_stop` events. If task `a` depends on task `b`, it will run `a` when the `task_stop` event from `b` has arrived. What it does is basically splitting dependencies and running them one by one. We could simulate this behavior by defining tasks on their own, without dependencies, and then run them manually in order:

    $ gulp c
    $ gulp b
    $ gulp a
    $ gulp

Let's rewrite the previous example with `run-sequence`:

```js
var gulp = require('gulp');
var runSequence = require('run-sequence');

gulp.task('a', function(cb) {
  setTimeout(cb, 1000);
});

gulp.task('b', function(cb) {
  setTimeout(cb, 1000);
});

gulp.task('c', function(cb) {
  setTimeout(cb, 1000);
});

gulp.task('default', function(cb) {
  runSequence('c', 'b', 'a', cb);
});
```

From the output of execution, we see the tasks are running in series:

    [01:31:48] Starting 'default'...
    [01:31:48] Starting 'c'...
    [01:31:49] Finished 'c' after 1 s
    [01:31:49] Starting 'b'...
    [01:31:50] Finished 'b' after 1.01 s
    [01:31:50] Starting 'a'...
    [01:31:51] Finished 'a' after 1 s
    [01:31:51] Finished 'default' after 3.01 s

The dependencies are specified as arguments to the `runSequence()` method. If one of the arguments is an array, the tasks in the array will be run in parallel. For example, we could update our `default` task as below:

```js
gulp.task('default', function(cb) {
  runSequence('c', ['b', 'a'], cb);
});
```

Now task `c` runs before both `b` and `a`, but `b` and `a` will run concurrently.

## The series() and parallel() methods

In the upcoming Gulp 4, Orchestrator will be replaced with [Undertaker](https://www.npmjs.com/package/undertaker) as the task system. Undertaker introduced the `series()` and `parallel()` methods for running tasks in series or in parallel. The exact feature is defined in the [Bach](https://www.npmjs.com/package/bach) module, which is a minimal async library Undertaker relies on.

Those new methods are very intuitive to use. To continue with our example, we could udpate the code with `series()` in Gulp 4:

```js
var gulp = require('gulp');

gulp.task('a', function(cb) {
  setTimeout(cb, 1000);
});

gulp.task('b', function(cb) {
  setTimeout(cb, 1000);
});

gulp.task('c', function(cb) {
  setTimeout(cb, 1000);
});

gulp.task('default', gulp.series('c', 'b', 'a'));
```

Here the output from running it:

    [02:27:31] Starting 'default'...
    [02:27:31] Starting 'c'...
    [02:27:32] Finished 'c' after 1 s
    [02:27:32] Starting 'b'...
    [02:27:33] Finished 'b' after 1 s
    [02:27:33] Starting 'a'...
    [02:27:34] Finished 'a' after 1 s
    [02:27:34] Finished 'default' after 3.01 s

If some of the dependencies should be run concurrently, we could even mix the `series()` and `paralle()` methods:

```js
gulp.task('default', gulp.series('c', gulp.parallel('b', 'a')));
```

This is similar to what we have done with the `run-sequence` plugin.

Gulp 4 has yet to be released. We need to install it from the [`4.0`](https://github.com/gulpjs/gulp/tree/4.0) branch of the GitHub repo:

    $ npm uninstall gulp -g
    $ npm uninstall gulp

    $ npm install gulpjs/gulp-cli#4.0 -g
    $ npm install gulpjs/gulp#4.0 --save-dev

The commands for uninstalling and installing local Gulp should be run from the project root directory.

## Conclusion

If you're using the stable version of Gulp (v3), try either [the simple solution](#the-simple-solution) or [the run-sequence plugin](#the-run-sequence-plugin). Gulp 4 has built-in support for running tasks in series or in parallel. If you're living on the edge, use [the series() and parallel() methods](#the-series-and-parallel-methods) instead.
