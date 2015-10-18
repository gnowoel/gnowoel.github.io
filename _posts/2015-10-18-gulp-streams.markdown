---
layout: post
title:  "Gulp streams"
date:   2015-10-18 12:00:00
---

Gulp is not just fast. It's really fast. Memory usage is low and disk I/O is reduced to the bare minimum. Its efficiency comes from [Node.js streams](https://nodejs.org/api/stream.html), or more specifically, [Node.js object streams](https://nodejs.org/api/stream.html#stream_object_mode), which is the mechanism Gulp uses for its inner workings.

A Gulp task usually works on files. It would read files from the disk, operate on them with some [plugins](http://gulpjs.com/plugins/) and finally save the results back, as shown in the following snippet:

{% highlight javascript %}
gulp.task('default', function() {
  return gulp.src('src/**/*')
    .pipe(plugin1)
    .pipe(plugin2)
    .pipe(plugin3)
    .pipe(gulp.dest('dist'));
});
{% endhighlight %}

Since the plugins are chainable with the `pipe()` method, even the most complicated task can be composed this way. As an example, suppose we want to transpile some CoffeeScript files, then minify them, and also concatenate them into one production-ready script. We could chain the [`gulp-coffee`](https://www.npmjs.com/package/gulp-coffee), [`gulp-uglify`](https://www.npmjs.com/package/gulp-uglify) and [`gulp-concat`](https://www.npmjs.com/package/gulp-concat) plugins as below:

{% highlight javascript %}
var coffee = require('gulp-coffee');
var uglify = require('gulp-uglify');
var concat = require('gulp-concat');

gulp.task('scripts', function() {
  return gulp.src('js/**/*')
    .pipe(coffee())
    .pipe(uglify())
    .pipe(concat('all.min.js'))
    .pipe(gulp.dest('dist/js'));
});
{% endhighlight %}

A Gulp plugin is in fact a Node.js object stream in another name. We can think of a pipe chain in a Gulp task as an assembly line. And every plugin is just a worker in front of a workstation. It takes a file from the upstream, do something with it, and then passes it off to the downstream. These steps will be performed again and again until the last file has been handled.

Because a plugin doesn't have to wait for the upstream to complete all its work before getting started, the process is very efficient. Also, by keeping the intermediate files in memory, it avoids the unnecessary disk I/O and further improves the performance.

Node.js core provides interfaces for defining an object stream. For a Gulp plugin, we should subclassing the [Transform](https://nodejs.org/api/stream.html#stream_class_stream_transform) class with the [`util.inherits`](https://nodejs.org/api/util.html#util_util_inherits_constructor_superconstructor) method:

{% highlight javascript %}
var util = require('util');
var Transform = require('stream').Transform;

util.inherits(PassThrough, Transform);

function PassThrough() {
  Transform.call(this, { objectMode: true });
}

PassThrough.prototype._transform = function(file, _, next) {
  // do something with the file
  this.push(file);
  next();
};

PassThrough.prototype._flush = function(done) {
  done();
}
{% endhighlight %}

In this example, we defined a passthrough stream that would hand objects over as-is. There are two methods we should override in the subclass. The `_transform` method is where we specify the operation performed on each file. Here we just use `this.push()` to pass the object off to the downstream. The optional `_flush` method is where we add some cleanup code that will be executed at the end of the stream.

This passthrough stream can be used as any other Gulp plugin, as shown in the following code:

{% highlight javascript %}
var passThrough = new PassThrough();

gulp.task('default', function() {
  return gulp.src('src/**/*')
    .pipe(passThrough)
    .pipe(gulp.dest('dist'));
});
{% endhighlight %}


Though the idea of a stream is not particularly difficult, the implementation is. We have to handle the class inheritance very carefully. In practice, however, we almost never use the Node.js API directly. There is a module called [through2](https://www.npmjs.com/package/through2) that will save us from this awkwardness:

{% highlight javascript %}
var through2 = require('through2');

var passThrough = through2.obj(transform, flush);

function transform(file, _, next) {
  // do something with the file
  this.push(file);
  next();
}

function flush(done) {
  done();
}
{% endhighlight %}

The functionality remains the same, but the interface is much simpler.
