---
layout: post
title: "'Error parsing body' in Magento installer"
date: 2013-01-23 19:59
comments: true
categories: magento
author: Darren Worrall
published: false
---

During the configuration step of the Magento installer, a sanity check is ran against the base url to check its accessible. As some people on the Magento forums [have noticed][forum-post], if you run through the Magento installer while running under [nginx][nginx] and [fpm][fpm], you can hit an error:

{% img /images/magento-installer-error.png %}

You know the url is fine because you're looking at it right now. As noted in the discussion thread, you can safely ignore the error by selecting 'Skip Base URL Validation Before the Next Step', but we were curious to figure out what was actually going on here.

A quick grep through the magento source code shows that particular exception being thrown from the Zend Response object:

{% codeblock Response.php lang:php %}
while (trim($body)) {
    if (! preg_match("/^([\da-fA-F]+)[^\r\n]*\r\n/sm", $body, $m)) {
        #require_once 'Zend/Http/Exception.php';
        throw new Zend_Http_Exception("Error parsing body - doesn't seem to be a chunked message");
}
{% endcodeblock %}

So Zend is having trouble parsing a response sent with chunked transfer encoding. Lets rewind a bit and see if we can figure out what request its trying to make, and what response its parsing. Using tcpdump, we can see that the request headers look like this:

{% codeblock %}
GET /index.php/install/wizard/checkHost/ HTTP/1.1
Accept: */*
Host: my.working.host
Connection: close
Accept-encoding: gzip, deflate
User-Agent: Varien_Http_Client
{% endcodeblock %}

The user agent is a dead give away that this is the request we are looking for. HTTP/1.1 says that the client will support chunked encoding, and the Accept-encoding header says it will accept a gzipped response - we already know that the response object is failing to parse a chunked response. Lets look at the response headers:

{% codeblock %}
HTTP/1.1 200 OK
Server: nginx/1.1.19
Date: Wed, 23 Jan 2013 17:19:26 GMT
Content-Type: text/html; charset=UTF-8
Transfer-Encoding: chunked
Connection: close
X-Powered-By: PHP/5.3.10-1ubuntu3.5
Set-Cookie: PHPSESSID=35k9l5uug9ccndh1sphg6sjr43; expires=Wed, 23-Jan-2013 18:19:26 GMT; path=/; domain=my.working.host; HttpOnly
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Cache-Control: no-store, no-cache, must-revalidate, post-check=0, pre-check=0
Pragma: no-cache
Content-Encoding: gzip
{% endcodeblock %}

So the response is chunked, and is gzipped, but Zend is telling that server that should be ok. Just what is that response? We can pull it out of the tcpdump of course, but if we `die(var_dump($body))` just before `trim` is called, we're told that $body looks like `string(27) "‹óutwõñÃZx"`


[forum-post]:http://www.magentocommerce.com/boards/26245/viewthread/280993
[nginx]:http://nginx.org/
[fpm]:http://php.net/manual/en/install.fpm.php
