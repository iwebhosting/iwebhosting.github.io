---
layout: post
title: "Hubot HTTP Daemon Support"
date: 2012-01-21 17:02
comments: true
author: Aaron Brady
categories: Hubot
---

[Hubot][1], Github's excellent CoffeeScript-based chat bot recently gained
support for running an internal HTTP daemon. Previously, this had to be done by
hand by plugins which expected to receive events from outside sources, like in
this post on [using Hubot for CI and Deploying][2], which inspired Darren's
[fleshed out take][3].

<!--more-->

This functionality hasn't yet made it into a release yet, and we also use XMPP
internally (instead of Campfire), so we had to make a few tweaks to deploy our
bot on Heroku. Being new to CoffeeScript and NPM (*and* Heroku) it took some
trial and error, but I just `git rebase`d it away so it looks like I got it
right first time!

I changed the dependency blob in the JSON to this:

{% codeblock "Depedencies to run from Hubot and XMPP master" lang:javascript %}
  "dependencies": {
    "hubot-xmpp": "git://github.com/markstory/hubot-xmpp.git",
    "hubot": "git://github.com/github/hubot",
    "hubot-scripts": "2.0.2",
    "optparse": "1.0.3"
  }
{% endcodeblock %}

Because [the HTTP support][4] requires the `PORT` environment variable (which
Heroku supplies to web apps) you'll need to change your `Procfile` from app to
web 

    web: bin/hubot -a xmpp -n Hubot

which also means changing your scale command to

    heroku ps:scale web=1

There's some [example HTTP calls on master][5], and (unlike the above
approaches) Hubot also includes Connect's [bodyParser][6] which Does the Right
Thing&trade; with POST bodies, whether they are URL encoded forms or JSON.

{% gist 1633778 %}

The above is enough for us to use Hubot like a puppet, posting the room and the
message and having it repeat it in the room:

{% codeblock "Make Hubot speak" lang:bash %}
curl -d room=yourroom@conference.iweb.co.uk \
  -d message=Hello http://appname.herokuapp.com/hubot/say
{% endcodeblock %}

So far, I've mostly been using it to fake the bot being more clever than it is

    <Hubot> [CAKE] Upstairs, via @jellis

but with our CI workflow (upcoming post!) it will start to be more genuinely
useful.

[1]: http://hubot.github.com/
[2]: http://tomb.io/posts/hubot-ci-and-deploying/
[3]: https://gist.github.com/1494013
[4]: https://github.com/github/hubot/blob/master/src/templates/README.md
[5]: https://github.com/github/hubot/blob/master/src/scripts/httpd.coffee
[6]: http://www.senchalabs.org/connect/middleware-bodyParser.html
