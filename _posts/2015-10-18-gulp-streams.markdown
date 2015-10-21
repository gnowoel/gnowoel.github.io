---
layout: post
title:  "Gulp streams"
date:   2015-10-18 12:00:00
---

Gulp is fast, really fast. Memory usage is kept low and disk I/O is reduced to the bare minimum. This efficiency comes from [Node.js streams](https://nodejs.org/api/stream.html), or more specifically, Node.js [object](https://nodejs.org/api/stream.html#stream_object_mode) [transform](https://nodejs.org/api/stream.html#stream_class_stream_transform) streams, which is the mechanism Gulp uses for its inner workings.

A Gulp task usually works on file objects. It would read files from the disk, operate on them with some [plugins](http://gulpjs.com/plugins/) and finally save the results back, as shown in the following snippet:

{% highlight javascript %}
gulp.task('default', function() {
  return gulp.src('src/**/*')
    .pipe(plugin1)
    .pipe(plugin2)
    .pipe(gulp.dest('dist'));
});
{% endhighlight %}

Since the plugins are chainable with the [`pipe()`](https://nodejs.org/api/stream.html#stream_readable_pipe_destination_options) method, even the most complicated task can be composed this way. As an example adapted from Gulp [README](https://github.com/gulpjs/gulp/blob/1ab1d2ad9ece791cf19b80c8f13fd02b05949a1e/README.md#sample-gulpfilejs), suppose we want to transpile some CoffeeScript files, then minify them, and also concatenate them into one production-ready script. We could chain the [`gulp-coffee`](https://www.npmjs.com/package/gulp-coffee), [`gulp-uglify`](https://www.npmjs.com/package/gulp-uglify) and [`gulp-concat`](https://www.npmjs.com/package/gulp-concat) plugins as below:

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

If we can think of a pipe chain in a Gulp task as an assembly line, then every plugin is just a worker in front of a workstation. It takes a file object from the upstream, act on it, and then passes it off to the downstream. These steps will be performed over and over until the last object has been handled.

Because a plugin doesn't have to wait for the upstream to complete all its work before getting started, the process is very efficient. And by keeping the intermediate files in memory, it avoids the unnecessary disk I/O and further improves the performance.

A Gulp plugin is in fact a Node.js object transform stream in another name. The [core API](https://nodejs.org/api/stream.html#stream_stream) provides interfaces for defining different kinds of streams. In the case of a Gulp plugin, we should subclassing the [Transform](https://nodejs.org/api/stream.html#stream_class_stream_transform) class with the [`util.inherits`](https://nodejs.org/api/util.html#util_util_inherits_constructor_superconstructor) method, and set the [`objectMode`](https://nodejs.org/api/stream.html#stream_object_mode) in the child class constructor:

{% highlight javascript %}
var util = require('util');
var Transform = require('stream').Transform;

util.inherits(PassThrough, Transform);

function PassThrough() {
  Transform.call(this, { objectMode: true });
}

PassThrough.prototype._transform = function(file, _, next) {
  // do something with the file object
  this.push(file);
  next();
};

PassThrough.prototype._flush = function(done) {
  // do something for cleanup
  done();
}
{% endhighlight %}

This is the simplest form of a Node.js object transform stream. It just passes objects through from upstream to downstream as-is. When inheriting, there are [two methods](https://nodejs.org/api/stream.html#stream_api_for_stream_implementors) in the interface we should override in the subclass. The [`_transform`](https://nodejs.org/api/stream.html#stream_transform_transform_chunk_encoding_callback) method is where we specify the operation performed on each file. We simply use the internal [`this.push()`](https://nodejs.org/api/stream.html#stream_readable_push_chunk_encoding) method here to hand the object over to the downstream. The optional [`_flush`](https://nodejs.org/api/stream.html#stream_transform_flush_callback) method is where we add some cleanup code that will be executed right before the stream ends.

This passthrough stream can be used in a gulpfile as with any other Gulp plugin, as shown in the following code:

{% highlight javascript %}
var passThrough = new PassThrough();

gulp.task('default', function() {
  return gulp.src('src/**/*')
    .pipe(passThrough)
    .pipe(gulp.dest('dist'));
});
{% endhighlight %}


Though the idea of a stream is not particularly difficult, the implementation *is*. We have to handle the class inheritance carefully and add option flags appropriately. In practice, however, we almost never use the core stream API directly. There is a handy module called [`through2`](https://www.npmjs.com/package/through2) that will save us from this awkwardness:

{% highlight javascript %}
var through2 = require('through2');

var passThrough = through2.obj(transform, flush);

function transform(file, _, next) {
  // do something with the file object
  this.push(file);
  next();
}

function flush(done) {
  // do something for cleanup
  done();
}
{% endhighlight %}

The functionality remains the same, but with the help of `through2`, our implementation is simpler and intent is clearer.
