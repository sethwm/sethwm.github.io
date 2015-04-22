---
layout: post
title: Hacking the Roomba Part 1, Introduction
permalink: /hacking-roomba-1/
---

I'd like to get my daughter interested in Software Development.  At the time of this writing, she's 4 years old.  There's a whole bunch of resources out there to teach programming to small children, but first I want to spark her interest.  I figure a good way to do that is to build something fun to play with.  That could be a game or some sort of art tool, but I feel like a robot would be even better.  What kid could resist, right?

Unrelatedly, I've been wanting to learn [clojure] for some time now but can't justify working clojure into any legitimate project I'm working on.  So an idea is born: let's mix robotics and clojure.

So what now?  Firing up the [ol' search engine] brings me to a [great talk] from [Carin Meier].  Apparently newish models of the [roomba vacuum] have an accessible serial port and a command interface.  Well... I have a roomba!  Got it as a wedding gift.  It hasn't seen much action in the past couple years, but I still have it.  And although it's a couple years old, my roomba is new enough to have a serial port (as [visual inspection] proves).

This seems like an affordable way to get started with my project.  Carin's video mentions a device called the [Rootooth], a bluetooth-to-serial-port adapter made for the roomba.  After a brief bout with insanity where I contemplated [building my own rootooth], I decided to just buy one.  Building it is only going to save me a couple bucks, and the intent here isn't to build a hardware component.  So instead I just bought it.

It arrived a couple weeks ago.  I don't have a tremendous surplus of time, so I've been working on it here and there at night or on weekeneds.  It hasn't gone smoothly, so the next few posts will document some of the things I've learned.  In fact, as of the time of this writing, I haven't gotten it completely working yet.  Soon I hope.

[clojure]: http://clojure.org/
[ol' search engine]: https://www.google.com/search?q=robots+and+clojure&ie=utf-8&oe=utf-8
[great talk]: http://www.infoq.com/presentations/clojure-robots
[Carin Meier]: http://gigasquidsoftware.com/#/about/index
[roomba vacuum]: http://www.irobot.com/roomba
[visual inspection]: https://www.youtube.com/watch?v=PEk30wPorZ8
[Rootooth]: https://www.sparkfun.com/products/12581
[building my own rootooth]: http://hackingroomba.com/projects/build-a-roomba-bluetooth-adapter/
