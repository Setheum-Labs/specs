# Multistream 2.0

This proposal describes a replacement protocol for multistream-select.

## Protocols

This document proposes 5 new, micro-protocols with two guiding principles:

1. Composition over complexity.
2. Every byte and round-trip counts.

This document *does not*, in fact, propose a protocol *negotiation* protocol.
Instead, it proposes a set of stream/protocol management protocols that can be
composed to flexibly negotiate protocols.

First, this document proposes 4 protocol "negotiation" protocols. "Negotiation"
is in quotes because none of these protocols actually involve negotiating
anything.

1. `multistream/advertise`: Inform the remote end about which protocols we
   speak. This should partially replace the current identify protocol.
2. `multistream/use`: Selects the stream's protocol using a multicodec.
3. `multistream/dynamic`: Selects the stream's protocol using a string protocol name.
4. `multistream/contextual`: Selects the stream's protocol using a protocol ID
   defined by the *receiver*, valid for the duration of the "session"
   (underlying connection). To use this, the *receiver* must have used the
   `multistream/advertise` To inform the initiator of *it's* mapping between
   protocols and contextual IDs.

Second, this document proposes an auxiliary protocol that can be used with the 4
multistream protocols to actually negotiate protocols. This is *primarily*
useful (a) in packet-based protocols (without sessions) and (b) when initially
negotiating a transport session (before protocols have been advertised and the
stream multiplexer has been configured).

1. `serial-stream`: A simple stream "multiplexer" that can multiplex multiple
   streams *in serial* over the same connection. That is, it allows us to
   negotiate a protocol, use it, and then return to multistream. It also allows
   us to speculatively choose a single protocol and then drop back down to
   multistream if that doesn't work.
   
All peers *must* implement `multistream/use` and *should* implement
`serial-stream`. This combination will allow us to apply a series of quick
connection upgrades (e.g., to multistream 3.0) with no round trips and no funny
business (learn from past mistakes).

Notes:

1. The "ls" feature of multistream has been removed. While useful, this really
   should be a *protocol*. Given the `serial-stream` protocol, this shouldn't be
   an issue as we can run as many sub-protocols over the same stream as we want.
2. To reduce RTTs, all protocols are unidirectional.
3. These protocols were *also* designed to eventually support packet protocols.
4. We considered a `speculative-stream` protocol where the initiator
   speculatively starts multiple streams and the receiver acts on at most one.
   This would have allowed for 0-RTT worst-case protocol negotiation but was
   deemed too complicated for inclusion in the core spec.

### Multistream Advertise

Unspeced (for now). Really, we just need to send a mapping of protocol
names/codecs to contextual IDs (and may be some service discovery information).
This is the subset of identify needed for protocol negotiation.

### Multistream Use

The `multistream/use` protocol is simply two varint multicodecs: the
multistream-use multicodec followed by the multicodec for the protocol to be
used. This protocol supports unidirectional streams. If the stream is
bidirectional, the receiver must acknowledge a successful protocol negotiation
by responding with the same multistream-use protocol sequence.

Every stream starts with multistream-use. Every other protocol defined here will
be assigned a multicodec and selected with `multistream/use.`

This protocol should *also* be trivial to optimize in hardware simply by prefix
matching (i.e., matching on the first N (usually 16-32) bits of the
stream/message).

### Multistream Dynamic

The `multistream/dynamic` protocol is like the `multistream/use` protocol
*except* that it uses a string to identify the protocol. To do so, the initiator
simply sends a varint length followed by the name of the protocol.

Including the `multistream/use` portion, the initiator would send:

```
<multistream/use><multistream/dynamic><length(varint)><name(string)>
```

Note: This used to use a fixed-width 16 bit number for a length. However, a
varint *really* isn't going to cost us much, if anything, in terms of
performance as most protocol names will be <= 128 bytes long. On the other hand,
using different number formats everywhere *will* cost us in terms of complexity.

### Multistream Contextual

The `multistream/contextual` protocol is used to select a protocol using a
*receiver specified*, session-ephemeral protocol ID. These IDs are analogues of
ephemeral ports.

In this protocol, the stream initiator sends a varint ID specified by the
*receiver* to the receiver.

Format:

```
<multistream/use><multistream/contextual><id(varint)>
```

The ID 0 is reserved for saying "same protocol" on a bidirectional stream. The
receiver of a bidirectional stream can't reuse the same contextual ID that the
initiator used as this contextual ID is relative *to* the receiver. Really, this
last rule *primarily* exists to side-step the TCP simultaneous connect issue.

This protocol has *also* been designed to be hardware friendly:

1. Hardware can compare the first 16 bits of the message against
   `<multistream/use><multistream/contextual>`.
2. It can then route the message based on the contextual ID. The fact that these
   IDs are chosen by the *receiver* means that the receiver can reuse the same
   IDs for all connected peers (reusing the same hardware routing table).
   
### Serial Stream

The `serial-stream` protocol is the simplest possible stream multiplexer.
Unlike other stream multiplexers, `serial-stream` can only multiplex streams
in *serial*. That is, it has to close the current stream to open a new one.

The protocol is:

```
<header (signed 16 bit int)>
<body>
```

Where the header is:

* -2 - Send a reset and return to multistream. All queued data (remote and
  local) should be discarded.
* -1 - Close: Send an EOF and return to multistream.
*  0 - Rest: Ends the reuse protocol, transitioning to a direct stream.
* >0 - Data: The header indicates the length of the data.

We could also use a varint but it's not really worth it. The 16 bit integer
makes implementing this protocol trivial, even in hardware.

Why: This allows us to:

