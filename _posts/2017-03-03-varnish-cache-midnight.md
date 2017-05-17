---
layout: post
title: Burning the Midnight Oil (with Web Caching)
date: {2017-03-03}
summary: Setting a Varnish cache to expire at midnight
categories: Varnish midnight cache expiration
published: true
---
## Varnish - Set Cache to Expire at Midnight


Configuring Varnish can be a tricky thing. I learned just how tricky when I needed to set my cached objects to expire at midnight each night. Much of the learning curve here came from having to learn the Varnish Configuration Language (VCL). The [Varnish Standard Module documentation](https://varnish-cache.org/docs/trunk/reference/vmod_std.generated.html) came in handy here, as I had to use built-in functions from std to do a bit of type juggling while I made my calcuations.

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
