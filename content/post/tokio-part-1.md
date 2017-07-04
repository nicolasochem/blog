+++
date        = "2017-07-02"
description = "Explanation of what MessagePack-RPC is"
tags        = ["Rust", "Tokio"]
project_url = "https://github.com/little-dude/rmp-rpc-blog"
title = "Tokio part 1 - MessagePack-RPC"
author = "little-dude"
+++

This post is the first of a serie of posts explaining how I made a tokio based
implementation of MessagePack-RPC. It explains what MessagePack-RPC is about.
If you're already familiar with RPC concepts, you may want to skip this
article.

-----------------------

### RPC

RPC stands for Remote Procedure Call. An RPC protocol defines a way for
processes to communicate. One process (we'll call it the _server_) opens an
input channel (usually a socket, but it can also be a stream like `stdin` for
example), and waits for commands. Then other processes (we'll call them
_clients_) can start sending messages to the server. The RPC protocol defines
defines the format of the messages, as well as how they are exchanged. The
[wikipedia article](https://en.wikipedia.org/wiki/Remote_procedure_call) gives
more details.

One widely used and simple RPC protocol is
[JSON-RPC](https://en.wikipedia.org/wiki/JSON-RPC). Clients and server exchange
JSON messages. Clients can send two types of messages: _requests_ and
_notifications_. A server can only send one type of message: `reponses`. When a
server receives a _request_, it **must** answer with a _response_. When it
receives a _notification_ it **must not** answer.

A _request_ message is a JSON string that looks like this:

```
{
    "jsonrpc": "2.0",
    "id": 3,
    "method": "add",
    "params": [2, 2],
}
```

- the `jsonrpc` attribute is the version of the protocol.
- the `id` attribute identify a request. Each request has a different ID.
- the `method` is the name of the command sent to the server.
- `params` can be an arbitrary JSON value. Here, it is a simple list of integers.

This requests is basically telling the server to `add` `2` and `2`. Here is a
potential response:

```
{
    "jsonrpc": "2.0",
    "id": 3,
    "result": "4",
}
```

- the `id` is the same than the request's `id`. If the client sends multiple
  requests, it can identify which response is for which request. We say that
  the protocol is **multiplexed**.
- the `result` can be an arbitrary JSON value. Here, it is an integer.

A _notification_ is similar to a request, but since no response is expected by
the clients, it does not have an ID:

```
{
    "jsonrpc": "2.0",
    "method": "save",
    "params": "/home/me/a_file.txt",
}
```

### Serialization formats

A serialization format defines a representation of data as a stream of
bytes. JSON is a serialization format, but there are other formats like
[protobuf](https://github.com/google/protobuf) or
[MessagePack](http://msgpack.org/).

Each programming language has its own representation of data and the power of
serialization formats is that they provide a common representation. For
instance a list of integer will is represented as `"[1, 2, 3]"` in JSON. But
both Rust and Python are able to deserialize this list and have their own
representation of it:

```
Rust                    JSON                  Python
          serialize             deserialize
Vec<u32> -----------> "[1,2,3]" -----------> [1, 2, 3]
```

In the context of Rust we can say that a serialization format defines how to
represent a `struct` as a stream of bytes.  This is what
[serde](http://serde.rs/) does.


### MessagePack-RPC

**MessagePack** is a [serialization](https://en.wikipedia.org/wiki/Serialization)
format very similar to JSON, but more compact, which makes it faster and
lighter to send over the network for example.

**MessagePack-RPC** is an RPC protocol that uses the MessagePack to exchange
message. It is extremly similar to the JSON-RPC protocol described above,
except that messages don't have any version field.

To quote [msgpack.org](msgpack.org), "it's like JSON, but fast and small". You
can find the [specifications on
github](https://github.com/msgpack/msgpack/blob/master/spec.md). In the Rust
ecosystem, the main implementation of MessagePack is
[`rmp`](https://github.com/3Hren/msgpack-rust) (which stands for Rust
MessagePack).
