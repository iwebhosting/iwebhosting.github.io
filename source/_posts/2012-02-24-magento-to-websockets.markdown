---
layout: post
title: "Magento to WebSockets, via Redis, Node and Juggernaut"
date: 2012-02-24 23:02
comments: true
categories: Magento Node Redis
author: Aaron Brady
---
I tried to squeeze more hipster web development tech into the title, but we'll
have to just make do with the above.

As our developers are building [Magento][] sites, I though it would be useful
to get to know it better. We know Magento from an operations perspective; it
likes a lot of IOs and CPU, runs well with ephemeral data stored on RAM disk,
needs a tuned APC etc., but I like to understand as much of the stack that
I look after as possible.

<!--more-->

I'm a recovered PHP developer, so this wasn't too hard. Magento's source is
very OO and the PEAR-style autoloader means that files are mostly in obvious
places. We're going to define `Iweb_Redis_Observer` - this will appear in
`app/code/local/Iweb/Redis/Observer.php`. We're also going to define an
`Iweb_Redis` plugin, so the configs will live in
`app/etc/modules/Iweb_Redis.xml` and `app/code/local/Iweb/Redis/etc/config.xml`.

When adding passive hooks into Magento (logging, for example) you can use the
[Event/Observer method documented on the Magento wiki][Events] - various
parts of the core fire events without having a tight binding to what is
receiving them. Classes register their interest in events via XML files. There
are a lot of XML files in Magento.

{% include_code Define a Plugin magento/Iweb_Redis.xml %}

{% include_code Add our Hooks magento/config.xml %}

{% include_code Add an Implementation magento/Observer.php %}

Simple, isn't it? We'll also need [Credis][] put somewhere the autoloader can
find it. Cloning `git://github.com/colinmollenhour/credis.git` into
`app/code/local/Credis` will do it.

Then [install Juggernaut][jn], install Redis (`apt-get install redis-server`)
and visit http://your.server:8080/ for the basic Juggernaut console:

![Juggernaut Console](http://o7.no/whqDZi)

If you've found this useful or have any comments, please let me know on Twitter
via [@insom][]. (If you're looking for [specialists in Magento][iw], I should point
you towards [our parent company][iws]).

[Magento]: http://www.magentocommerce.com/
[Events]: http://www.magentocommerce.com/wiki/5_-_modules_and_development/0_-_module_development_in_magento/customizing_magento_using_event-observer_method
[Credis]: https://github.com/colinmollenhour/credis
[jn]: https://github.com/maccman/juggernaut/blob/master/README.md
[@insom]: http://twitter.com/insom
[iw]: http://www.iweb.co.uk/
[iws]: http://www.iwebsolutions.co.uk/
