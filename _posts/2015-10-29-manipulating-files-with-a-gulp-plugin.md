---
layout: post
title:  "Manipulating files with a Gulp plugin"
date:   2015-10-29 12:00:00
---

A Gulp plugin is just a Node.js module that happens to return a stream.
With [`through2`](https://www.npmjs.com/package/through2), a thin
wrapper of the core [stream API](https://nodejs.org/api/stream.html),
creating a Gulp plugin can be as simple as this:

{% highlight javascript %}
var through = require('through2');

module.exports = function() {
  return through.obj(function(file, _, next) {
    next(null, file);
  )};
};
{% endhighlight %}

The [`through.obj()`](https://github.com/rvagg/through2) method
generates a [transform
stream](https://nodejs.org/api/stream.html#stream_class_stream_transform)
in [object mode](https://nodejs.org/api/stream.html#stream_object_mode).
Which means it would take an object from the upstream, do some
processing, and then push an object off downstream.

In a Gulp task, the objects being passed to a plugin is usually of the
[Vinyl](https://www.npmjs.com/package/vinyl) class. A Vinyl object is a
JavaScript representation of a file. It defines methods and properties
for manipulating the file it represents. For instance, we would use
[`file.contents`](https://github.com/gulpjs/vinyl#contents) for changing
the contents of a file.

The Gulp documentation gives an
[example](https://github.com/gulpjs/gulp/blob/a2715207edb284b44921537e4d5901e2be1a482c/docs/writing-a-plugin/guidelines.md#what-does-a-good-plugin-look-like)
to illustrate how we can manipulate a file via a Vinyl object. The need
is to add some text to the beginning of all files and it can be
accomplished by concatenating the user-provided text and the original
contents of the file.

By default, `file.contents` is a
[Buffer](https://nodejs.org/api/buffer.html) object. To concatenate
buffers, we could use the
[`Buffer.concat`](https://nodejs.org/api/buffer.html#buffer_class_method_buffer_concat_list_totallength)
method:

{% highlight javascript %}
var prefixText = new Buffer('YOUR TEXT');

file.contents = Buffer.concat([prefixText, file.contents]);
{% endhighlight %}

We save the result back to the `file.contents` property, so the contents
of the underlying file will be changed accordingly.

Putting this idea to the skeleton code above, we now have a Gulp plugin
that does some real work:

{% highlight javascript %}
var through = require('through2');

function gulpPrefixer(prefixText) {
  prefixText = new Buffer(prefixText);

  return through.obj(function(file, _, next) {
    file.contents = Buffer.concat([prefixText, file.contents]);
  });

  cb(null, file);
}

module.exports = gulpPrefixer;
{% endhighlight %}

Save the module to file, and we can use it as would any other Gulp
plugin:

{% highlight javascript %}
var prefixer = require('./gulp-prefixer');

gulp.task('default', function() {
  return gulp.src('src/**/*')
    .pipe(prefixer('YOUR TEXT'))
    .pipe(gulp.dest('dist'));
});
{% endhighlight %}

The `file.contents` was populated with the invocation of the `gulp.src`
method. If we set the `buffer` flag to `false`, the value
`file.contents` would be stream:

{% highlight javascript %}
var prefixer = require('./gulp-prefixer');

gulp.task('default', function() {
  return gulp.src('src/**/*', { buffer: false })
    .pipe(prefixer('YOUR TEXT'))
    .pipe(gulp.dest('dist'));
});
{% endhighlight %}

For this case, concatenating the user text and the original contents
means piping them to a stream sequentially. We could use `through2` to
create a transform stream as the target:

{% highlight javascript %}
var stream = through();
var prefixText = new Buffer('YOUR TEXT');

stream.write(prefixText);
file.contents = file.contents.pipe(stream);
{% endhighlight %}

Without arguments, the `through` method just returns a passthrough
stream, which takes an input and push it off as-is.  Note here we use
the `through` method rather than `through.obj`, because we're dealing
with the internal contents of a stream, which are buffers, not objects.

The `stream.write` method is used for manually pushing a buffer. And the
`stream.pipe` method is used to chain together two streams.

With this in place, we can write the plugin as follows:

{% highlight javascript %}
var through = require('through2');

function prefixStream(prefixText) {
  var stream = through();
  stream.write(prefixText);
  return stream;
}

function gulpPrefixer(prefixText) {
  prefixText = new Buffer(prefixText);

  return through.obj(function(file, _, next) {
    file.contents = file.contents.pipe(prefixStream(prefixText));
  });

  cb(null, file);
}

module.exports = gulpPrefixer;
{% endhighlight %}

A real Gulp plugin should handle both cases, and should also consider
other requirements such as error handling. Refer to the [official
documentation](https://github.com/gulpjs/gulp/blob/a2715207edb284b44921537e4d5901e2be1a482c/docs/writing-a-plugin/guidelines.md#what-does-a-good-plugin-look-like)
for a compele code listing.