1. Try protocols and fall back on others.
2. More importantly, it allows us to speak a bunch of protocols before setting
   up a stream multiplexer. Specifically, we can use this for
   `multistream/advertise` to send an advertisement as early as possible.

## Upgrade Path

#### Short term

The short-term plan is to first negotiate multistream 1.0 and *then* negotiate
an upgrade. That is, on connect, the *initiator* will send:

```
<len>/multistream/1.0.0\n
<len>/multistream/2.0.0\n
```

As a batch. It will then wait for the other side to respond with either:

```
<len>/multistream/1.0.0\n
<len>na\n
```

in which case it'll continue using multistream 1.0, or:

```
<len>/multistream/1.0.0\n
<len>/multistream/2.0.0\n
```

in which case it'll switch to multistream 2.0.

Importantly: When we switch to multistream 2.0, we'll tag the connection (and
any sub connections) with the multistream version. This way, we never have to do
this again.

## Example

So, that was way too much how and not enough why or WTF? Let's try an example
where,

1. The initiator supports TLS1.3 and SECIO.
2. The receiver only supports TLS1.3.
3. They both support yamux.
4. They both support DHT.
5. secio and tls have multicodecs but yamux and dht don't.

If we're still in the transition period, the initiator would start off by sending:

```
<len>/multistream/1.0\n
<len>/multistream/2.0\n
```

If the receiver DOES NOT support multistream 2.0, it will reply with:

```
<len>/multistream/1.0\n
<len>na\n
```

At this point, the client will fall back on multistream 1.0.

Otherwise, the receiver will send back...

```
<len>/multistream/1.0\n
<len>/multistream/2.0\n
```

...to complete the upgrade.

We're now in multistream 2.0 land. Once we're done with the transition period,
we'll start here to skip a round-trip.

Now that we're using multistream 2.0, the initiator will send, in a single
packet:

```
<multistream/use (multicodec)><serial-stream (multicodec)>                 // use serial-stream to make the stream recoverable
  <len>                                                                    // serial-stream message framing
    <multistream/use (multicodec)><multistream/advertise (multicodec)>     // select advertise protocol
      supported security protocols...                                      // 
  -1                                                                       // return to multistream (EOF)

<multistream/use (multicodec)><serial-stream (multicodec)>                 // open a new serial-stream
  <len>
    <multistream/use (multicodec)><tls (multicodec)>                       // select TLS
      <initial tls packet...>                                              // initiate TLS
```

The receiver will respond with:

```
<multistream/use (multicodec)><serial-stream (multicodec)>                 // respond to serial stream
  <len>
    <multistream/use (multicodec)><multistream/advertise (multicodec)>     // select advertise protocol
      security protocols...
  -1                                                                       // return to multistream (EOF)

<multistream/use (multicodec)><serial-stream (multicodec)>                 // respond to second serial stream
  0                                                                        // transition to a normal stream.
<multistream/use (multicodec)><tls (multicodec)>                           // select TLS
  <response tls packet...>                                                 // complete TLS handshake
```

This:

1. Responds to the advertisement, also advertising available security protocols.
2. Accepts the TLS stream.
3. Finishes the TLS handshake.

If the receiver had *not* supported TLS, it would have reset the serial-stream.
In that case, the initiator would have used the protocols advertised by the
receiver to select an appropriate security protocol.

Finally, the initiator will finish the TLS negotiation, send a advertise packet,
*optimistically* negotiate yamux, and sends the DHT request.

```
  0                                                                        // transition to a normal stream.

<tls client auth...>                                                       // finish TLS

<multistream/use (multicodec)><serial-stream (multicodec)>                 // use serial-stream to make the stream recoverable
  <len>                                                                    // serial-stream message framing
    <multistream/use (multicodec)><multistream/advertise (multicodec)>     // select advertise protocol
      <advertise data>                                                     // comlete advertise information (protocols, etc.)
  -1                                                                       // return to multistream (EOF)

<multistream/use (multicodec)><serial-stream (multicodec)>                 // open a new serial-stream
  <len>
    <multistream/use (multicodec)><multistream/dynamic (multicodec)>       // select multistream/dynamic
        <len>/yamux/1.0.0                                                  // select yamux
      <new yamux stream>                                                   // create the stream
        <multistream/use (multicodec)><multistream/dynamic (multicodec)>   // select multistream/dynamic
            <len>/ipfs/kad/1.0.0                                           // select kad dht 1.0
          <dht request...>                                                 // send the DHT request
```

And the receiver will send:

```
<multistream/use (multicodec)><serial-stream (multicodec)>             // use serial-stream to make the stream recoverable
  <len>                                                                // serial-stream message framing
    <multistream/use (multicodec)><multistream/advertise (multicodec)> // select advertise protocol
      <advertise data>                                                 // comlete advertise information (protocols, etc.)
  -1                                                                   // return to multistream (EOF)

<multistream/use (multicodec)><serial-stream (multicodec)>             // open a new serial-stream
  -1                                                                   // transition to that stream (we speak yamux)

<multistream/use (multicodec)><multistream/dynamic (multicodec)>       // select multistream/dynamic
    <len>/yamux/1.0.0                                                  // select yamux
  <yamux stream 1>                                                     // respond to the new yamux stream
    <multistream/use>                                                  // select multistream/dynamic
      <multistream/dynamic>
        <len>/ipfs/kad/1.0.0                                           // select kad dht
      <dht response...>                                                // send the DHT response
```

Note: Ideally, we'd be able to avoid the optimistic yamux negotiation. However,
to do that, some protocol information will have to be embedded in the TLS
negotiation and exposed through a connection-level `Stat` method.

Alternatively, we could choose to include this information in the advertisement
sent *before* the security transport. However, that has some security
implications.