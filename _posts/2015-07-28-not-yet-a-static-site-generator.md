---
layout: post
title:  "Not yet a static site generator"
date:   2015-07-28 12:00:00 +0800
---

I am going to create a static site generator to power this blog.

Starting from scratch is a good thing. It gives me a chance to imagine what it should be.

My requirement is simple. I want it convenient to use and fast to run.

Once started, the server should never stop even if I messed up a config file. The browser would reload automatically while I'm editing a blog post. When done, publishing a new post should be one command away.

To reduce IO, the same file should never be read from the hard disk more than once. If a file has changed, only the related files, rather than the whole site, would be regenerated.

The blogging software would model Jekyll but written in Node.js.

I call it [Blogware](https://github.com/blogware/blogware).
