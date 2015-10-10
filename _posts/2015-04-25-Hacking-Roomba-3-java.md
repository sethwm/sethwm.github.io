---
layout: post
title: Hacking the Roomba Part 3, Java Interop
permalink: /hacking-roomba-3/
---

In my [last post]({{site.baseurl}}//hacking-roomba-2), I affirmed that I could talk to my [rootooth](https://www.sparkfun.com/products/12581) directly through the serial device.  The next step in getting things up and running with clojure is a java shim.  There's a great library out there called [roombacomm](http://hackingroomba.com/code/roombacomm/), but it uses [rxtx](http://rxtx.qbang.org/) for serial communication which proved problematic for me.  This post will talk about getting our basic beep program working in Java.

## Serial Communication in Java

Serial port i/o in Java is in a fairly rough state, as [Jim Connors reflects](https://blogs.oracle.com/jtc/entry/java_serial_communications_revisited).  There is [some work](https://wiki.openjdk.java.net/display/dio/Main) to elevate serial communication to a first class citizen in java, but that's in the future.

In the meantime, there's a bunch of open source libraries for serial communication in java, with varying levels of usability:

* [RxTx](http://rxtx.qbang.org)
* [jSerialComm](http://fazecast.github.io/jSerialComm/)
* [PureJavaComm](http://www.sparetimelabs.com/purejavacomm/purejavacomm.php)
* [Java Simple Serial Connector](https://code.google.com/p/java-simple-serial-connector/)

#### RxTx

Roombacomm is built on rxtx.  For whatever reason, rxtx hasn't had active maintenance in a while.  None of the existing pre-built binary versions floating around seem to work on my Mac in Java 8.  So I decided to build from souce.  There's a lot of options.

* [Fizzed](http://fizzed.com/oss/rxtx-for-java)
* [on GitHub](https://github.com/rxtx/rxtx)
* [Oracle](https://blogs.oracle.com/jtc/resource/rxtx-2.1-7r2-Java8/rxtx-2.1-7r2.zip)

I'm building for Java 8 on OS X, and some changes had to be made to the source to get it working.  This [post on StackOverflow](http://stackoverflow.com/questions/13139765/rxtxserial-dll-for-macos-10-8) describes it well.  One thing that was a bit of a trick for me is that the build wants glibtool, but I only had libtool.  I tried changing it in the Makefile, but the build failed.  However if I attempted to reinstall libtool via `brew install libtool`, it recognized that I already had libtool, and installed the new software as glibtool.  Unexpected but welcome.  After building, I dropped the artifacts in the java library location:

```bash
$ ls -l /Library/Java/Extensions
total 256
-rw-r--r--  1 seth  Users  59356 Apr 25 18:27 RXTXcomm.jar
-rwxr-xr-x  1 seth  Users  69156 Apr 25 18:27 librxtxSerial.jnilib
```

I converted the beep program from [my last post]({{site.baseurl}}//hacking-roomba-2) into a Java program.  Call it my hello world:

```java
/*
 * The MIT License (MIT)
 * 
 * Copyright (c) 2015 Seth Markle, http://sethwm.github.io
 * 
 * Permission is hereby granted, free of charge, to any person obtaining a copy
 * of this software and associated documentation files (the "Software"), to deal
 * in the Software without restriction, including without limitation the rights
 * to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
 * copies of the Software, and to permit persons to whom the Software is
 * furnished to do so, subject to the following conditions:
 * 
 * The above copyright notice and this permission notice shall be included in
 * all copies or substantial portions of the Software.
 * 
 * THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
 * IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
 * FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
 * AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
 * LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
 * OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN
 * THE SOFTWARE.
 */

import gnu.io.*;
import java.io.*;
import java.util.*;

public class HelloWorld {

    private static final String PORT_NAME = "/dev/cu.RNBT-73C3-RNI-SPP";
    private static final int RATE     = 115200;
    private static final int DATABITS = 8;
    private static final int PARITY   = SerialPort.PARITY_NONE;
    private static final int STOPBITS = SerialPort.STOPBITS_1;

    public static void main(String argv[]) throws Exception {
        
        SerialPort port = null;
        InputStream input = null;
        OutputStream output = null;
        try {
            /*
             * Instantiate Serial Port
             */
            Enumeration portList = CommPortIdentifier.getPortIdentifiers();
            while (portList.hasMoreElements()) {
                CommPortIdentifier portId = (CommPortIdentifier) portList.nextElement();
                System.err.println("Found comm port: " + portId.getName());
        
                if (portId.getPortType() == CommPortIdentifier.PORT_SERIAL) {
                    if (portId.getName().equals(PORT_NAME)) {
                        port = (SerialPort)portId.open("hello world", 20_000);
                        input  = port.getInputStream();
                        output = port.getOutputStream();
                        port.setSerialPortParams(RATE, DATABITS, STOPBITS, PARITY);
                    }
                }
            }
            System.err.println("Open attempt: SUCCEEDED");
    
            /*
             * Kick the port
             */
            port.setDTR(false);
            try { Thread.sleep(500); } catch(Exception e) { }
            port.setDTR(true);

            /*
             * Speak!
             */
            if (output == null) return;
            System.err.println("Sending bytes");
            byte[] bytes = toBytes(new int[] {
                128, 132,
                140, 0, 1, 62, 32,
                141, 0
            });
            for (int i = 0; i < bytes.length; i++) {
                output.write(bytes[i]);
                Thread.sleep(250);
            }
            output.flush();
        } finally {
            if (input != null) input.close();
            if (output != null) output.close();
            if (port != null) port.close();
            System.err.println("Disconnected");
        }

    }

    private static byte[] toBytes(int[] input) {
        byte[] bytes = new byte[input.length];
        for (int i = 0; i < input.length; i++) bytes[i] = (byte) (input[i] & 0xff);
        return bytes;
    }

}
```

And it failed.  I got a `gnu.io.PortInUseException`.  Luckily someone has [solved](http://trac.research.cc.gatech.edu/GART/browser/sandbox/comm/edu/gatech/gart/sensor/witiltv3/SerialComm.java?rev=150) this:

> For reasons I don't understand RXTX is not using `/var/spool/uucp` for lock files.  It's using `/var/lock`.  A directory that didn't exist on my system.  Once I created this directory I started getting different errors for the RXTX code.  This directory requires read/write permissions for the user.  I just used the blanket `chmod 777`.

That got me to my next issue.  The program worked (the roomba beeped), but it segfaulted afterwards, which is a seg fault in the native code:

```
$ java -cp . HelloWorld     
Experimental:  JNI_OnLoad called.
Stable Library
=========================================
Native lib Version = RXTX-2.1-7
Java lib Version   = RXTX-2.1-7
Found comm port: /dev/tty.Bluetooth-Incoming-Port
Found comm port: /dev/cu.Bluetooth-Incoming-Port
Found comm port: /dev/tty.Bluetooth-Modem
Found comm port: /dev/cu.Bluetooth-Modem
Found comm port: /dev/tty.RNBT-73C3-RNI-SPP
Found comm port: /dev/cu.RNBT-73C3-RNI-SPP
Open attempt: SUCCEEDED
Sending bytes
#
# A fatal error has been detected by the Java Runtime Environment:
#
#  SIGSEGV (0xb) at pc=0x0000000125ae26b4, pid=52675, tid=3335
#
# JRE version: Java(TM) SE Runtime Environment (8.0_45-b14) (build 1.8.0_45-b14)
# Java VM: Java HotSpot(TM) 64-Bit Server VM (25.45-b02 mixed mode bsd-amd64 compressed oops)
# Problematic frame:
# C  [librxtxSerial.jnilib+0x36b4]  Java_gnu_io_RXTXPort_nativeDrain+0x184
#
# Failed to write core dump. Core dumps have been disabled. To enable core dumping, try "ulimit -c unlimited" before starting Java again
#
# An error report file with more information is saved as:
# /Users/seth/roomba/roombacomm-0.96/hs_err_pid52675.log
#
# If you would like to submit a bug report, please visit:
#   http://bugreport.java.com/bugreport/crash.jsp
# The crash happened outside the Java Virtual Machine in native code.
# See problematic frame for where to report the bug.
#
```

Even though the roomba responded, the segfault is concerning.  Based on the function where it crashed and the output to stderr, it likely crashes in `output.flush()`.  Commenting out that line affirms that hypothesis.  So RxTx might actually work here.  However, sometimes disconnect doesn't work and I have to unpair and re-pair after running it.  So it might make sense to explore another serial library just in case I need it.   "[Why build one when you can have two at twice the price?](http://www.imdb.com/title/tt0118884/quotes?item=qt1542424)"

#### jSerialComm

jSerialComm was next on my list to try.  Why?  Because their website is current, so maybe this has a chance of working.  Roombacomm doesn't use jSerialComm, but I can rewrite roombacomm if I need to.  Here is the HelloWorld application for jSerialComm.  And it actually works without dumping.

```java
/*
 * The MIT License (MIT)
 * 
 * Copyright (c) 2015 Seth Markle, http://sethwm.github.io
 * 
 * Permission is hereby granted, free of charge, to any person obtaining a copy
 * of this software and associated documentation files (the "Software"), to deal
 * in the Software without restriction, including without limitation the rights
 * to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
 * copies of the Software, and to permit persons to whom the Software is
 * furnished to do so, subject to the following conditions:
 * 
 * The above copyright notice and this permission notice shall be included in
 * all copies or substantial portions of the Software.
 * 
 * THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
 * IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
 * FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
 * AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
 * LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
 * OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN
 * THE SOFTWARE.
 */

import com.fazecast.jSerialComm.*;
import java.io.*;
import java.util.*;

public class HelloWorld {

    private static final String PORT_NAME = "/dev/cu.RNBT-73C3-RNI-SPP";
    private static final int RATE     = 115200;
    private static final int DATABITS = 8;
    private static final int PARITY   = SerialPort.NO_PARITY;
    private static final int STOPBITS = SerialPort.ONE_STOP_BIT;

    public static void main(String argv[]) throws Exception {
        
        SerialPort port = null;
        InputStream input = null;
        OutputStream output = null;
        try {
            /*
             * Instantiate Serial Port
             */
            for (SerialPort p : SerialPort.getCommPorts()) {
                if (p.getSystemPortName().equals(PORT_NAME)) port = p;
                System.err.println("Found comm port: " + p.getSystemPortName());
            }
            if (port == null) throw new IllegalStateException("Couldn't find: " + PORT_NAME);
    
            /*
             * Connect
             */
            boolean opened = port.openPort();
            System.err.println("Open attempt: " + (opened?"SUCCEEDED":"FAILED"));
            input = port.getInputStream();
            output = port.getOutputStream();
            port.setComPortParameters(RATE, DATABITS, STOPBITS, PARITY);
    
            /*
             * Kick the port
             */
            int settings = port.getFlowControlSettings();
            port.setFlowControl(settings ^ SerialPort.FLOW_CONTROL_DTR_ENABLED);
            try { Thread.sleep(500); } catch(Exception e) { }
            port.setFlowControl(settings);
    
            /*
             * Speak!
             */
            if (output == null) return;
            System.err.println("Sending bytes");
            byte[] bytes = toBytes(new int[] {
                128, 132,
                140, 0, 1, 62, 32,
                141, 0
            });
            for (int i = 0; i < bytes.length; i++) {
                output.write(bytes[i]);
                Thread.sleep(250); // seems to work better when I slow things down
            }
            output.flush();
        } finally {
            if (input != null) input.close();
            if (output != null) output.close();
            boolean success = false;
            if (port != null) success = port.closePort();
            System.err.println("Disconnected: " + success);
        }

    }

    private static byte[] toBytes(int[] input) {
        byte[] bytes = new byte[input.length];
        for (int i = 0; i < input.length; i++) bytes[i] = (byte) (input[i] & 0xff);
        return bytes;
    }

}
```

## Conclusion
There are a variety of serial communication libraries for java.  At least two of them actually work with OS X 10.9.5 and Java 8, though one of them segfaulted along the way.  Next we'll look at expanding our HelloWorld example to use roombacomm directly from java.
