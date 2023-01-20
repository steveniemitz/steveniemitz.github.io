---
layout: post
title: Reverse Engineering Shade Store Electric Shades
featured: true
date: '2019-05-17 02:09:46'
tags:
- iot
- reverse-engineering
- debugging
- python
---

Reading notes: 
- You can find all the finished code for this on my [GitHub](https://github.com/steveniemitz/blindcontrol), but I recommend reading through first
- I've annotated a few places with footnotes.  These will be retrospective comments, feel free to read them after, or not at all.  Typically they'll slightly contradict something I've written in prose.  I find that many "I did this" stories tend to only talk about the successes, and not the failures.  I try to point out some pitfalls and missteps in the footnotes here.

---

Recently I bought some electric shades from the [Shade Store](https://www.theshadestore.com/collections/motorized-window-treatments).  They're great, with really high quality remote controls, but sadly I could never get their Alexa integration to work (apparently no one else could either based on their 1 star reviews), and Google Home was unsupported.[^1]

They do offer a phone app (for Android and iOS) though so you can control them from your phone.  There's a bridge that connects to your wifi that handles sending the RF commands.  While the phone connects directly to the bridge when on the same wifi network (I confirmed this by turning off my internet and disconnecting the phone from the cell network), it can also control them over the internet.  

Given all this, I decided to spend the time and reverse engineer the protocol the app used to communicate with them.  What I ended up finding was a little more surprising than I expected...

## Initial Investigations

My starting point was the phone app.  While I use an iPhone, I have a lot more experience developing (and reversing) Android apps, so I started with that.  I pulled the apk from [apkpure](https://apkpure.com/the-shade-store/com.dooya.id2ui.ssade) and got started.  The very first thing that jumped out was the apk name.  We'll come back to that soon.

## APK decompiling

To start digging into the APK, I dusted off my install of [apktool](https://ibotpeaches.github.io/Apktool/) and decompiled the apk I downloaded above, simply with `apktool d <bigname.apk>`.  This gave me a bunch of .smali files (the de-facto android bytecode assembly language), among lots of other resources and such.  Smali is actually pretty easy to understand, but I like being able to use an IDE to navigate around code, so I then "disassembled" the `classes.dex` file in the apk (the compiled bytecode) into a jar via [dex2jar](https://github.com/pxb1988/dex2jar).  This jar I could then decompile into java.  I use the decompiler built into IntelliJ, which uses [fernflower](https://github.com/JetBrains/intellij-community/tree/master/plugins/java-decompiler/engine), but others like [jd-gui](http://java-decompiler.github.io/) are great as well.

## Digging into the java code

With the actual code in a mostly-usable format I started to dig in.  There's a _LOT_ of stuff in the apk, but given the package was prefixed `com.dooya.id2ui` I figured that would be a good area to start looking at.  In here is a lot of UI related code, but what struck my eye was a `netty` package buried in there.  For those unfamiliar, [netty](https://netty.io/) is one of the leading network IO libraries in java.  This hopefully would contain some form of network communication code, so it was a likely good place to start. (we'll get back to this soon)

Also referenced by the netty package was a `com.gizwits` package, which looked like a SDK of some sort.  We'll definitely get back to that too.[^2]

## Some Googling Ensues

At this point I wanted to do some research.  A couple things stuck out:

* The app was made by com.dooya, and seemed mostly-unaffiliated with The Shade Store (not super surprising)
* The app also used some form of SDK by gizwits.

Searching for Dooya helped clarify things a lot.  Dooya is a Chinese company that looks like they specialize in in-home motorization products.  Shade Store just re-brands their white-label motors and puts their own shades on them.  Sadly, I couldn't find any useful information on any kind of developer SDK or protocol documentation on their site, but its also mostly in Chinese so :shrug: who knows.  One question down.

Next up was Gizwits.  Gizwits is a "IoT" platform from what I can gather.  This was a great find, and they have a lot of developer documentation (also though sadly mostly in Chinese).  I signed up for a developer account so I could get access to all their documentation. 

### Reverse Engineering Chinese Phone numbers

As part of the sign up process you actually need to give them a phone number.  My initial go-to of 555-555-5555 did not work here sadly.  A quick Googling of the Chinese cell-phone format turned up a [very helpful wikipedia page](https://en.wikipedia.org/wiki/Telephone_numbers_in_China).  I decided I wanted to be on the China Telecom LTE network, so I chose 133 as my prefix and made up the other 8 digits.  Hilariously this worked, and they never even asked to confirm my number.  Anyways...that was a fun distraction.  

## And we're back in the Cloud...Gizwits

With a developer account in hand I turned on Google Translate and started clicking around.  The IoT platform seems pretty interesting, and they even go so far as to provide the actual device firmware for common microcontrollers to build the devices themselves.  I was more interested in the APIs at this point though, so I kept clicking around.  After some searching I found something that looked promising, documentation for a HTTP API.[^3]  

The docs had a few interesting APIs that seemed directionally close to what I'd need.

* A login and authentication spec (not oauth though, sad).
* A way to get bound devices to an account.
* A way to control devices.

One thing that all APIs shared was a required "appid" header.  Now that I knew what to look for, I started searching back around the APK.  I was able to pretty quickly extract the appid from the resources file from my decompiled APK.  Using [Postman](https://www.getpostman.com/), a great HTTP request builder tool, I tried to log in (/app/login).  Their login supports a few methods, including login/password.  I already had an account from using the Shade Store app, so I tried my username and password, along with the appid I ripped from the APK, and it worked!  Woo.  Logging in returns a token that needs to be then sent in a header with each request.

Next up, I hit the bindings endpoint, which I assumed would return the actual shade devices I had attached.  Sadly, this turned out to be false.  Instead I only received two devices in the response.  It turns out the devices returned were actually the wifi bridges for the shades, not the shades themselves!  This was ok though, I was happy to just be able to get SOMETHING back.[^4]

**IMPORTANT NOTE: The returned devices here are addressed by something called a DID, we'll come back to that.**

This did mean though that I now needed to figure out how to enumerate the actual motors these wifi bridges were controlling.  Doing this turned out to be much more work than I expected!

## The Device Control APIs

The first nudge in the right direction I got from a sample web app distributed with their SDK ([GitHub](https://github.com/gizwits/gizwits-wechat-js-sdk)).  This was a treasure trove of information.[^5]  

The example app is built using web sockets.  From reversing the android app I knew there was some kind of push communication, but it was implemented via a helper native library, and I didn't feel like taking the time to reverse that.[^6]  This was a fully working demo of how to login on the websocket and send commands.

Now that I had this, the last piece (and most complicated!) was figuring out *what* commands to send exactly.  From what I could divine from the documentation, there appeared to be two modes, "raw" and "attr_v4".  I needed to figure out both which to use, and what to send.[^7]

### Back to the code

It was time to really start digging into the decompiled APK.  I already knew the dooya module had a lot of what appeared to be the control code, so I dug into that.  The most promising classes I found were the FrameEncoder and FrameDecoder.  Both of these were implemented as netty channel handlers, which further made me think they were what I was looking for.

The code was structured such that a raw byte payload would be decoded (by the frame decoder) into a Frame object.  By digging through how the decoder parsed the bytes into a frame object, I was able to piece together the binary spec.[^8]

Roughly, the frame consists of:
(note, all values are little-endian)

* A static preample (the string `Smart_Id1_y:`)
* The length of the payload (2 bytes)
* The frame type (2 bytes)
* A sequence number (2 bytes)
* A "reserved" field of 6 bytes (all 0s)
* Any number of data fields, where a data field is defined as:
  * A data key field (2 bytes)
  * The length of the data value (2 bytes)
  * The value itself
* A trailer byte 0xFF
* The checksum of the frame (1 byte), excluding the preamble.  The checksum calculation is simply the sum of all the bytes (excluding the preamble and length) mod 256 (eg, the last byte of the sum).

The decompiled code defined the possible frame types (~100 of them), as well as data keys.  I now needed to figure out what data keys needed to be with each request type.  To do this, I had two options:

1. Continue attempting to dig through the decompiled code and find where frames were being created.
1. Sniff traffic as it went over the wire

Neither of these were super great.  #1 was complicated by the fact that the decompiled code was incomplete in many places.  The java decompilers I tried were choking on a few fairly important classes.  #2 is just annoying to set up, and I didn't have an android phone handy (I was actually on a train while I was doing this).

### The big discovery

I decided to dig around the API more and see if I could find anything.  I eventually stumbled upon the `/app/devices/$did/raw_data` endpoint (remember DID from above?).  This was the last piece of the puzzle.  This endpoint provides a log of all messages exchanged between my devices (both my phone app, and the IoT devices themselves) and the gizwits cloud.  This was the big discovery that helped me figure the rest out.

## The raw messages

The raw messages logged definitely helped validate my reverse engineering of the frame codec above, however, there was a preamble attached before the request and response messages that I also needed to figure out.  

An example header looked like: `00 00 00 03 34 00 00 90`

After looking at a bunch of requests and responses, I was able to start to find some patterns:[^9]

- All messages had a prefix of `00 00 00 03` (an API version number maybe?)
- The next byte(s) seemed to be a [varint](https://developers.google.com/protocol-buffers/docs/encoding#varints), this would be the length of the remaining message
- The next two bytes were always `00 00`
- The last byte for requests ended with `90`, responses with `91`.

Some reverse engineering tips:

- Constants are great to find, but be sure you have enough sample diversity to make sure they're actually constant!
- VarInts can be a huge pain if you don't know what you're looking for.  However, seeing bytes with a high bit set followed by one without can be a great sign, especially if your message is also >255 bytes.
- A lot of this is incremental.  The faster you can start pulling apart pieces of a message, the more you'll understand and the faster you'll be able to get more pieces.  Optimize for forward progress initially for SOME messages and not correctness.

Once I had everything figured out, I wrote up a quick python script that would decode the message log from the `raw_data` API endpoint and run them all through the decoder, printing out "pretty" formatted frames. [^10]

### What do we send?

Now that I knew how to encode (and decode!) full frames, it was time to start pulling apart the application level protocol.  We know the application frame format, so we can decode requests that were sent.  My hunch from looking at the the enums from the decompiled APK was that `DEVICE_EXECUTE_REQ` would be useful, so I decided to look for that in the log (using the tool I built above).  Using my phone, I opened the shades in one room, and then checked the device log.  Sure enough, I saw a frame with `DEVICE_EXECUTE_REQ` as the frame type.

The frame itself had a couple data keys as well:

- `DEVICE_CMD`
- `DEVICE_ADDR_CHANNEL`

Now that I knew what I was looking for, I poked around the APK code looking for uses of `DEVICE_CMD`.  I found some good hits, it looked like the `MotoCmd` enum was used as values for the key, and sure enough, it had useful values like `UP` and `DOWN`.  These also matched the example frame I was looking at!

`DEVICE_ADDR_CHANNEL` was an interesting one, since no API call I had found returned device (the actual shade) level data yet.  I knew I was missing something, so I had to figure out how to actually enumerate the devices.

Conveniently, in dumping out all the frames I had captured, I found `DEVICE_LIST_REQ`.  This was a simple one since it had no data fields.  The response (`DEVICE_LIST_RESP`) was one BIG frame (a few hundred bytes!), that contained a few fields for each device associated with a bridge.  Useful fields there were `DEVICE_ADDR_CHANNEL` (we were looking for this!) and (`NAME`).

I finally had enough to actually do something useful.

## Putting it all together, I HAVE CONTROL

Now I was really close.  With a full frame spec in hand, it was time to build the frame encoder and a PoC client.  (aside: my first iteration of this was in scala, but I later ported it to python so it'd perform better on AppEngine.  I'll use python for code samples.)

First, we needed a way to communicate with the Gizwits cloud.  As I mentioned above, I had found a websocket example already, so started with websockets as my communication channel.

Establishing a connection was fairly simple, the websocket address is returned in the bindings API endpoint.

The flow then goes:

- Enumerate all bound devices (bridges) and get their DIDs (remember this from above)
- Connect the websocket to the host in the result from above
- Login using a `login_req` message
- Wait for a `login_resp` message back
- Send a `DEVICE_LIST_REQ` for each DID
- Read and parse the `DEVICE_LIST_RESP` into a useful structure, keeping track of name and channel
- Pick a device we want to control
- Build a `DEVICE_EXECUTE_REQ` frame, using the `DEVICE_ADDR_CHANNEL` we got from the list response
- Send the frame over the websocket, addressed to the DID we enumerated above that returned the shade device we wanted to control.
- ?????
- Watch our shade go up! (or down)

(Messages can also be posted directly over the REST API, but my initial iteration simply used a websocket.)

## The end(...?)

Thats enough for this one post.  We successfully reverse engineered everything needed to control our shades remotely.

In the next post, I'll talk about building a Google Home Skill, and how to integrate with what we've built above into it.

You can check out all the code for this on my [github](https://github.com/steveniemitz/blindcontrol), and I'll be writing more posts walking through the code there.

[^1]: My first footnote! In retrospect maybe I should have just bought shades that supported Alexa and Google Home...but then I wouldn't have ever had any of this fun!
[^2]: This probably took the most "energy of activation" because I was starting at nothing.  Once I started getting some leads, classes to look at, things to google, etc, the reversing process started moving much faster.
[^3]: I actually thing I stumbled across the HTTP API by just going to some of the endpoints I found referenced in the configuration and trying different URLs.  I did eventually find the HTTP API docs as well though.
[^4]: I don't talk about it here because it isn't very interesting, but I spent probably ~1 hour trying all the different API endpoints and trying to figure out what they did (if anything).  In retrospect it was useful because it helped me get more familiar with the APIs and terminology.
[^5]: Again, I'm glossing over a few hours of searching around their website and the internet.  A lot of time was spent just looking around google hits for "gizwits sdk", checking out what they had to download, etc.  I even spent time looking at their C code for the MCU firmware they distribute hoping that I'd be able to find useful code there.  The web app example was definitely the tipping point though.
[^6]: In retrospect, I actually believe the native library uses MQTT rather than web sockets.
[^7]: This is a good example of how knowledge compounds.  Once I had found the websocket documentation and example, things started moving MUCH faster.
[^8]: In actuality the bundled code was SO poorly written I'm not sure how much value it provided, but I would use it as a reference when writing my own decoder.
[^9]: Hilariously, later, I found this well documented in the websocket API documentation on the Gizwits developer site as well.  It's funny what you find once you know what to look for...
[^10]: This was done mostly concurrently with the actual reverse engineering, it'd be pretty difficult to do this without iterating and seeing what works.