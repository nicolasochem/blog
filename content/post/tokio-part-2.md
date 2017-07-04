+++
date        = "2017-07-02"
description = "How to build an incomplete MessagePack-RPC client and server with tokio-proto"
tags        = ["Rust", "Tokio"]
project_url = "https://github.com/little-dude/rmp-rpc-blog"
title = "Tokio part 2 - The Tokio stack"
author = "little-dude"
+++

This post is the third post of a serie of posts explaining how I made a tokio
based implementation of MessagePack-RPC. It is an overview of the tokio stack.

I'll assume that you understand what futures are and how they work. If
not, I recommend reading [this excellent blog
post](http://asquera.de/blog/2017-03-01/the-future-with-futures/).

--------------------------------

Disclaimer: Async IO and event loops are complex topics that I don't pretend to
understand. I just know some basic concepts that I'll try to explain with my
own words. Here is the thing: you don't need to understand these topics to use
Tokio, and that what makes it awesome in my opinion.

### tokio-core

Let's start with the main piece of Tokio:
[tokio-core](https://docs.rs/tokio-core/). `tokio-core` provided two things:

- a `tokio_core::net` module that provides TCP/UDP utilities. These utilities
  are intended to be similar to the
  [`std::net`](https://doc.rust-lang.org/std/net) ones, but they are designed
  to be asynchronous: they are not blocking. For instance,
  [`std::net::TcpStream::connect`](https://doc.rust-lang.org/std/net/struct.TcpStream.html)
  blocks until the connection is established (or fails to be established) and
  the outcome is returned, whereas
  [`tokio_core::net::TcpStream::connect`](https://docs.rs/tokio-core/0.1.8/tokio_core/net/struct.TcpStream.html)
  returns immediately a future that can be polled until it finishes.
- a
  [`Core`](https://docs.rs/tokio-core/0.1.8/tokio_core/reactor/struct.Core.html)
  (aka "reactor" or "event loop")which runs futures. We can already run futures
  with threads (via [`std::thread::spawn`](https://doc.rust-lang.org/1.6.0/std/thread/fn.spawn.html) or pools of threads (via
  [`futures_cpupool`](https://docs.rs/futures-cpupool)), so why use an
  event loop instead? I'm not entirely sure myself, but here are a few hints:

  - Threads are expensive when there are many of them, due to context switches.
    You don't want to spawn thousands of threads, especially for IO extensive
    work, since most of them are going to spend most of their time waiting
    anyway.
  - Efficiently managing multiple threads is hard. Tokio's event loop handles
    this for us. I don't need to know how many threads there are, or which ones
    should be parked or unparked.

### tokio-io, tokio-proto, and tokio-service

`tokio-core` is quite minimalistic, but tokio also provides a few crates that
make it easy to implement some common services, such as client and servers for
request/response based protocols.

- `tokio-io` contains traits and types to work asynchronously with streams of
  bytes.
- `tokio-proto` implements some logic that is common to many protocols
  request/response based protocols.
- `tokio-service` provides a `Service` trait implements how a request is
  handled.

Here is an illustration of how these crates are used together to implement server:

![tokio server stack illustration](/content/images/tokio-stack-server-view.png)

The server receives a stream of bytes from a socket. A `Codec` reads this
stream and decode meaningful messages. These messages are then passed to the
`Protocol`, which forwards the requests to the `Service`. The `Protocol` is
kind of a black box, but we can imagine that for multiplexed protocols (ie
protocol for which requests have an ID), it keeps track of the IDs and make
sure responses are sent with the ID of the request they correspond to. The
`Service` handles the request and return a response, which is in turn handled
by the `Protocol`, which passes it to the `Codec` that sends it.

The stack is quite similar for a client:

![tokio client stack illustration](/content/images/tokio-stack-client-view.png)
