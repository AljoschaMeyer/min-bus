# Min-Bus

A somewhat minimal system for coordinating inter-process communication.

## Abstractions

Min-Bus is a system that allows different processes - called *plugins* - to discover and talk to each other. To do so, it provides a *server*, where plugins can connect to. When connecting to the server, a plugin must specify a *name* (arbitrary byte string not longer than 255 bytes), and a *version* (unsigned 16 bit integer). Connecting succeeds if no other plugin with the same name is currently connected. Versions signify non-breaking changes to the API exposed by a plugin. If a plugin performs breaking changes to its API, it needs to change its name.

Once the connection is established, the plugin can perform the following actions:

- query the currently connected plugins (name + version)
  - this opens a stream where the server first sends the list of currently connected plugins, and then sends a notification whenever a plugin connects or disconnects
  - the plugin can cancel the stream at any time
- be notified of other plugins that what to connect to it. It can either reject the connection, or allow it.
- ask to be connected to another plugin (by name). If the other plugin accepts, both of them are given
  - a bidirectional, byte-oriented communication channel to the other plugin
    - with an explicit, credit-based backpressure mechanism
    - either plugin is allowed to close/cancel it
    - plugins can use any protocol they like to communicate
  - a bidirectional channel over which the plugin can send heartbeat pings and receive heartbeat pongs
  - a bidirectional channel over which the plugin receives hearbeat pings and can send back heartbeat pongs
- disconnect

## Implementation

A server implementation is free to choose any mechanism for plugins to connect to, as long as there is a [bpmux](https://github.com/AljoschaMeyer/bpmux) implementation that runs over that mechanism. For example, the server could listen for tcp connections on a specific port, or it could use unix domain sockets. A server is allowed to expose any number of such mechanisms (e.g. both a local unix domain socket and a tcp socket with a public address), all plugins can talk to each other regardless of how they connected to the server. For public-facing ports, it is recommended to enforce authentication and to encrypt all data.

After connection establishment, all communication happens via bpmux. The server should should always supply the plugin with enough credit so that it can perform queries. The server should also respond to any heartbeat pings. The server may send heartbeat pings to a plugin and disconnect it if it does not react after a well-documented, implementation specific time.

### Registration Message

Initially, the server does nothing but wait for the plugin to send its registration message. This is a message that consists of an unsigned 8 bit integer, followed by that many bytes as the name of the plugin, followed by a uint16 version of the plugin. If there is already a plugin of that name, the server simply terminates the connection. Else it notifies all subscribed plugins that the new plugin has been registered and goes into the *connected state*.

After sending the registration message, a plugin can send any of the following to the server:

### Diconnection Message

The plugin sends a message with a zero-length payload. This is not strictly necessary, the plugin can instead directly terminate the underlying connection. But in case of connection types where this can not be directly detected by the server, it is polite to send this message prior to disconnecting.

Upon receiving this message, the server does all the cleanup necessary for disconnected plugins (see section "Server Bahavior Upon Plugin Disconnection")

### Connection Query

The plugin sends a request with a zero-length payload. The server sends a response containing the repeated `<length_uint8><name><version_uint16>` tuples of all currently connected plugins, except for the requesting plugin itself.

### Connection Stream

The plugin opens a stream with a zero-length payload. The server immediately sends the set of currently connected plugins (except for the plugin itself) as a single message to the stream, containing repeated `<length_uint8><name><version_uint16>` tuples. Afterwards, whenever a plugin connects, the server sends a message containing the byte `0000_0000` followed by the `<length_uint8><name><version_uint16>` of the plugin to the stream. Whenever a plugin disconnects, the server sends a message containing the byte `0000_0001` followed by the `<length_uint8><name><version_uint16>` of the plugin to the stream,.
If a plugin opens a `connection stream` while it still has another one open, the server must immediately cancel the new one (with a zero-length payload).

### Connect to Plugin

For plugin `a` to establish a connection to plugin `b`, plugin `a` opens a duplex with a payload consisting of the byte `0000_0000` followed by `<length_uint8><name>` of plugin `b`. If `a` has already established a connection to `b` that hasn't been closed/canceled in both direction, the server must directly cancel and close the duplex with a zero-length payload. Else, the server must allocate enough memory for two 64 bit integers and two buffers of bytes for caching data, each of size at least one byte. If it fails to do so, it must both close and cancel the duplex with the byte `0000_0000` as the payload. If memory allocation suceeded, the server then opens a duplex to plugin `b`, with a payload consisting of the byte `0000_0000` followed by the `<length_uint8><name><version_uint16>` of plugin `a`.

All pings sent by `a` get relayed along that channel. All credit given by `a` is buffered into one of the 64 bit integer the server allocated. Likewise, pings by `b` are forwarded and credit given by `b` is buffered as well.

Plugin `b` can deny the connection by closing and cancelling the duplex with a zero-length payload. In that case, the server must close and cancel the duplex with `a` with the byte `0000_0001` as the payload. Plugin `b` can accept the connection by sending a zero-length message down the duplex. In that case, the server must send a zero-length message down the duplex it shared with `a`. It then sends all buffered credit to `a` and `b` respectively. The logical connection between `a` and `b` is now established, consisting of the two duplexes.

After the connection has been established, the server forwards all messages, hearbeat pings, heartbeat pongs, and credit along the duplexes. To the plugins, this looks like a direct connection between them. There is a situation however where the server may choose to not directly relay all credit. Imagine a setting where there is a very fast connection between `a` and the server, but a very slow connection between the server and `b`. Furthermore assume that `b` gives a lot of credit to `a`. `a` sends a lot of data, but the server can not relay it fast enough to `b`, so it has to buffer data. In this setting, `b` would effectively determine how much data the server has to buffer. To prevent this, the server can simply put an upper limit to the credit it hands to `a`, and store any excess credit given by `b` in one of the 64 bit integers. When more data could be sent from the server to `b`, it can then send more credit to `a`, removing it from the 64 bit integer. The server should use a scheme that is more efficient than sending credit whenever data could be sent, as that might result in a lot of credit data being transmitted.

If one of the plugins cancels/closes parts of the connection, the server prepends a byte `0000_0000` to the payload before relaying it.

### Server Bahavior Upon Plugin Disconnection
When plugin `foo` sends a disconnection message, or the connection to the server is closed or errors, the server must perform the following cleanup work (only once per plugin):

- notify all connection streams
- for any connections between `foo` and other plugins, cancel and close the duplex between server and the remaining plugin, both with the byte `0000_0001` as the payload.

### Handling Unexpected Messages
If the plugin sends any data to the server that does not fit into one of the flows described above, the server must do the following:

- if it is a message, ignore it
- if it is a request, cancel it with a zero-length payload
- if it is a sink, cancel it with a zero-length payload
- if it is a stream, close it with a zero-length payload
- if it is a duplex, cancel and close it with a zero-length payload

This allows backwards-compatible protocol extensions. Clients can use these to check whether some feature that might be added to min-bus is unsupported by the server, without disrupting the connection.
