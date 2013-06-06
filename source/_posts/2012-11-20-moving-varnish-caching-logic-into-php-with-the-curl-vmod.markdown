---
layout: post
title: "Moving Varnish caching logic into PHP with the cURL VMOD"
date: 2012-11-20 14:46
comments: true
categories: PHP Varnish Speed
author: Aaron Brady
---
As part of our general quest to provide [fast Magento hosting][faster], we've
spent some time exploring new features in [Varnish 3.0][v3]. VMOD support means that there's a supported way of extending VCL with modules in their own name
spaces. Some of the modules from [the VMOD list][vmod list] looked really
interesting; particularly [Redis][vredis], [Memcached][vmemcached] and
[cURL][vcurl].

In this post I'm going to use the [cURL][vcurl] VMOD to decide whether to show
a page from the cache or not, by deciding if the user is logged in. This is the
kind of thing you *could* do with VCL if you were happy to set custom cookies
alongside your apps own session cookie, but as you want to make more complex
decisions I thought it would be interesting to push this into PHP.

<!--more-->

#### Building the cURL VMOD

On Ubuntu Precise, install the dependencies:

    # apt-get install varnish php5 apache2 libvarnishapi-dev varnish-dbg \
    build-essential libcurl4-openssl-dev git automake libtool autoconf \
    libpcre3-dev

Grab the varnish source, this is required to build the VMOD. We also have to
build (but not install) it:

    $ apt-get source varnish
    $ cd varnish*
    $ ./configure
    $ make

Check the VMOD source code out and generate the configure file:

    $ git clone git://github.com/varnish/libvmod-curl.git
    $ cd libvmod-curl
    $ ./autogen.sh
    $ ./configure VARNISHSRC=$HOME/varnish-3.0.2/
    $ make
    # make install

On my machine it put everything in the right place, and I was able to just put
`import curl;` into my default.vcl and start using it right away.

#### Creating a simple PHP test application

Let's create a small app with a page which is expensive to generate, but can be cached for users who haven't stored any state in their session.

{% codeblock "slow.php" lang:php %}
<?php
session_start();
sleep(5);
print_r($_SESSION);
{% endcodeblock %}

The above file, if we didn't override Varnish, would never be cachable because
the [session_start][ssphp]() call will emit a `Set-Cookie` header, and Varnish
will not normally cache a response which sets cookies.

Anonymous users should always see an empty array, and users with state should
always see a "YES".

{% codeblock "start_session.php" lang:php %}
<?php
session_start();
$_SESSION['started'] = 'YES';
session_write_close();
{% endcodeblock %}

This will set the above "YES". Once this is visited, we expect that we'll see
an uncached version of the site.

As of right now visiting `/slow.php` should take 5 seconds, reloading it will
take 5 seconds (and still show a blank array), then visiting
`/start_session.php` will be fast and finally visiting `/slow.php` will take
5 seconds, and show "YES". This is because, right now, nothing is cached.

#### Just enough VCL to let PHP make the decision

{% codeblock "default.vcl" lang:c %}
import curl;

backend default {
    .host = "127.0.0.1";
    .port = "80";
}

sub vcl_recv {
        /* The next 5 lines take the cookie line and normalise it to just
         * the session_id */
        
        set req.http.x-cookie = ";" + req.http.cookie;
        set req.http.x-cookie = regsuball(req.http.x-cookie, "; +", ";");
        set req.http.x-cookie = regsuball(req.http.x-cookie, ";(PHPSESSID)=", "; ");
        set req.http.x-cookie = regsuball(req.http.x-cookie, ";[^ ][^;]*", "");
        set req.http.x-cookie = regsuball(req.http.x-cookie, "^[; ]+|[; ]+$", "");
        
        curl.fetch("http://localhost/decider.php?c=" + req.http.x-cookie);
        
        remove req.http.x-cookie;
        set req.http.x-session = curl.header("X-Started");

        if (req.http.x-session != "yes") {
                remove req.http.cookie;
        }
}

sub vcl_hit {
        if(req.http.x-session == "yes") {
                return (pass);
        }
}

sub vcl_fetch {
        if(req.http.x-session != "yes" && req.url !~ "add.php") {
                remove beresp.http.set-cookie;
                set beresp.ttl = 5m;
        }
}
{% endcodeblock %}

The above temporarily adds an `X-Cookie` header which will just contain the PHP
session ID. We're going to pass this to our `decider.php` script, and it will
pass back an `X-Started` header saying whether we consider this session to be
"properly" started. We do all of this in the `vcl_recv` step, immediately as
the request comes in to Varnish.

`vcl_hit` will check if we consider the session to be started and if so will
issue a pass from now on; this is the code that stops cached pages being
served to people with "proper" sessions.

`vcl_fetch` will now look at the response and, if `X-Session` is not "yes",
will strip off the `Set-Cookie` and set the ttl, allowing it to be cached.

The only unpleasant part in all of this is where we check if the URL is
`add.php`, on line 35. We have to allow the cookie through from this
script so it can start our session.

### Our example 'decider'

Finally, the contents of our example decider file:

{% codeblock "decider.php" lang:php %}
<?php
session_start();
$sessionid = $_GET['c'];
/* The next line is horribly Ubuntu-specific and should just serve as
 * a demonstration. */
$b = file_get_contents('/var/lib/php5/sess_' . $sessionid, 'r');
session_decode($b); /* This loads the session data into $_SESSION */
if($_SESSION['started']) {
	header('X-Started: yes');
}
{% endcodeblock %}

Basically, we will only consider a session to have started fully if the
`started` key in the session exists. The presence of the session cookie
is not enough.

On an e-commerce store you could decide that a session isn't "proper" until
the user is logged in, or has an item in their cart, allowing almost all of
your visitors to continue to see cached versions of the site.

Some large PHP packages (*cough* Magento) are very free with sending cookies
and starting sessions, but the common advice is to override this with
lots of logic in the VCL about what can start a session. Moving this logic
back into PHP seems like a natural step.

The speed of your decider is critical; it's hit for every request. In
production you probably want to have logic above our `vcl_recv` block
which short-circuits it for images and other statics. It will probably
not be acceptible to create, say, a new `Mage` object from PHP in order
to check if the page could be served from cache, given the current session.

&nbsp;

If you've found this useful, please let me know on Twitter via [@insom][]. I
plan on following this post up with a Magento-specific example of how to
build a fast store by bypassing much of Magento's logic on starting sessions.

*That said* a colleague has an alternative pure-PHP solution to Magento's
specific session starting behaviour, but the example will hopefully show how
the above can be applied to other non-trivial web applications.

[faster]: /blog/2012/09/11/diagnosing-magento-speed-issues-with-strace/
[v3]: https://www.varnish-cache.org/releases/varnish-cache-3.0.0
[vmod list]: https://www.varnish-cache.org/vmods
[vredis]: https://github.com/zephirworks/libvmod-redis
[vmemcached]: https://github.com/sodabrew/libvmod-memcached
[vcurl]: https://github.com/varnish/libvmod-curl/
[ssphp]: http://php.net/session_start
[@insom]: http://twitter.com/insom