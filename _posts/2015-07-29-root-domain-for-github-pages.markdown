---
layout: post
title:  "Root domain for GitHub Pages"
date:   2015-07-29 12:00:00
---

The domain given by GitHub Pages for this blog is [gnowoel.github.io](http://gnowoel.github.io/). I'd like to switch to the shorter [gnowoel.com](http://gnowoel.com/). The simplest way to customize the domain is by configuring a CNAME record:

    www CNAME gnowoel.github.io

If we added the record in the DNS settings of [gnowoel.com](http://gnowoel.com/), the site can then be accessed with [www.gnowoel.com](http://www.gnowoel.com/). For a root domain without `www`, we can use an A record instead:

    gnowoel.com A 23.235.47.133

Unlike a CNAME, an A record requires an IP address.

Although it works, binding to a static IP would lose all the benefits of [the GitHub Pages CDN](https://github.com/blog/1715-faster-more-awesome-github-pages), which could've returned the IP of the nearest physical server. To make a root domain work with a CDN, some DNS providers introduced non-standard records such as ALIAS or ANAME. While hesitating to adopt them, I came across a clever solution that solves the problem without breaking the standard.

It's a service from CloudFlare called [CNAME Flattening](https://support.cloudflare.com/hc/en-us/articles/200169056-CNAME-Flattening-RFC-compliant-support-for-CNAME-at-the-root). It looks like a CNAME but actually works as an A record. To set it up, just add a CNAME record for the root domain:

    gnowoel.com CNAME gnowoel.github.io

This notation is not standard but it's only used internally. In practice, CloudFlare would walk through the CNAME chains until it finds an IP and then dynamically generate a standard-compliant A record:

    $ dig gnowoel.com
    gnowoel.com.		30	IN	A	23.235.47.133

Since the DNS query still goes through the subdomain, the IP we get should be the nearest physical server returned by the CDN. In this case, it's from California.

I connected to a VPN and tried again. This time, it's from Sydney.

Brilliant.
