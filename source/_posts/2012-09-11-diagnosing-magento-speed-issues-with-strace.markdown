---
layout: post
title: "Diagnosing Magento speed issues with strace"
date: 2012-09-11 10:28
comments: true
categories: Magento
author: Aaron Brady
---
When your application depends on a remote service, and that service is either
down or slow, it will directly affect your own application's performance.

![Slow Website](/assets/magento-slow.png)

There are tools targeted directly at developers, like [Xdebug][] and
[KCachegrind][] but traditional sysadmin tools also have a place and provide a
lot of information that language-level debugging tools omit.

<!--more-->

Because tools like [gdb][1], [time][2] and [strace][3] are designed to work on
single processes, the first thing to do is get your Magento page running as a
separate process, outside of Apache or any other web server. All Magento wants
to know is the HTTP Host header and the URI that has been called. You can pass
these in via the standard [cgi-bin][4] environment variables. Running just
`php` on its own will fire up an interpreter, just like if it was called as a
cgi-bin and similar to how it run a script when called through a web server.

Just change to the top of your Magento tree; typically your `public_html` or
equivalent folder and run a single request:

	$ HTTP_HOST=www.website.com REQUEST_URI=/slow-page.html php index.php

You should get a screenful of generated HTML. We don't care about this too
much, so in future we'll add `>/dev/null` to suppress the output of the script.

The simplest thing we can do is run [time][2] - this will give us at least the
amount of CPU seconds spent in user and system, and the 'wall clock time' of
how long the process took to run. On modern Linux systems it also gives the
percentage CPU used, the peak amount of RAM consumed and the amount of
pagefaults.

	$ HTTP_HOST=www.website.com REQUEST_URI=/slow-page.html time php index.php >/dev/null
	
	0.82user 0.24system 0:01.73elapsed 60%CPU (0avgtext+0avgdata 403520maxresident)k

This one page took 1.7 seconds to render completely, 0.8 of those seconds were
used by PHP using the CPU for something, and 0.24 were spent in system, meaning
that the Linux kernel was doing work on your behalf. The remaining 0.67 seconds
were spent waiting for ... something. In most cases the something is a response
from the network or from the disks. (The response from the network can include
things like MySQL, which is effectively a network connection, even if it's on
the same machine). This request also took 403Mb of RAM to run, including shared
memory.

In an ideal world, each request would spend as close to 100% of its time in
user as possible, and none of it waiting or in system, so let's try and track
down where those missing milliseconds are going. For this, we'll use
[strace][3].

In brief, strace will print out each of the system calls that your process
makes. A system call is where control is passed from user to system, basically
whenever you ask the kernel to do something for you, like open a file, send
data over a socket, or write something to a file descriptor. It also includes a
lot of noisy stuff like getting the time of day, allocating RAM and blocking on
mutexes, so we'll apply a filter to strace to just get the system calls we're
likely to be interested in. You can do this with the `-e` parameter, and full
documentation is in the [man page][3].

	$ HTTP_HOST=www.website.com REQUEST_URI=/slow-page.html strace -e trace=sendto,connect,open,write php index.php >/dev/null

There will be a *lot* of output, so here's a quick guide to figuring out what's
going on.

If you see a lot of lines like this:

	open("/home/website/www.website.com/public_html/app/code/core/Mage/Catalog/Model/Product/Media/Config.php", O_RDONLY) = 6
	open("/home/website/www.website.com/public_html/app/code/core/Mage/Media/Model/Image/Config/Interface.php", O_RDONLY) = 6
	open("/home/website/www.website.com/public_html/app/code/core/Mage/Catalog/Helper/Image.php", O_RDONLY) = 6
	open("/home/website/www.website.com/public_html/app/code/core/Mage/Catalog/Model/Product/Image.php", O_RDONLY) = 6

Then you aren't using compilation. This means that Magento is having to open
hundreds (797 in my test) of little files, and that's contributing to your time
spent in system. For our purposes though, having compilation off makes things
easier to debug, so for the next run turn it off, if it isn't off already. Look
for any places where the output appears to pause. In my test, I saw a sendto()
call block. Adding `-T` to strace gives you the relative time spend in each
system call, at the end of the line.

