---
layout: post
title: "Lunchtime Project: Apache Space Core Module"
date: 2012-02-08 21:08
comments: true
author: Aaron Brady
categories: C Apache Lunchtime
---

Show and tell time!

[Darren][daz] mentioned in passing how we should return quotes in the HTTP
responses for our new product. [Slashdot already does this][futurama] with
Futurama quotes, and it didn't take long to add support for the main WSGI
application, but in the interests of security the public-facing website doesn't
run any dynamic languages; no Python, no PHP.

<!--more-->

I investigated using [mod\_headers][1] and the [RewriteMap][] functionality of
[mod\_rewrite][2], but even if that *would* work the random selection built
into RewriteMap requires that all of the quotes be on one line. Unfortunately,
in our case [the source material][space] is 88 lines long.

So, on a box with just Apache, there's just one reasonable choice: learn how to
write Apache modules. [mod\_rpaf][rpaf] is just about the simplest module I
know of. It tweaks the inbound headers with code like this:

{% codeblock "Add X-Forwarded-For" lang:c %}
if (is_in_array(r->connection->remote_ip, cfg->proxy_ips) == 1) {
  if (fwdvalue = apr_table_get(r->headers_in, "X-Forwarded-For")) {
    apr_array_header_t *arr = apr_array_make(r->pool, 0, sizeof(char*));
{% endcodeblock %}

Building and installing a standalone module is incredibly simple if you have
the apache2.2-dev package installed (or equivalent for your distribution):

    apxs2 -i -a -c *.c
    echo "LoadModule rpaf_module /usr/lib/apache2/modules/mod_rpaf.so" >> /etc/apache2/httpd.conf

Lets look at the major parts:

{% codeblock "Boilerplate Config" lang:c %}
module AP_MODULE_DECLARE_DATA rpaf_module;

typedef struct {
    int enable;
    int sethostname;
    apr_array_header_t *proxy_ips;
} rpaf_server_cfg;

static void *rpaf_create_server_cfg(apr_pool_t *p, server_rec *s) {
    rpaf_server_cfg *cfg = (rpaf_server_cfg *)apr_pcalloc(p, sizeof(rpaf_server_cfg));
    if (!cfg)
        return NULL;

    cfg->proxy_ips = apr_array_make(p, 0, sizeof(char *));
    cfg->enable = 0;
    cfg->sethostname = 0;

    return (void *)cfg;
}
{% endcodeblock %}

Seems reasonable, we define a one-off struct, allocate it in the server config
callback and set some defaults. The RPAF code uses and apr\_array, but they're
not the easiest thing to iterate over, according to some [APR tutorial][] that
I found. However, a [ring][] looks ideal: we can loop over it any random amount
and be confident that we won't walk off of the end, so we don't even need to
keep a count of how many items are in it. (Though, we do)

{% codeblock "Initialisation, Callbacks" lang:c %}
static const command_rec rpaf_cmds[] = {
    // SNIP - Aaron
    AP_INIT_ITERATE(
                    "RPAFproxy_ips",
                    rpaf_set_proxy_ip,
                    NULL,
                    RSRC_CONF,
                    "IP(s) of Proxy server setting X-Forwarded-For header"
                    ),
    { NULL }
};

static void register_hooks(apr_pool_t *p) {
    ap_hook_post_read_request(change_remote_ip, NULL, NULL, APR_HOOK_MIDDLE);
}

module AP_MODULE_DECLARE_DATA rpaf_module = {
    STANDARD20_MODULE_STUFF,
    NULL,
    NULL,
    rpaf_create_server_cfg,
    NULL,
    rpaf_cmds,
    register_hooks,
};
{% endcodeblock %}

Okay, everything's still reasonable with this, we're providing an array of
configuration directives and respective callbacks, a function that gets called
to register some hooks, and a struct that gets read when the module is
`dlopen()`ed pointing to these things.

Our versions of these:

{% codeblock "Ring structure and Config structure" lang:c %}
typedef struct _quote_t {
    APR_RING_ENTRY(_quote_t) link;
    const char *quote;
} quote_t;

typedef struct _quote_ring_t quote_ring_t;
APR_RING_HEAD(_quote_ring_t, _quote_t);

module AP_MODULE_DECLARE_DATA space_core_module;

typedef struct {
    quote_ring_t *ring;
    int count;
} space_core_server_cfg;

static void *space_core_create_server_cfg(apr_pool_t *p, server_rec *s) {
    space_core_server_cfg *cfg = (space_core_server_cfg *)apr_pcalloc(p, sizeof(space_core_server_cfg));

    if (!cfg)
        return NULL;

    cfg->ring = apr_palloc(p, sizeof(quote_ring_t));
    cfg->count = 0;
    APR_RING_INIT(cfg->ring, _quote_t, link);

    return (void *)cfg;
}
{% endcodeblock %}

`quote_ring_t` will be the container for a bunch of `quote_t`s which themselves
include a pointer to their next link and a string (for the quote).
`space_core_server_cfg` will be just a count of the number of quotes and the
ring structure itself. Suitable macro hackery is going on under `APR_RING_INIT`
- I don't even *want* to know what.

{% codeblock "Configuration and Initialisation" lang:c %}
static const command_rec space_core_cmds[] = {
    AP_INIT_TAKE1(
                    "SpaceCoreFile",
                    space_core_file,
                    NULL,
                    RSRC_CONF,
                    "Location of Quotes File"
                    ),
    { NULL }
};

static void register_hooks(apr_pool_t *p) {
    ap_hook_fixups(ap_headers_fixup, NULL, NULL, APR_HOOK_LAST);
}

module AP_MODULE_DECLARE_DATA space_core_module = {
    STANDARD20_MODULE_STUFF,
    NULL,
    NULL,
    space_core_create_server_cfg,
    NULL,
    space_core_cmds,
    register_hooks,
};
{% endcodeblock %}

This is a fairly straight swap: lots of things and renamed to `space_core`, the
hook we register is `ap_hook_fixups`, a suitable place to manipulate the
response. The possible hooks are documented [in this guide][hooks]. We defined
just one parameter `SpaceCoreFile` which should be the absolute filename of the
quotes file.

We're using `AP_INIT_TAKE1` instead of `AP_INIT_ITERATE`; this is because
mod\_rpaf takes a list of IPs, but we really just want one filename. This is
expanded on [in this Apache tutorial][pink] (wow, remember webpages like this?).

## The good stuff!

Most of this has been fairly boilerplate, this is where the (admittedly small)
magic happens:

{% codeblock "Load the File in" lang:c %}
static const char *space_core_file(cmd_parms *cmd, void *dummy, char *filename) {
    server_rec *s = cmd->server;
    space_core_server_cfg *cfg = (space_core_server_cfg *)ap_get_module_config(s->module_config, &space_core_module);
    char buf[1024] = {0};

    apr_file_t *fd;
    apr_file_open(&fd, filename, APR_READ, APR_OS_DEFAULT, cmd->pool);
    while(APR_EOF != apr_file_gets(buf, 1024, fd)) {
        if(strlen(buf) > 1) {
            buf[strlen(buf)-1] = '\0';
            quote_t *elem = apr_palloc(cmd->pool, sizeof(quote_t));
            elem->quote = apr_pstrdup(cmd->pool, buf);
            APR_RING_INSERT_TAIL(cfg->ring, elem, _quote_t, link);
            cfg->count++;
        }
    }   

    return NULL;
}
{% endcodeblock %}

We'll create a staticly allocated buffer, filled with NULs. We'll trust that
the filename is good, because I don't check any return values. `apr_file_gets`
is a portable version of `fgets` which respects maxium lengths. 

If the line has more than one character in it (including the newline, so
essentially: if it's not blank) then we strip off the trailing newline,
duplicate the string and add it to a fresh `quote_t` structure.

`APR_RING_INSERT_TAIL` does some more magic macro work to expand the ring with
the new element, and we increment the quote count.

Pulling a quote out should be a simple case of picking a random number,
advancing through the ring a few times and then setting it in the response:

{% codeblock "Set the Header" lang:c %}
static int ap_headers_fixup(request_rec *r) {
    space_core_server_cfg *cfg = (space_core_server_cfg *)ap_get_module_config(r->server->module_config, &space_core_module);
    int rand = random() % cfg->count;
    quote_t* elem;

    for(elem = APR_RING_FIRST(cfg->ring); elem && rand; elem = APR_RING_NEXT(elem, link))
        rand--;

    if(elem)
        apr_table_set(r->headers_out, "X-Space-Core", elem->quote);

    return DECLINED;
}
{% endcodeblock %}

And that's it! There's a [gist with the finished code][gist] with the whole
file, and I've packaged it up (for fun) [as a .deb for Ubuntu Lucid][deb] - just

    apt-get install libapache2-mod-space-core
    a2enmod space_core
    /etc/init.d/apache2 reload

and you're done!

    $ curl -I http://localhost/
    HTTP/1.1 200 OK
    Date: Wed, 08 Feb 2012 22:17:04 GMT
    Server: Apache/2.2.14 (Ubuntu)
    X-Space-Core: What’s your favorite thing about space? Mine is space.
    Last-Modified: Tue, 07 Feb 2012 22:18:12 GMT
    ETag: "124bde-b1-4b867279581c1"

This could easily be rounded out as a generate quotes module with a few more
configuration directives: if you've found this useful, please let me know
on Twitter via [@insom][]

![Nyan Core](http://i0.kym-cdn.com/photos/images/original/000/118/802/halolz-dot-com-portal2-nyancatspacecore.gif)


[daz]: http://twitter.com/DazWorrall
[futurama]: http://keithdevens.com/weblog/archive/2002/Jul/20/FuturamaQuotesHTTPHeaders
[1]: http://httpd.apache.org/docs/current/mod/mod_headers.html
[2]: http://httpd.apache.org/docs/current/mod/mod_rewrite.html
[RewriteMap]: http://httpd.apache.org/docs/current/mod/mod_rewrite.html#rewritemap
[space]: http://theportalwiki.com/wiki/Core_voice_lines#Space_core_lines
[rpaf]: http://stderr.net/apache/rpaf/
[APR tutorial]: http://dev.ariel-networks.com/apr/apr-tutorial/html/apr-tutorial-19.html
[ring]: http://dev.ariel-networks.com/apr/apr-tutorial/html/apr-tutorial-19.html#ss19.4
[hooks]: http://httpd.apache.org/docs/2.0/developer/modules.html#messy
[pink]: http://www.apachetutor.org/dev/config
[gist]: https://gist.github.com/1774452
[deb]: https://launchpad.net/~bradya/+archive/space-core/ 
[@insom]: http://twitter.com/insom
