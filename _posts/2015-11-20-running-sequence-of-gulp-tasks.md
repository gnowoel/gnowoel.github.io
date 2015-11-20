---
layout: post
title:  "Running sequence of Gulp tasks"
date:   2015-11-20 12:00:00
---

Modern web development involves many tools with new ones coming out each day. The essence of the web is still the same: HTML, CSS and JavaScript. However the workflow has changed radically. For building a web app today, we might use authoring tools like CoffeeScript and Sass instead of writing JavaScript and CSS by hand, and only in the last step we convert them into something that browsers recognize.

As a build tool, Gulp is capable of managing these complexities and automating the whole workflow.

## Tasks in series and parallel

Gulp used to lack an effective way of managing complex workflow. The problem is solved with the introduction of the `gulp.series()` and `gulp.parallel()` methods in Gulp 4.

As the names of the methods imply, they are used for running tasks in series or in parallel. The signatures of them are the same. Both accept an arbitrary number of arguments, with each being a task definition function or a registered task name.

For example, we could run two tasks in series:

{% highlight js %}
gulp.series('task1', 'task2');
{% endhighlight %}

Here `task1` and `task2` are names of tasks that have been registered with `gulp.task()`. Task names and definition functions can be used interchangeably:

{% highlight js %}
gulp.parallel('task1', function task2() {});
{% endhighlight %}

This time, two tasks will run in parallel. The first is specified as a registered task name and the second is simply a task definition.

Executing `gulp.series()` or `gulp.parallel()` will not run the task combo. It just returns a function that can be used as a valid task definition. This means these methods can be nested together, as shown below:

{% highlight js %}
var combo = gulp.series('task1', gulp.parallel('task2', 'task3'));
{% endhighlight %}

In this case, The task named `task1` would run first. After it's done, `task2` and `task3` would start at the same time.

We also saved the return value of the invocation `gulp.series()` in this example. To make it runnable, we just need to register it with `gulp.task()`:

{% highlight js %}
gulp.task(combo);
{% endhighlight %}

Now the tasks can be run by issuing a `gulp combo` command.

## A complete example

Let's take an example to show how `gulp.series()` and `gulp.parallel()` can be used in practice.

Consider we use CoffeeScript and Sass during development. Before transpiling CoffeeScript and preprocessing Sass code, we want to delete the destination directory to remove the files from a previous build. When all those are ready, we notify the browser to refresh the web pages.

The complete Gulpfile would look something like:

{% highlight js %}
var gulp = require('gulp');

gulp.task('clean', function(done) {
  console.log('Clearing out files');
  done();
});

gulp.task('scripts', function(done) {
  console.log('Traspiling CoffeeScript');
  done();
});

gulp.task('styles', function(done) {
  console.log('Preprocessing Sass');
  done();
});

gulp.task('livereload', function(done) {
  console.log('Reloading pages')
  done();
});

gulp.task('default', gulp.series(
  'clean',
  gulp.parallel('scripts', 'styles'),
  'livereload'
));
{% endhighlight %}

Here we just show a line of message for each task since the real implementations are not relevant. When running the `default` task with the `gulp` command, we get the following output:

    $ gulp
    [21:36:03] Using gulpfile.js
    [21:36:03] Starting 'default'...
    [21:36:03] Starting 'clean'...
    Clearing out files
    [21:36:03] Finished 'clean' after 1.07 ms
    [21:36:03] Starting 'parallel'...
    [21:36:03] Starting 'scripts'...
    [21:36:03] Starting 'styles'...
    Traspiling CoffeeScript
    [21:36:03] Finished 'scripts' after 655 μs
    Preprocessing Sass
    [21:36:03] Finished 'styles' after 978 μs
    [21:36:03] Finished 'parallel' after 1.44 ms
    [21:36:03] Starting 'livereload'...
    Reloading pages
    [21:36:03] Finished 'livereload' after 322 μs
    [21:36:03] Finished 'default' after 5.84 ms

From the log messages, the `clean` task runs first. Then, Gulp starts the `scripts` and `styles` tasks simultaneously. And finally, the `livereload` task gets run after both `scripts` and `styles` are finished.
