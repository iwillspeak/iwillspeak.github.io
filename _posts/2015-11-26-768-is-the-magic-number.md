---
title: 768 is the Magic Number
layout: post
published: true
---

When reconstrucing a binary MSMQ message from a backup I was first
tempted to just set the `Body` property on the message directly. This
doesn't quite work however as the `Body` property is serialised with
whatever formatter you're using when the message is sent.

```csharp
var mq = new MessageQueue(@".\Private%\test");
var messageBody = File.ReadAllBytes("...");

var message = new Message();
message.Body = messageBody;

mq.Send(message); // double-encodes the body :-(
```

There is a way around this serialisation though. MSDN has this to say:

> The contents of the `Body` property are serialized when the message is
> sent, using the `Formatter` property you specify. The serialized
> contents are found in the `BodyStream` property. You can also set the
> `BodyStream` property directly, for example, to send a file as the
> data content of a message.

Sounds just like what we want! Just setting the `BodyStream` property
to a `MemoryStream` with our bytes in gets us started. The only
problem is that tools peeking at the message don't know what format it
is in. This is where the magic number `768` makes it's show. By
setting the `BodyType` to this magic number tools will know the body
is serlialised with a `BinaryMessageFormatter`:

```csharp
var message = new Message();
message.BodyStream = new MemoryStream(messageBody);
message.BodyType = 768; // Binary formatted .NET object
```

Now you can send the message and recipients won't know the differnece
from the origional backup.
