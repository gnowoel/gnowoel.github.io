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
Which means it would take an object from the upstream, and then push
it off downstream.

In a Gulp task, the object being passed through is usually of the
[Vinyl](https://www.npmjs.com/package/vinyl) class. A Vinyl object is a
JavaScript representation of a file. By manipulating a file, we
basically mean invoking a method of the object or changing the value of
a property. For instance, the
[`file.contents`](https://github.com/gulpjs/vinyl#contents) property is
what we need for changing the contents of a file.

The Gulp documentation gives an
[example](https://github.com/gulpjs/gulp/blob/a2715207edb284b44921537e4d5901e2be1a482c/docs/writing-a-plugin/guidelines.md#what-does-a-good-plugin-look-like)
for adding text to the beginning of all files. This functionality can be
accomplished by concatenating the user-provided text and the original
file contents, and then saving the result back to the `file.contents`
property. By default, `file.contents` is a
[Buffer](https://nodejs.org/api/buffer.html) object. To concatenate
buffers, we could use the
[`Buffer.concat`](https://nodejs.org/api/buffer.html#buffer_class_method_buffer_concat_list_totallength)
method:

{% highlight javascript %}
var prefixText = new Buffer('YOUR TEXT');

file.contents = Buffer.concat([prefixtext, file.contents]);
{% endhighlight %}

By incorporating this idea to the skeleton code above, we now have a Gulp
plugin that does some real work:

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

There's one thing we need to be aware. The `file.contents` can also be a
Stream object. It happens when we set the `buffer` flag to `false` in
the invocation of `gulp.src`:

{% highlight javascript %}
var prefixer = require('./gulp-prefixer');

gulp.task('default', function() {
  return gulp.src('src/**/*', { buffer: false })
    .pipe(prefixer('YOUR TEXT'))
    .pipe(gulp.dest('dist'));
});
{% endhighlight %}

For this case, we need to create a new stream in our plugin and use it
as the target for piping text and the original contents to. The new
stream will be saved back as the value of `file.contents`:

{% highlight javascript %}
var stream = through();
var prefixText = new Buffer('YOUR TEXT');

stream.write(prefixText);
file.contents = file.contents.pipe(stream);
{% endhighlight %}

The `stream.write` method is used for manually pushing a buffer to a
stream. To chain together two streams, just use `stream.pipe`.

Now the plugin can be written as follows:

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

A real Gulp plugin would consider both cases, and also other
requirements such as error handling. Refer to the [official
documentation](https://github.com/gulpjs/gulp/blob/a2715207edb284b44921537e4d5901e2be1a482c/docs/writing-a-plugin/guidelines.md#what-does-a-good-plugin-look-like)
for a compele code listing.
