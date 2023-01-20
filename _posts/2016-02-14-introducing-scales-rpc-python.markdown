---
layout: post
title: Introducing Scales - A highly scalable, performant RPC library for Python
date: '2016-02-14 19:13:02'
tags:
- python
- rpc
- thrift
- thriftmux
- finagle
---

TLDR: Check out scales on [github](https://github.com/steveniemitz/scales) and pypi as [scales-rpc](https://pypi.python.org/pypi/scales-rpc)

## Some back story...
This post has been long in the making.  

Almost a year ago, at TellApart, a coworker was debugging a weird issue with some of our Python thrift services (the client side was python, server was Java, [finagle](https://twitter.github.io/finagle/) specifically).  Randomly, but fairly frequently, we'd see our python clients getting the wrong responses from their RPC calls.  Specifically, after investigating more, we were seeing them getting the response for the *PREVIOUS* call!

This seemingly impossible scenario makes sense once you understand how the python thrift TCP client works.  It maintains a single TCP connection between RPC calls, and sends framed request / response messages over that connection.  A full RPC call is just basically:

1. Serialize message onto the wire.
- Wait for a response (by blocking on a socket read).
- Read the response.

Now, if everything is going well, you end up with a pipeline looking like:
`Req1 -> Resp1 -> Req2 -> Resp2` 

However, if the client is interrupted between step 2 and 3 (that is, it never reads the response), your pipeline is now in an undefined state, looking something like:
`Req1 -> Resp1 -> Req2 -> |Interupted| -> Req3 -> Resp2 -> Req4 -> Resp3`

As you can see, since Resp2 was never read, and the connection was never reset, all responses are now for the request BEFORE them.

"How can this happen?" you may ask.  Well, the answer is actually a fairly common operation, *client side timeouts*.  In our case, we would perform a client side timeout by aborting the greenlet that was running the request.  Since there was no code to clean up the socket after the timeout, the result was the connection getting into the above state.

Now, the short-term solution is fairly simple, after performing a timeout, the connection can simply be reset via closing an reopening the socket.  However, this is a fairly expensive operation, and our service timeouts are fairly low (in the order of 20-30ms).  Because of this, the overhead of reconnecting can be prohibitive.

At this point, you may be seeing where this is going.  We quickly realized that since Thrift itself has no solution to performing efficient client side timeouts, we needed to look for another protocol that does.  

Very conveniently, as I said above, our services were running on Twitter's Finagle RPC framework.  Finagle provides another thrift transport called ThriftMux, that was designed pretty much exactly for this use case.  ThriftMux is a multiplexed protocol, supporting multiple pending requests over a single socket, and additionally supporting robust client initiated timeouts.  We had already been using ThriftMux for services on the Java side, so there was nothing to do on the server side for this to work in python.

So, I looked around for an existing python library that could talk ThriftMux.  Sadly, finding none, I decided to write my own.  What started out as a simple ThriftMux implementation (also with the aim of improving the terrible python thrift experience), turned into a full-featured RPC library.  Thus, **Scales** was born.

## Introducing Scales
Scales was designed from the ground up to be a scalable, reliable, extensible, and asynchronous RPC library for python.

Lets look into each of those buzz words.
### Scalable
Scales was designed (and named) to support the very high performance and latency requirements at TellApart.  Our python services would need to complete RPC calls in milliseconds, with end-to-end request processing time requirements being in the sub-100ms range.  
Scales is built on top of [gevent](http://www.gevent.org/), using many of its features and optimizations to support high degrees of concurrency across many clients.  
Scales supports hundreds of downstream servers per client, and uses ZooKeeper as a first-class service discovery mechanism.  However, any other service discovery strategy (etcd, consul, etc) could be plugged in.

### Reliable
The bundled Scales load balancers and transports are built from the ground up to be highly reliable in the event of downstream failures.  Both bundled load balancers maintain the state of downstream servers and attempt to avoid ones that have failed, as well as performing robust, exponential back-off reconnects to servers marked down.

Scales has been battle-tested at TellApart and performs over 200,000 requests/sec across our production python services.

### Extensible
All components of Scales are extensible, and implementing new service types typically only requires plugging in serialization and transport logic.  

A modular design via message sinks encourages separation of concerns for a service, typically partitioning logic into serialization, transport, routing, etc.

### Asynchronous
As above, Scales is built with gevent at its core, however, importantly it does not rely on monkey patching at all.  Scales will work perfectly fine (and just as importantly, will be just as performant) in a non-monkey patched environment.

One of the key design points of Scales is that all operations are asynchronous, and this is visible in all aspects of the stack.  All operations on a Scales client have asynchronous as well as blocking versions.  

## Protocol support
Scales supports multiple protocols out of the box, exposing a consistent interface across all of them via a builder pattern.

- ThriftMux
- Thrift
- HTTP
- Kafka (producer only)
- Redis (highly experimental)

## Getting started
Getting started is simple, just `pip install scales-rpc` and start making RPC calls!  All service clients are created via a `NewClient` method, which just wraps `NewBuilder().Build()`

A simple example is making a HTTP client to get pages from www.google.com
```
from scales.http import Http
client = Http.NewClient('tcp://www.google.com:80')
response = client.Get('/')
print(response.text)
```

## More to come
In the next few weeks I'll be writing more posts explaining advanced uses of Scales, as well as going into how it was implemented.

For now, check it out on [github](https://github.com/steveniemitz/scales) or download it from [pypi](https://pypi.python.org/pypi/scales-rpc).