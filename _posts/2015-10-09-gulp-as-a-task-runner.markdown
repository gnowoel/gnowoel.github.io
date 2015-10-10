---
layout: post
title:  "Gulp as a task runner"
date:   2015-10-09 12:00:00
---

Though advertised as a build system, Gulp is in fact a generic task runner.

In the [README](https://github.com/gulpjs/gulp/blob/47623606afb698f66a4085ad6f73bc7270ad1654/README.md) of Gulp 3, we can find a sample task for cleaning up the destination directory:

{% highlight javascript %}
var gulp = require('gulp');
var del = require('del');

gulp.task('clean', function() {
  return del(['build']);
})
{% endhighlight %}

We don't use any Gulp API in the `clean` task here, just the plain Node.js code.

However, a task defined with `gulp.task` is more of an exported entry point than a reusable code block. We can execute the command `gulp clean` to perform the task, but there's no simple way to invoke it specifically elsewhere within the gulpfile.

To define a reusable code block, the Gulp team suggests simply using JavaScript functions. Applying this idea to our example, we get the following code:

{% highlight javascript %}
var gulp = require('gulp');
var del = require('del');

gulp.task('clean', clean);

function clean() {
  return del(['build']);
}
{% endhighlight %}

Now the functionality has been wrapped in a separate function. Wherever we want to clean up the directory, perhaps within another task, just call the `clean` function.

You may have already noticed, however, there's a catch in the updated task definition. We unnecessarily used the name `clean` twice:

{% highlight javascript %}
gulp.task('clean', clean);
{% endhighlight %}

Apparently, the Gulp team was aware of this redundancy. In the upcoming Gulp 4 release, we have shortcut to do the same thing:

{% highlight javascript %}
gulp.task(clean);
{% endhighlight %}

Note we just pass the function itself as the parameter to the `gulp.task` method. This is a clean solution. It leverages the fact that the function name is already embedded in the function object.

The revised example is exactly what we see in the [README](https://github.com/gulpjs/gulp/blob/13e25e2ab8839cd006b40ea2ed9e6fdf18fff901/README.md) of Gulp 4:

{% highlight javascript %}
var gulp = require('gulp');
var del = require('del');

gulp.task(clean);

function clean() {
  return del(['build']);
}
{% endhighlight %}

No explicit string of name there is in the task definition, but we can still run the task with the `gulp clean` command as before. Just remember to always use a named function for a task. Anonymous functions won't work, even if being assigned to a variable later.

What we learned here is that Gulp 4 now uses plain JavaScript functions as the new task system. The `gulp.task` method should only be used for defining the exported entry points.

If JavaScript functions is the best task system, why would we have to call Gulp a task runner to run a task?
