---
layout: post
title: "Forcing Exim Local Delivery on Ubuntu (For Development)"
date: 2013-06-06 17:16
comments: true
categories: Exim Ubuntu
author: Aaron Brady
---

Sometimes you want to run development code against your production database before it goes live, as a test, but want to make sure that there are no side effects from this process.

Recently a developer at iWeb wanted to test some email code's interaction with a real mail server, without potentially mailing thousands of real customers. Mocks wouldn't be suitable because the tests cross the API boundary; it's not enough to know it's called, they needed to see what it *did*.

<!--more-->

#### Installing Exim

If you don't already have Exim installed, just install `exim4-daemon-light`. When Debconf asks you for the configuration type, you want "Local Delivery Only":

![Local delivery](http://17k.co.uk/s/09207fc6.png)

Select what you want for the next three dialogs, and you *do* want to split configuration into small files:

![Small files, please](http://17k.co.uk/s/91ffcd4b.png)

Now, create the file `/etc/exim4/conf.d/router/050_force_local_delivery` with the following contents:

	forced:
	  debug_print = "R: forced for $local_part@$domain"
	  driver = redirect
	  allow_fail
	  allow_defer
	  data = "iweb" # change this to a local user on your machine

Update your config and restart Exim:

	# update-exim4.conf
	# /etc/init.d/exim4 restart

Putting it in as "050" means that this will fire before the `nonlocal` router, which is in `200_exim4-config_primary`. If you see the error "Mailing to remote domains not supported", check that your file is named correctly and that you ran `update-exim4.conf` and restarted Exim.

And then give it a test:

	# mail anyone@anywhere.com < /dev/null
	No message, no subject; hope that's ok
	# tail -3 /var/log/exim4/mainlog
	2013-06-06 17:36:32 1UkdAu-0000b5-3y <= root@test.iweb U=root P=local S=338
	2013-06-06 17:36:32 1UkdAu-0000b5-3y => iweb <anyone@anywhere.com> R=local_user T=maildir_home
	2013-06-06 17:36:32 1UkdAu-0000b5-3y Completed

&nbsp;

If you've found this useful, please let me know on Twitter via [@insom][].

[@insom]: http://twitter.com/insom