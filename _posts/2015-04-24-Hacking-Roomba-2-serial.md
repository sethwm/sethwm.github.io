---
layout: post
title: Hacking the Roomba Part 2, Serial Communication
permalink: /hacking-roomba-2/
---

In my [previous post]({{site.baseurl}}/hacking-roomba-1/), I talked about ordering a [rootooth](https://www.sparkfun.com/products/12581) to get started playing with robots and clojure.  In this post I talk about communicating with it over the serial port.  Before writing any real code, I wanted to talk to the roomba directly over the serial interface just to make sure things worked.

Before reading this, you might want to read a [basic refresher on serial communication](https://learn.sparkfun.com/tutorials/serial-communication).

## Documentation
There's two key pieces of documentation to get started to use the rootooth.  They're both important.

* Roomba Docs
  * [iRobot&reg; SCI Specification](http://www.robotappstore.com/files/KB/Roomba/Roomba_SCI_Spec_Manual.pdf)
  * [iRobot&reg; Create Open Interface](http://www.irobot.com/filelibrary/pdfs/hrd/create/Create%20Open%20Interface_v2.pdf)
* Rootooth Docs
  * [Bluetooth Data Module Command Reference](http://cdn.sparkfun.com/datasheets/Wireless/Bluetooth/bluetooth_cr_UG-v1.0r.pdf)

You'll notice that the roomba docs have two versions with different cover pages.  I haven't taken the time to dig into what the difference is.  I've been using the first one myself.  They're probably just different revisions of the same interface.

## Toolset

There's a ton of tools available for communicating with serial devices.  Google will be your friend here.  There's two that I gravitate towards.

* [screen](https://www.gnu.org/software/screen/manual/screen.html), which is terminal multiplexer that can also [speak to serial devices](http://www.cyberciti.biz/hardware/5-linux-unix-commands-for-connecting-to-the-serial-console/).  Odds are good that if you're going to use screen for this, you already know what it is.
* [CoolTerm](http://freeware.the-meiers.org/) is a graphical serial terminal.  Available for most platforms.

The thing I like about CoolTerm is that it makes it easy to send arbitrary hex to the device.  Most of the Roomba commands aren't in the basic ascii, so it's annoying to type commands.  To use this feature, under the Connection menu select "Send String..." and choose hex.

#### Interfaces
First you need to pair with the roomba device, however you'd pair with a bluetooth device on your platform.  Some docs say the pin code is `1234`, others say `0000`.  However my device required no pin.  So good luck with that.

Once you pair, the rootooth presents a device under `/dev` that you could communicate with like any other device.  For this, I'm using OS X.  The rootooth presented two devices:

* `/dev/cu.RNBT-73C3-RNI-SPP`
* `/dev/tty.RNBT-73C3-RNI-SPP`

StackOverflow explains [the difference](http://stackoverflow.com/questions/8632586/macos-whats-the-difference-between-dev-tty-and-dev-cu) between `/dev/cu.*` and `/dev/tty.*`.  Net result: `/dev/cu.*` is the outbound device.

#### Baud Rate
I had a little trouble at first setting the baud rate setup for screen.  I was trying to use `stty` but couldn't get the results to stick.  As it turns out, BSD variants (eg. Darwin/OS X) behave differently than linux when it comes to serial port configuration.  [This site](http://www.clearchain.com/blog/posts/using-serial-devices-in-freebsd-how-to-set-a-terminal-baud-rate) and [this one](https://discussions.apple.com/thread/3798003) provide some clarity.

CoolTerm makes this much easier.  For the roomba, depending on the model of the device, your serial config will be one of:

* 57600 8N1 (roomba 4xx and Dirt Dog)
* 115200 8N1 (roomba 5xx and 7xx)

## False Start

My first attempts at communication failed.  I made an assumption that, in hindsight, was ridiculous.  When you communicate with the rootooth, you're talking over bluetooth.  The rootooth itself establishes a serial connection to the roomba.  Which means that the baud configuration set on my laptop was ignored.  The rootooth is preset with its own serial configuration.  And it wasn't the right one to talk to the roomba.

I should add that I wasted days on this.

## Rootooth Configuration

To fix this, you have to interact with the rootooth's command module.  The reference above describes the module, but here's what needs to happen.  Note that there's no echoing or prompts when you log in, you'll be typing blind.  However, I'm going to illustrate the configuration as if there's echoing and prompts.  Just makes it easier to demonstrate.  This uses an ascii command set, so screen or CoolTerm work fine.  The following sets the baud rate to 115200.

```
> $$$
CMD
> ST,255
AOK
> U,115k,N
AOK
> D
***Settings***
BTA=0006666973C3
BTName=RNBT-73C3
Baudrt=115K
Mode  =Slav
Authen=1
PinCod=1234
Bonded=0
Rem=NONE SET
```

## Success!

With the rootooth properly configured, you can talk to the roomba now.  Here's a sample sequence of commands that gets the Roomba to beep:

```
128 132
140 0 1 62 32
141 0
```

I wrote a quick ruby program to convert that to hex:

```ruby
#!/usr/bin/ruby

# The MIT License (MIT)
#
# Copyright (c) 2015 - Seth Markle - http://sethwm.github.io
#
# Permission is hereby granted, free of charge, to any person 
# obtaining a copy of this software and associated documentation 
# files (the "Software"), to deal in the Software without restriction,
# including without limitation the rights to use, copy, modify, 
# merge, publish, distribute, sublicense, and/or sell copies of 
# the Software, and to permit persons to whom the Software is 
# furnished to do so, subject to the following conditions:
#
# The above copyright notice and this permission notice shall 
# be included in all copies or substantial portions of the Software.
#
# THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, 
# EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES 
# OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND 
# NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT 
# HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, 
# WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, 
# OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER 
# DEALINGS IN THE SOFTWARE.

hex_lines = []

$stdin.readlines.each{ |line|
    parts = line.chomp.split(/ /)
    results = []
    parts.each{|part| results << "%02x" % part.to_i }
    hex_lines << results
}

hex_lines.each{|l|
    l.each{|v| print "#{v} " }
    puts
}
puts "-------"
hex_lines.each{|l|
    l.each{|v| print "#{v} " }
}
puts
```

Which produces:

```bash
$ echo "128 132\n140 0 1 62 32\n141 0" | ./script
80 84 
8c 00 01 3e 20 
8d 00 
-------
80 84 8c 00 01 3e 20 8d 00 
```

Sending that to the roomba via CoolTerm gets it to beep.

## Credits
Anna Sandler from the [Robot App Store](http://www.robotappstore.com/) wrote a [great tutorial](http://www.robotappstore.com/Knowledge-Base/1-Introduction-to-Roomba-Programming/15.html) on hacking the roomba.  [Hadrien Hamel](http://hbot.benou.fr/doku.php?id=about) wrote a helpful note on [configuring the roomba](http://hbot.benou.fr/doku.php?id=rootooth_howto).
