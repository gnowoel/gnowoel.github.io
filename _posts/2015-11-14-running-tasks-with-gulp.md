---
layout: post
title:  "Running tasks with Gulp"
date:   2015-11-14 12:00:00
---

Even though Gulp is advertised as a build system, a tool we usually use
for transforming project source files, it's an all-around task runner to
the core.

What is more surprising is that Gulp does not provide any fancy API for
defining a task. For this purpose, we just use the plain old JavaScript
functions.

## A simple task

Tasks should be defined in *gulpfile.js*. A simple task that prints out
a greeting message would go like this:

{% highlight js %}
function greet(done) {
  console.log('Hi Gulp!');
  done();
}
{% endhighlight %}

Gulp runs a task asynchronously, and there must be a way to mark the
end. The argument `done` is what we need to signal the completeness of
the task.

A task is always defined as a function, but not all functions in the
Gulpfile are necessarily tasks. To let Gulp know, we must register them
explicitly. The `gulp.task()` method will do this:

{% highlight js %}
gulp.task('greet', greet);
{% endhighlight %}

The first argument of the method is the name of the task, and the second
is the function that defines what to do.

That's pretty much all we need, and here is the complete Gulpfile:

{% highlight js %}
var gulp = require('gulp');

function greet(done) {
  console.log('Hi Gulp!');
  done();
}

gulp.task('greet', greet);
{% endhighlight %}

Now we can run the task by executing the `gulp` command:

    $ gulp greet
    [21:50:03] Using gulpfile ~/project/greet/gulpfile.js
    [21:50:03] Starting 'greet'...
    Hi Gulp!
    [21:50:03] Finished 'greet' after 2.58 ms

The `gulp` command accepts an arbitrary number of task names as options.
In this case, the only name is `greet` and the corresponding task will
be run.

## Task names

When registering a task with `gulp.task()`, the name argument is
optional. If we omit the task name, Gulp will use the name of the
function as the task name.

In our example, because both the task and the function are named
"greet", we can safely omit the first argument:

{% highlight js %}
gulp.task(greet);
{% endhighlight %}

The name of the function can be accessed with the `function.name`
property. However, the property is readonly, as illustrated below:

{% highlight js %}
function fn() {};
console.log(fn.name); //=> 'fn'
fn.name = 'something';
console.log(fn.name); //=> still 'fn'
{% endhighlight %}

This could cause problems when registering an anonymous function without
providing a task name. The `name` property of an anonymous function is
always empty and can't be changed. Gulp will complains about the missing
of a task name:

{% highlight js %}
// Don't do this

var gulp = require('gulp');

var greet = function(done) {
  console.log('Hi Gulp!');
  done();
};

gulp.task(greet);
{% endhighlight %}

The workaround is simple. Instead of setting `function.name`, just give a
value to the `displayName` property:

{% highlight js %}
var gulp = require('gulp');

var greet = function(done) {
  console.log('Hi Gulp!');
  done();
};
greet.displayName = 'greet';

gulp.task(greet);
{% endhighlight %}

Gulp first check the name argument. If it's not available, the function
name will be used. For an anonymous function, it will fallback to the
`displayName` property.

## The default task

When executing the `gulp` command without a task name option, Gulp will
search for the `default` task.

Let's update our task and change its name to `default`:

{% highlight js %}
var gulp = require('gulp');

function greet(done) {
  console.log('Hi Gulp!');
  done();
}

gulp.task('default', greet);
{% endhighlight %}

Now we can run our task with a simple `gulp` command:

    $ gulp
    [21:17:09] Using gulpfile ~/project/greet/gulpfile.js
    [21:17:09] Starting 'default'...
    Hi Gulp!
    [21:17:09] Finished 'default' after 2.56 ms

In practice, we almost always add a default task for our project. It
serves as the entry point of our build process. Imagine this: with a
single `gulp` command, our entire processing workflow will get
automated.
