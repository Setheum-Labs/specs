# Identify v1.0.0

The identify protocol is used to exchange basic information with other peers
in the network, including addresses, public keys, and capabilities.

There are three variations of the identify protocol, `identify`,
`identify/push`, and `identify/delta`.

### `identify`

The `identify` protocol has the protocol id `/ipfs/id/1.0.0`, and it is used
to query remote peers for their information.

The protocol works by opening a stream to the remote peer you want to query, using
`/ipfs/id/1.0.0` as the protocol id string. The peer being identified responds by returning
an `Identify` message and closes the stream.

### `identify/push`

The `identify/push` protocol has the protocol id `/ipfs/id/push/1.0.0`, and it is used
to inform known peers about changes that occur at runtime.

When a peer's basic information changes, for example, because they've obtained a new
public listen address, they can use `identify/push` to inform others about the new
information.

The push variant works by opening a stream to each remote peer you want to update, using
`/ipfs/id/push/1.0.0` as the protocol id string. When the remote peer accepts the stream,
the local peer will send an `Identify` message and close the stream.

Upon recieving the pushed `Identify` message, the remote peer should update their local
metadata repository with the information from the message. Note that
`identify/push` sends a complete `Identify` message containing the full state at
the time of broadcast. To send partial updates instead, peers may use the
`identify/delta` protocol described below.


### `identify/delta`

The `identify/delta` protocol has the protocol id `/p2p/id/delta/1.0.0`, and
like `identify/push`, it is used to notify known peers about runtime changes. In
contrast to `identify/push`, the delta protocol does not send the full
`Identify` message state. Instead, it sends only a small message containing
protocols which were added or removed. This allows more efficient updates when a
peer dynamically changes its capabilites at runtime.

When a peer adds or removes support for one or more protocols, it may open a
stream to each remote peer that it wants to update using `/p2p/id/delta/1.0.0`
as the protocol id. When the remote peer accepts the stream, the local peer will
send an `Identify` message with only the `delta` field set. The value of the
`delta` field is a `Delta` message, which will include the protocol ids of added
or removed protocols in the `added_protocols` and `rm_protocols` fields,
respectively.

## The Identify Message

```protobuf
message Identify {
  optional string protocolVersion = 5;
  optional string agentVersion = 6;
  optional bytes publicKey = 1;
  repeated bytes listenAddrs = 2;
  optional bytes observedAddr = 4;
  repeated string protocols = 3;
  optional Delta delta = 7;
}

message Delta {
  repeated string added_protocols = 1;
  repeated string rm_protocols = 2;
}
```

The `delta` field of the `Identify` message is only set when sending partial
updates as part of the [`identify/delta`](#identify-delta) protocol. When the
`delta` field is set, it MUST be the only field present in the `Identify`
message, with all other fields unset.

### protocolVersion

The protocol version identifies the family of protocols used by the peer.
The current protocol version is `ipfs/0.1.0`; if the protocol major or minor
version does not match the protocol used by the initiating peer, then the connection
is considered unusable and the peer must close the connection.

### agentVersion

This is a free-form string, identifying the implementation of the peer.
The usual format is `agent-name/version`, where `agent-name` is
the name of the program or library and `version` is its semantic version.

### publicKey

This is the public key of the peer, marshalled in binary form as specicfied
in [peer-ids](../peer-ids).


### listenAddrs

These are the addresses on which the peer is listening as multi-addresses.

### observedAddr

This is the connection source address of the stream initiating peer as observed by the peer
being identified; it is a multi-address. The initiator can use this address to infer
the existence of a NAT and its public address.

For example, in the case of a TCP/IP transport the observed addresses will be of the form
`/ip4/x.x.x.x/tcp/xx`. In the case of a circuit relay connection, the observed address will
be of the form `/p2p/QmRelay/p2p-circuit`. In the case of onion transport, there is no
observable source address.

### protocols

This is a list of protocols supported by the peer.