{% codeblock "Long strace(1) output" %}
open("/home/website/www.website.com/public_html/app/design/front-end/iweb/default/template/catalog/product/view/media.phtml", O_RDONLY) = 6 <0.000027>
open("/home/website/www.website.com/public_html/app/code/core/Mage/Catalog/Model/Product/Media/Config.php", O_RDONLY) = 6 <0.000020>
open("/home/website/www.website.com/public_html/app/code/core/Mage/Media/Model/Image/Config/Interface.php", O_RDONLY) = 6 <0.000018>
open("/home/website/www.website.com/public_html/app/code/core/Mage/Catalog/Helper/Image.php", O_RDONLY) = 6 <0.000019>
open("/home/website/www.website.com/public_html/app/code/core/Mage/Catalog/Model/Product/Image.php", O_RDONLY) = 6 <0.000019>
open("/home/website/www.website.com/public_html/media/catalog/product/c/a/camden_brown_2.png", O_RDONLY) = 6 <0.000018>
open("/home/website/www.website.com/public_html/media/catalog/product/c/a/camden_brown.png", O_RDONLY) = 6 <0.000017>
open("/home/website/www.website.com/public_html/lib/Varien/Image.php", O_RDONLY) = 6 <0.000025>
open("/home/website/www.website.com/public_html/lib/Varien/Image/Adapter.php", O_RDONLY) = 6 <0.000018>
open("/home/website/www.website.com/public_html/lib/Varien/Image/Adapter/Gd2.php", O_RDONLY) = 6 <0.000018>
open("/home/website/www.website.com/public_html/lib/Varien/Image/Adapter/Abstract.php", O_RDONLY) = 6 <0.000017>
open("/home/website/www.website.com/public_html/media/catalog/product/c/a/camden_brown.png", O_RDONLY) = 6 <0.000023>
sendto(6, "\24\0\0\0\26\0\1\3\305\tOP\0\0\0\0\0\0\0\0", 20, 0, {sa_family=AF_NETLINK, pid=0, groups=00000000}, 12) = 20 <0.000026>
connect(6, {sa_family=AF_FILE, path="/var/run/nscd/socket"}, 110) = -1 ENOENT (No such file or directory) <0.000025>
connect(6, {sa_family=AF_FILE, path="/var/run/nscd/socket"}, 110) = -1 ENOENT (No such file or directory) <0.000531>
open("/etc/host.conf", O_RDONLY)        = 6 <0.000019>
open("/etc/resolv.conf", O_RDONLY)      = 6 <0.000017>
open("/etc/hosts", O_RDONLY|O_CLOEXEC)  = 6 <0.000017>
connect(6, {sa_family=AF_INET, sin_port=htons(80), sin_addr=inet_addr("10.7.0.197")}, 16) = -1 EINPROGRESS (Operation now in progress) <0.000091>
sendto(6, "GET /media/catalog/product/c/a/c"..., 60, MSG_DONTWAIT, NULL, 0) = 60 <0.000167>
sendto(6, "Host: www.website.com\r\n", 27, MSG_DONTWAIT, NULL, 0) = 27 <0.000041>
sendto(6, "\r\n", 2, MSG_DONTWAIT, NULL, 0) = 2 <0.000192>
{% endcodeblock %}

There's a lot of information here, but taking it line by line:

* 1: The `Catalog_Product_Media` template is loaded. This is in a custom skin,
  so it's a good place to look for inadvertent changes that might affect speed.

* 2-5: Dependencies of this template are loaded by the Magento autoloader,
  `Catalog_Model_Product_Media_Config` through to `Catalog_Model_Product_Image`.

* 6-7: Loading source images from the media folder.  

* 8-11: Creating `Varien_Image` objects, loading the classes in.  

* 13-18: Doing a lookup of the website's own name, in nscd and then in
  /etc/hosts. Because the site is defined in /etc/hosts no DNS lookup is done,
  but it would happen here if that wasn't the case. It would look like this:

{% codeblock "Example DNS request" %}
connect(6, {sa_family=AF_INET, sin_port=htons(53), sin_addr=inet_addr("91.208.170.133")}, 16) = 0 <0.000021>
sendto(6, "\3140\1\0\0\1\0\0\0\0\0\0\3www\vwebsite\3com"..., 37, MSG_NOSIGNAL, NULL, 0) = 37 <0.000060>
connect(6, {sa_family=AF_INET, sin_port=htons(53), sin_addr=inet_addr("91.208.170.133")}, 16) = 0 <0.000015>
sendto(6, "&\312\1\0\0\1\0\0\0\0\0\0\3www\vwebsite\3com"..., 37, MSG_NOSIGNAL, NULL, 0) = 37 <0.000037>
connect(6, {sa_family=AF_INET, sin_port=htons(53), sin_addr=inet_addr("91.208.170.133")}, 16) = 0 <0.000015>
sendto(6, "\23m\1\0\0\1\0\0\0\0\0\0\3www\vwebsite\3com"..., 37, MSG_NOSIGNAL, NULL, 0) = 37 <0.000037>
{% endcodeblock %}

* 19: Creating a connection to the web-server (ourselves in this case).

* 20-22: Making a request for an image that we have locally.

At this point the site will block, waiting for the web-server to respond with
an image. If, as in this case, it's the local server, you may not even notice
because it's so quick, but if your site is under load, or the images are being
loaded from a CDN there could be a significant delay introduced. Let's find the
code that caused this - the best place to start looking is the .phtml file
nearest to the network request.

I can't show the full file, but a careful read through of the file shows that
we're passing a URL to a function expecting a file page. Due to PHP's [fopen()
wrappers][5] you can actually do this, and your code will run, and you may not
even realise the resources that are being consumed behind the scenes.

{% codeblock "Accidental fopen() wrapper usage" lang:php %}
<?php
$img = array(
	'zoomImg'       => $_image->getData('url'),
	'zoomImgPath'=> $_image->getData('path'),
);
$img['size'] = getimagesize($img['zoomImg']);
{% endcodeblock %}

Changing that last `zoomImg` to `zoomImgPath` resolves the issue - the results
on the front-end are the same, but no extra requests are made to the
web-server. This frees up more connections for legitimate users and has removed
a potential source for latency.

As with any micro-benchmark like this, it's important to check that the gains
that you're making are significant in the end, and also very important to check
that in your production environment you see an improvement and no regressions.
In this case, you would probably run a [siege][6] against the site before and
after the change to verify that the improvements are valid when running under
an actual web server.

If you've found this useful, please let me know on Twitter via [@insom][]

[1]: http://linux.die.net/man/1/gdb
[2]: http://linux.die.net/man/1/time
[3]: http://linux.die.net/man/1/strace
[4]: http://www.ietf.org/rfc/rfc3875
[5]: http://php.net/manual/en/wrappers.php
[6]: http://www.euperia.com/linux/tools-and-utilities/speed-testing-your-website-with-siege-part-one/720
[@insom]: http://twitter.com/insom
[Xdebug]: http://www.xdebug.org/
[KCachegrind]: http://kcachegrind.sourceforge.net/html/Home.html