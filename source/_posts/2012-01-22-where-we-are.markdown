---
layout: post
title: "Where we are"
date: 2012-01-23 15:41
comments: true
author: Darren
categories: 
---
A mini [state of the union][1], if you like. I thought it would be worthwhile to outline where we are at the moment, and touch a little on whats coming up.

<!--more-->

iWeb has been in business since 1995 designing and developing websites - mainly in PHP - and over time the hosting side of business has grown from a few sites running on a handful of servers into a multi datacenter operation, spanning hundreds of virtual servers.

Managing an infrastructure that large by hand isn't particularly feasible, so we have been using [Puppet][2] for our configuration management, driven by our custom built CMDB/inventory system, which is a [Django][3] web app. It's about 18 months old now, and it's been instrumental in allowing us to scale our infrastructure - we've moved away from shared hosting (one of the motivators to increase our automation), and deployed a lot of new machines, all managed by puppet. We've learned a lot, and are planning a 'version 2' of the web app this year, which I'm sure we'll be blogging about.

In terms of monitoring, we've experimented a lot since I started here. It's a [hot topic][4] everywhere at the moment, and iWeb is just the same. Early on we used [Munin][5], which we had trouble scaling - no so much with the number of hosts, but with the data resolution we were looking for. We switched to [collectd][6] and its been great - extremely lightweight, even with a 10 second resolution. It has a number of [generic plugins][7], which are fantastic for collecting all sorts of application specific metrics (like [exim stats][8], for example). While its proved great at collecting metrics, collected doesn't provide any UI - instead it ships with a number of [write plugins][9] which can be used to persist data (or not, as we'll soon see…). Having used Munin we were used to dealing with RRD's, so we ran with those, and cooked up a simple [Flask][10] based [web interface][11] to display them (though we actually used [collectd-web][12] to render the graphs). For a time we used it's notification system to power our Icinga alerts, but we stopped that in the end - our alerting is a whole other blog post :)

Lately we've actually been sending all our collectd metrics into [Graphite][13] - which is a fantastic tool for combining and visualising metrics. We're also able to collect additional metrics not suited to collectd (the data isn't available for collection every n seconds, or it's expensive to collect that data every n seconds). We may even turn off our RRD generation completely in the near future, Graphite has given us many new ways to visualise what's happening (or what has happened) on our infrastructure. Because we in operations largely know the software stacks running on our customers machines, we are able to combine application metrics with system metrics in a way we haven't been able to before. I suspect this will lead to blog posts :)

Our alerting is one of the areas thats least changed in my time here, at least in terms of the technology. Alert states are maintained by [Icinga][14], and we have written an app to aggregate that data, display interesting state changes and if necessary alert the on call personnel that something is broken. This is an area we have targeted for improvement this year - there's data locked up in here that can be useful to customers, their project managers and developers. It will be fun coming up with new ways to expose it.

iWeb has a popular [ftp service][15] which has been running for a few years now. It's due a bit of a refresh - not just from a design point of view, but the current system has grown to the point where is poses a number of engineering challenges, particularly around data storage. We've been working on the new service since just before Christmas and will be ready to launch soon - it's particularly exciting for us in operations as - owing to these engineering challenges - we are developing the whole stack ourselves, and have designed the components parts to be the platform upon which we will launch a number of new hosted services this year. Not only is the technology interesting here, but the process as well - we've built a continuous deployment workflow so that we can release features and fixes as fast as possible. As Aaron mentioned, we'll be blogging about this a bit more in the near future.

So, thats a little bit about where we are, and where we're heading at the moment.

[1]: http://en.wikipedia.org/wiki/State_of_the_Union_address
[2]: http://puppetlabs.com/
[3]: https://www.djangoproject.com/
[4]: https://twitter.com/#!/search/%23monitoringsucks
[5]: http://munin-monitoring.org/
[6]: http://collectd.org/
[7]: http://collectd.org/wiki/index.php/Category:Generic_Plugins
[8]: http://collectd.org/wiki/index.php/Plugin:Tail/Config#Exim
[9]: http://collectd.org/wiki/index.php/Category:Callback_write
[10]: http://flask.pocoo.org/
[11]: https://github.com/iwebhosting/collectd-flask
[12]: http://kenny.belitzky.com/projects/collectd-web
[13]: http://graphite.wikidot.com/
[14]: https://www.icinga.org/
[15]: http://www.iweb-ftp.co.uk/