# Broker

WebSocket based publish / subscribe broker inpired by MQTT using
Node-RED

## About

Implement a publish / subscribe message broker in Node-RED with support
for message persistence, without requiring any external servers. This
message broker supports the following features:

- Broadcast each message received from any client to all connected
  clients
- Support `msg.retain` in a manner inspired by MQTT
- Use only protocols and services native to Node-RED

### Similarities to MQTT

- `msg.retain` can be used to store messages on the broker
- `msg.topic` is used as the key when retaining messages

### Differences from MQTT

- All message properties are preserved and passed through, not
  just `msg.topic` and `msg.payload`
- Clients must group and filter messages for themselves, i.e. there
  is no mechanism built into the broker for subscribing to specific
  topics

## Publish / Subscribe Server

The broker is implemented on a server instance of Node-RED using a
pair of WebSocket input and output nodes configued to "listen" on
a particular URI. Each message received from the `websocket in` node
on the server is passed through more-or-less as-is to the
`websocket out` node listening on the same URI:

1. Delete `msg._session` so that the message will be broadcast to all
  clients
2. Process `msg.retain` using a `function` node
    - If `msg.retain` is `true` in the incoming message, handle
      retention as described below
    - Otherwise, just pass the message through
3. Set `msg.retain` to `false` for all messages that arrive from the
  `websocket in` node before sending them to the `websocket out` node

In addition, this flow uses a `status` node to detect when clients
connect to the `websocket out` node. Each client is sent copies of
all currently retained messages when it connects. All such retained
messages are exact copies of the original messages previously received
from the `websocket in` node, including `msg.retain` being set to
`true`.

### Message Retention

The message broker requires that a filesystem-backed context store
exists named `"file"`. This requires a modification to _settings.js_:

```
contextStorage: {
    default: "memoryOnly",
    memoryOnly: { module: 'memory' },
    file: { module: 'localfilesystem' }
},
```

All references to flow context described below use the `"file"`
context store.

When a message arrives through the `websocket in` node with
`msg.retain` set to `true`, a `function` node:

1. Signals an error and returns `null` if `msg.topic` is not valid
2. If `msg.topic` is valid, the function then checks the value of
  `msg.payload`
    - If `msg.payload` is the empty string, any existing value
      associated with `msg.topic` is deleted from the object
      named `retained` in flow context
    - Otherwise, a clone of the complete incoming `msg` (not just
     `msg.payload`) is stored as the value of `msg.topic` in
     in the `retained` object in flow context

The server flow uses a `status` node to detect when clients connect to
the `websocket out` node. Each time a client connects the `status` node
triggers another `function` node to send all currently retained messages
to the newly connected client. Note that `msg._session` is copied from
the `status` event to each retained message so that only the newly
connected client will receive it.

## Publish / Subscribe Client

A publish / subscribe client is implemented simply as a pair of the
WebSocket nodes configured to "connect to" the URL corresponding to
the broker's URL. For example, if the server flow as described above
is running on a Node-RED instance whose IP address is `192.168.0.42`
using the default port, `1880` and the server "listener" URI is `/broker`
then a client would have its WebSocket nodes configured to "connect to"
`ws://192.168.0.42:1880/broker`.

Publishing a message is simply sending it to a `websocket out`
configured to connect to the server's URL.

Subscribing to messages is simply processing any input received using a
`webscoket in` node configured to connect to the server's URL.

Unlike MQTT nodes, there is no built-in mechanism to subscribe to
specific topics. Any `websocket in` node on any client will receive
all messages broadcast by all clients. To emulate the MQTT feature of
subscribing to a particular topic, use `switch` nodes with regular
expressions or some similar mechanism on the client side. Note that
all properties are retained and passed through by the broker, so such
filter rules do not have to restrict themselves only to `msg.topic`
and clients can access all properties of the published message, not just
`msg.payload`. The only exceptions is `msg.retain` which is modified by
the server as described above. As with MQTT, the state of `msg.retain`
on the client side can be used to distinguish new messages from ones
that were retained before a given client connected and so may have
already been acted upon or be out of date in some other way.