---
title: Burning the Midnight Oil (with Web Caching)
date: 2017-03-03 00:00:00 Z
categories:
- Varnish
- midnight
- cache
- expiration
layout: post
summary: Setting a Varnish cache to expire at midnight
---

## Varnish - Set Cache to Expire at Midnight


I recently delved into the world of web caches. In case you're not familiar, caching is the ability to save a part of website for quick and easy retrieval. Often developers will cache parts of sites that are slow and have fairly static content. Our digital collections site had for many years been quick and nimble; however, over the past year, it had started to show its age and become increasingly pedestrian. In order to remedy this, we first delved into our site's code to fix any inefficiencies that might be holding things up. But, after doing all that work, the site, though much improved, needed a bit more pep. We turned to web caching as a solution. Luckily, we weren't completely unfamiliar with the concept, as we had used it in part of our infrastructure already; however, now, we had proposed to apply it to our entire website--something we had never done before.

We chose Varnish, a modern and popular web caching software. This software stands in front of our website, kind of like a guard in front of a palace. If someone asks for something (a web page, an image, a piece of metadata), it checks to see if it has a copy of it already available. If it does, then it gives it back to the user immediately; if not, it asks the repository for the content--which is where the site can be slow...waiting for the content to be retrieved from the repository. As I mentioned previously, many people use web caching when their site is mostly static content; ours, however, is a bit more dynamic. New content is being added or updated fairly regularly. Therefore, if we want to realize any of the speed gains from web caching, we had to find a smart balance between serving content quickly and serving the most up-to-date results. For the most part, this is not a difficult thing to accomplish, but the tricky part comes in our search results page. This page is the most heavily used part of the site, and one that needs the most current data. Our solution, therefore, was to have the cache reset at midnight each night. The difficult part is in the implementation. Every time someone visits a new search results page, we have to activate a setting (known as a header) that lists the amount of seconds remaining until midnight. This header is what will ensure the cache works properly. What follows is the code (and a fairly in the weeds explanation) of how to accomplish this. My hope is anyone else faced with this challenge will find the information below useful.

Configuring Varnish can be a tricky thing. Much of the learning curve here came from having to learn the Varnish Configuration Language (VCL). The [Varnish Standard Module documentation](https://varnish-cache.org/docs/trunk/reference/vmod_std.generated.html) came in handy here, as I had to use built-in functions from std to do a bit of type juggling while I made my calcuations.

You'll note that I used HTTP headers as a form of variable in this case, and that because neither standard VCL nor the vmod_std module supported the used of the modulus operator, my calculations are a bit long. In case you need to perform a similar calcuation, I've included the code below. 

{% highlight ruby lineanchors %}

#Make sure to import the std module at the top of your .vcl file. Like so:
import std;


sub vcl_backend_response {

  # Set expiration date to expire at midnight
  # First, calculate the amount of seconds that have occurred today; there are 86400s in a day
  # Normally, this is the amount of seconds since Linux epoch % number of seconds in a day;
  # however, vcl doesn't support the modulus operator (which would give us the remainder), so
  # here's the long-hand version

  set beresp.http.exp = std.integer(std.time2integer(now, 0) / 86400, 0);
  set beresp.http.exp = std.integer(std.integer(beresp.http.exp, 0) * 86400, 0);
  set beresp.http.exp = std.integer(std.time2integer(now, 0) - std.integer(beresp.http.exp, 0), 0);

  # Now we need to calculate the amount of seconds we have left before midnight
  # Subtract the amount of seconds in a day from the amount that we've already gone through in order to get the amount of seconds remaining
  # Also, make that final number into a string because that's how ttl will need it

  set beresp.http.exp = 86400 - std.integer(beresp.http.exp, 0) + "s";
  set beresp.ttl = std.duration(beresp.http.exp, 1d);

  # We're going to let Varnish respond with its expiration date (aka Expires Header)
  unset beresp.http.expires;

  # Now let's reset the Expires header based upon our current ttl
  set beresp.http.Expires = "" + (now + beresp.ttl);
}

{% endhighlight %}
