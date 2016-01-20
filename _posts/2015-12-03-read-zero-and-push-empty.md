---
layout: post
title:  "read(0) and push('')"
date:   2015-12-03 12:00:00 +0800
---

In the [API docs](https://nodejs.org/api/stream.html) of Node.js streams, `read(0)` and `push('')` are used a couple times in the examples. I found them obscure in my first read, even for the second.

`read()` is a method belongs to Readable streams. It should be called by a consumer to read more bytes from a stream. `read(0)` simply means read zero bytes from the stream.

If it’s for nothing, why would we even try to read them? Well, from the consumer’s point of view, it gets nothing. But it’s still a `read()` call, which will trigger some internal mechanism of a Readable stream.

A Readable stream maintains a read buffer. It’s size is decided by the `highWaterMark` option when initializing the stream object. When a consumer requests more bytes by calling `read()`, the stream would first fill up the read buffer with the low-level `_read()` call and then pass the requested bytes over.

This happens even the consumer requests for zero bytes of data.

Now we can understand the usage of `read(0)` in the following snippet from the API docs:

```js
source.on('readable', function() {
  self.read(0);
});
```

`self` is a Readable stream and `source` is the underlying system. Whenever data is readable in the underlying system, it would call `read(0)` to fill up the internal buffer.

Let’s turn to `push('')`.

When implementing a Readable stream, we must define a `_read()` method for fetching data from the underlying system. And in the `_read()` definition, we must use the `push()` method to add the fetched data to the read buffer.

Here’s an example of a `_read()` method:

```js
MyReadable.prototype._read = function(size) {
  var data = fetchDataSomehow(size);
  if (data) this.push(data);
};
```

A `read()` call would trigger a `_read()` call. In turn, a `_read()` call would trigger a `push()` call. We say `read()` and `push()` must appear in pairs.

In some cases, even if the `read()` call reads nothing, we still need to use `push()` to end the read-push cycle.

One case is when `unshift()` is used to “un-consume” some data, as the following snippet from the API docs shows:

```js
this.unshift(b);
this.push('');
```

Like `push()`, `unshift()` add some data to the read buffer. But it can not replace the role of `push()` for ending the reading process. To quote the API docs:

> Note that, unlike stream.push(chunk), stream.unshift(chunk) will not end the reading process by resetting the internal reading state of the stream.

That’s why `push('')` should be used immediately after the `unshift()` call.

Another case of using `push('')` should be obvious. It’s when we get nothing from the underlying system. Perhaps the underlying system is not ready yet or only zero bytes are requested.

The API docs also give an example of this case:

```js
var chunk = this._source.read();
if (chunk === null)
  return this.push('');
```

It should be all meaningful by now.
