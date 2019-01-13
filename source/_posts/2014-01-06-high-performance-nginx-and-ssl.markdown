---
layout: post
title: "High performance Nginx and SSL"
date: 2014-01-06 17:51:19 +1300
comments: true
categories: 
---

As we move towards launch, we're doing a lot of long delayed polish and testing.  Checking browser compatability, adding cacheing, integrating with a CDN, and generally making sure everything is working just right.  We're also doing some stress tests with the excellent [blitz.io](http://blitz.io/) tool.

## About that…

So, I queued up a basic blitz.io test (or a "rush", in their terminology), and…

{% img /images/blog/whups.png 350 350 'whups' 'whups' %}

For those unfamiliar, the grey line shows increasing load, the blue line shows replies, the red line shows errors.  Or in simple terms, about the time I got 40 hits a second, everything fell over.

On the surface, this is kind of surprising, since I'm using Nginx, and serving up a static pre-rendered page.  Nginx is well known to be fast and really good at serving static content; most of the blog posts and StackOverflow comments about how to fix "my site falls over when I stress test it" boil down to "use Nginx!".  But I'm already using it, so…um.

## A few tweaks

First off, I poked around a bit, and realised that I actually wasn't serving up the page directly from disk; a small config error meant I was actually hitting a node.js process which was serving up the static page directly from disk.  Node isn't quite as fast as Nginx, but it's quite fast enough; fixing my config didn't do anything.

So then I started tweaking Nginx config variables.  `worker_processes` didn't seem to do much.  And neither did `worker_rlimit_nofile`, nor `open_file_cache`, or turning `access_log` to `off`, or well…anything.  My server froze up and cried little tears of pain every time I hit the test.

## Progress

Well, okay.  So something is bottlenecked.  And it's not clear what exactly is bottlenecked either, so…hm.  Let's try running `top` on the server while we hit it with a test.

Oh look, my CPU usage is 100%.  That certainly explains the results I'm seeing, but Nginx is meant to be super low CPU when it comes to serving static files!  What's going on?

{% pullquote %}{" At this point, most of your are probably yelling at your monitors:"It says SSL in your title!  SSL termination is computationally expensive!  You're a flaming idiot!" "}{% endpullquote %}
	
Indeed.  It does, it is, and I might be.  Because once I realised what was going wrong I ran an rush on the non-SSL URL for our app.  And you can probably guess the results:
<div class="clearfix"></div>

{% img /images/blog/hmm.png 350 350 'hmm' 'hmm' %}

What you're seeing here is that the response time stayed pretty constant, while hits tracked overall volume closely.  In short, it responded (quickly!) to whatever I threw at it up to 250 hits/second which is as high as I decided to go.  Which is what Nginx is famed for, so uh…good.  (Not shown:  Our CPU usage, which was essentially nil.  No surprise; serving static files from RAM over plain-old-HTTP is practically free.)

But now what?  I kind need SSL termination.

## The fix

### Hardware

Step one is to look at the hardware I'm throwing at the problem.  Which, as it turns out, is a $10/month Digital Ocean droplet.  For the price, they're not bad — 30GB of SSD disk, 1GB of RAM, and 1 vCPU.  It's that last bit which is clearly killing us now though.

Well, it's the work of a moment to resize our droplet to the next tier up with 2 vCPU.  Let's reset to vanilla Nginx settings and see what that does:

{% img /images/blog/better.png 350 350 'better' 'better' %}

Well, that is technically better…kind of.  Okay, now with all those tuned `worker_processes` settings we copied off the internet?

{% img /images/blog/nope.png 350 350 'nope' 'nope]' %}

Well, it's certainly spiky looking.  But not what we're going for.

### Software

Nginx is very configurable.  Maybe there are some specific SSL settings?  But of course there are!

```
ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
ssl_ciphers ECDHE-RSA-AES256-SHA384:AES256-SHA256:RC4:HIGH:!MD5:!aNULL:!eNULL:!NULL:!DH:!EDH:!AESGCM;
ssl_prefer_server_ciphers on;
ssl_session_cache shared:SSL:10m;
ssl_session_timeout 10m;
```

{% img /images/blog/success.png 350 350 'yes!' 'yes!]' %}

Well how about that?

## The conclusion

Well, the obvious conclusion is "know how to use your tools", but that's a bit too general.  Let's look for something a bit more specific.  Such as:

Make sure to configure Nginx properly for SSL!