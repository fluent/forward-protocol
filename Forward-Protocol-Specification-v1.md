# Fluentd Forward Protocol Specification (v1)

This is a protocol specification for Fluentd `forward` input/output plugins. This protocol is also used by fluent-logger software, and many other software in ecosystem (e.g., Docker logging driver for Fluentd).

This protocol version is v1. This spec is supported by Fluentd `v0.14` (`v0.14.5` and later), and `v1`. [Protocol version v0](https://github.com/fluent/fluentd/wiki/Forward-Protocol-Specification-v0) is supported by `v0.10` and `v0.12`, and protocol version `v1` is compatible with `v0` (All implementations which supports `v1` also supports `v0`).

## Changes

* September 6, 2016: the first release of this spec.
* September 12, 2016: fix inconsistent specification about EventTime.

### Major Changes Between v0 and v1

* Handshake phase added
* "compress" option and CompressedPackedForward mode added
* SSL/TLS transport layer handling added

## Abstract

This specification describes the fluentd forward protocol, which is used to authenticate/authorize clients/servers, and to transport events from hosts to hosts over network.

##  Terminology

The keywords "MUST", "MUST NOT", "SHOULD", "SHOULD NOT" and "MAY" in this document are to be interpreted as described in [RFC 2119](https://tools.ietf.org/html/rfc2119). The following terms are also used:

- event: a virtual set of tag, time and record. tag MAY be shared by 2 or more events in actual representation if these events have same tag.
- client: an endpoint that sends events. `out_forward` plugin is the reference client implementation shipped with the fluentd distribution.
- server: an endpoint that receives events. `in_forward` plugin is the reference server implementation shipped with the fluentd distribution.
- connection: a TCP connection between two endpoints.
- [msgpack](https://github.com/msgpack/msgpack/blob/master/spec.md): a light weight binary representation of serialized objects.

## Protocol Specification

### Heartbeat Message

- Client MAY send UDP packets to the same port of connection, to check existence of Servers.
- Server SHOULD listen the same port of data transport connection in UDP, and SHOULD respond to source address when heartbeat message arrived.

UDP heartbeat message SHOULD be a byte of `0x00` in both of Client and Server.

### Network Transport Protocol

- Server MUST listen any TCP port for data transport connection, and MAY be able to establish SSL/TLS session on that connection.
- Client MUST be able to connect to servers via TCP, and MAY connect to servers using SSL/TLS.

### Connection Phases

Connections have two phases on each sessions.

1. **Handshake**(optional): In this phase, servers and clients send messages each other to establish connections after authentication and authorization. It can be skipped when server doesn't require it.
1. **Transport**: In this phase, clients send events to servers, and servers send response messages if client require it.

Once handshake phase completes, clients can use that connection in transport phase until disconnection.

Handshake phase is optional. If server doesn't require authentication/authorization, connection becomes transport phase immediately, and client can send events without any preparation.

### Fluentd Forward Client

- Client sends a msgpack `array` which contains handshake messages, or one or more events to the server through connections.
- Client MUST respond to the server for authentication/authorization messages in handshake phase by sending `PING` described below.
- Client MUST disconnect the connection if authentication operation fails.
- Client MUST choose a carrier mode for events from four: `Message`, `Forward`, `PackedForward` and `CompressedPackedForward` described below.
- Client MAY send a msgpack `nil` value for heartbeat health check usage without any event record payload.
- Client MAY send multiple msgpack `array`s of events in transport phase on a single connection, continuously.

### Fluentd Forward Server

- Server MUST receive msgpack `array`s which contains handshake messages, or one or more events, from the client through connections.
- Server MAY disconnect the connection when it has network/domain ACL (access control list) and the client is not allowed by ACL.
- Server MAY send `HELO` and `PONG` messages in handshake phase to the client for authentication and authorization.
- Server MUST disconnect the connection if authentication/authorization operation fails.
- Server MUST detect the carrier mode by inspecting the second element of the array.
- Server SHOULD ignore any request value other than `array` format.
- Note: In addition to the four carrier modes, `in_forward` plugin also accepts JSON representation of a single event for convenience. It detect it by the first byte of the request. It is unavailable with authentication/authorization.

## Handshake Messages

Handshake messages are to establish connections with assurance for:

- Server and client share the same key (`shared_key` string) for the connection.
- Client has correct username and password for the connection (if required by server).

All handshake messages are msgpack `array`s which contains values described below. The first element of these messages MUST show the types of messages.

### Authentication and Authorization

- Authentication: Both server and client authenticate the peer node by having same `shared_key` string. (required)
- Authorization: Server authorize client by having valid `username` and `password` pair. (optional)

Some values are used to implement authentication/authorization, and these are shared by server and client in handshake phase.

- `nonce` is generated by server, sent in `HELO` and used to generate `shared_key_hexdigest` in `PING` and `PONG`
- `shared_key_salt` is generated by client, send in `PING` and used to generate `shared_key_hexdigest` in `PING` and `PONG`
- `auth` salt is generated by server, sent in `HELO` and used to generate `password` (hexdigest) in `PING`

### HELO

HELO message is sent from server to client.

- `type` is a string, equals to `HELO`.
- `options` is key-value pairs to tell options enabled in the connection to the client.

name | Ruby type | msgpack format | content
--- | --- | --- | ---
type | String | str | "HELO"
options | Hash | map | described below

HELO `options` has some keys and values described below.

name | Ruby type | msgpack format | content
--- | --- | --- | ---
nonce | String | str\|bin | a binary string of nonce to generate digest in PING message
auth | String | str\|bin | a binary string of salt to generate digest for user authentication (authentication not required if it's empty)
keepalive | true\|false | bool | (optional) server disallow long life connections if false (default: true)

```json
[
  "HELO",
  {
    "nonce": "xxxxxxx",
    "auth": "yyyyyyyy",
    "keepalive": true
  }
]
```

NOTE: HELO is inspired by the command to start SMTP sessions.

### PING

PING message is sent from client to server after HELO.

- `type` is a string, equals to `PING`.
- `client_hostname` is a string represents the hostname of client.
- `shared_key_salt` is a binary string of salt to generate `shared_key_digest`.
- `shared_key_hexdigest` is a hex string of SHA512 digest of `shared_key_salt`, `client_hostname`, `nonce` and `share_key`.
- `username` is a username string for the connection, or empty string if authentication is not required.
- `password` is a hex string of SHA512 digest of `auth` salt, `username` and raw password string, or empty string if authentication is not required.

name | Ruby type | msgpack format | content
--- | --- | --- | ---
type | String | str | "PING"
client_hostname | String | str | FQDN hostname of client
shared_key_salt | String | str\|bin | a salt binary(String)
shared_key_hexdigest | String | str | sha512_hex(shared_key_salt + client_hostname + nonce + shared_key)
username | String | str | a username or empty string
password | String | str | sha512_hex(auth_salt + username + raw_password) or empty string

```json
[
  "PING",
  "client.hostname.example.com",
  "xxxxxxx_shared_key_salt",
  "deadbeef_shared_key_hex_digest",
  "a_user_name",
  "c0ffee_password_hex_digest"
]
```

### PONG

PONG message is sent from server to client after PING.

- `type` is a string, equals to `PONG`.
- `auth_result` is a boolean value to represent result of authentication/authorization (true means success).
- `reason` is a strong to show the reason why client is disallowed, or empty string if `auth_result` is true.
- `server_hostname` is a string represents the hostname of server.
- `shared_key_hexdigest` is a hex string of SHA512 digest of `shared_key_salt`, `server_hostname`, `nonce` and `shared_key`.

name | Ruby type | msgpack format | content
--- | --- | --- | ---
type | String | str | "PONG"
auth_result | true\|false | bool | auth succeeded or not
reason | String | str | reason why auth failed, or ''
server_hostname | String | str | FQDN hostname of server
shared_key_hexdigest | String | str | sha512_hex(shared_key_salt + server_hostname + nonce + shared_key)

```json
[
  "PONG",
  true,
  "",
  "server.hostname.example.com",
  "deadbeef_shared_key_hex_digest"
]
```

If `auth_result` is false, server will disconnect the connection. Otherwise, client can use the connection in transport phase.

## Event Modes

Once the connection becomes transport phase, client can send events to servers, in one event mode of modes described below.

### Message Modes

It carries just a event.

- `tag` is a string separated with '.' (e.g. myapp.access) to categorize events.
- `time` is a EventTime value (described below), or a number of seconds since Unix epoch.
- `record` is key-value pairs of the event record.
- `option` is optional key-value pairs, to bring data to control servers' behavior.

name | Ruby type | msgpack format | content
--- | --- | --- | ---
tag | String |str | tag name
time | EventTime\|Integer | ext\|int | time from Unix epoch in nanosecond precision(EventTime), or in second precision (Integer)
record | Hash | map | pairs of keys(String) and values(Object)
option | Hash | map | option (optional)

```json
[
  "tag.name",
  1441588984,
  {"message": "bar"},
  {"option": "optional"}
]
```

NOTE: EventTime is formatted into Integer in json format.

### Forward Mode

It carries a series of events as a msgpack `array` on a single request.

name | Ruby type | msgpack format | content
--- | --- | --- | ---
tag | String | str | tag name
entries | MultiEventStream | array | list of Entry
option | Hash | map | option (optional)

```json
[
  "tag.name",
  [
    [1441588984, {"message": "foo"}],
    [1441588985, {"message": "bar"}],
    [1441588986, {"message": "baz"}]
  ],
  {"option": "optional"}
]
```

### PackedForward Mode

It carries a series of events as a msgpack binary on a single request.

- `entries` is a binary chunk of `MessagePackEventStream` which contains multiple raw msgpack representations of `Entry`.
- Client SHOULD send a `MessagePackEventStream` as msgpack `bin` format as its binary representation.
- Client MAY send a `MessagePackEventStream` as msgpack `str` format for compatibility reasons.
- Server MUST accept both formats of `bin` and `str`.
- Server MAY decode individual events on demand but MAY NOT do right after request arrival. It means it MAY costs less, compared to `Forward` mode, when decoding is not needed by any plugins.  
- Note: `out_forward` plugin sends events by the `PackedForward` mode. It encloses event records with msgpack `str` format instead of `bin` format for a backward compatibility reason.

name | Ruby type | msgpack format | content
--- | --- | --- | ---
tag | String | str | tag name
entries | MessagePackEventStream | bin\|str | msgpack stream of Entry
option | Hash | map | option (optional)

```json
[
  "tag.name",
  "<<MessagePackEventStream>>",
  {"option": "optional"}
]
```

Note for v2 protocol: PackedForward messages should be sent in `bin` format.

### CompressedPackedForward Mode

It carries a series of events as a msgpack binary, compressed by gzip, on a single request. The supported compression algorithm is only gzip.

- `entries` is a gzipped binary chunk of `MessagePackEventStream`, which MAY be a concatenated binary of multiple gzip binary strings.
- Client MUST send an option with `compressed` key with the value `gzip`.
- Client MUST send a gzipped chunk as msgpack `bin` format.
- Server MUST accept `bin` format.
- Server MAY decompress and decode individual events on demand but MAY NOT do right after request arrival. It means it MAY costs less, compared to `Forward` mode, when decoding is not needed by any plugins.  

name | Ruby type | msgpack format | content
--- | --- | --- | ---
tag | String | str | tag name
entries | CompressedMessagePackEventStream | bin | gzipped msgpack stream of Entry
option | Hash | map | option including key "compressed" (required)

```json
[
  "tag.name",
  "<<CompressedMessagePackEventStream>>",
  {"compressed": "gzip"}
]
```

### Entry

Entry is an `array` representation of pairs of time and record, used in `Forward`, `PackedForward` and `CompressedPackedForward` mode.

name | Ruby type | msgpack format | content
--- | --- | --- | ---
time | EventTime\|Integer | ext\|int | time from Unix epoch in nanosecond precision(EventTime), or in second precision (Integer)
record | Hash | map | pairs of keys(String) and values(Object)

### Option

It carries an optional meta data for the request.

- Client MAY send key-value pairs of options.
- Server MAY just ignore any options given.
- `size`: Clients MAY send the `size` option to show the number of event records in an entries by an integer as a value. Server can know the number of events without unpacking entries (especially for PackedForward and CompressedPackedForward mode).
- `chunk`: Clients MAY send the `chunk` option to confirm the server receives event records. The value is a string of Base64 representation of 128 bits `unique_id` which is an ID of a set of events.
- `compressed`: Clients MUST send the `compressed` option with value `gzip` to tell servers that entries is `CompressedPackedForward`. Other values will be ignored.

```json
{"chunk": "p8n9gmxTQVC8/nh2wlKKeQ==", "size": 4097}
{"chunk": "p8n9gmxTQVC8/nh2wlKKeQ==", "size": 1023, "compressed": "gzip"}
```

### Response

- Server SHOULD close the connection silently with no response when the `chunk` option is not sent.
- `ack`: Server MUST respond `ack` when the `chunk` option is sent by client. The `ack` response value MUST be the same value given by `chunk` option from client. Client SHOULD retry to send events later when the request has a `chunk` but the response has no `ack`.

```json
{"ack": "p8n9gmxTQVC8/nh2wlKKeQ=="}
```

### EventTime Ext Format

`EventTime` uses msgpack extension format of type 0 to carry nanosecond precision of `time`.

- Client MAY send `EventTime` instead of plain integer representation of second since unix epoch.
- Server SHOULD accept both formats of integer and EventTime.
- Binary representation of `EventTime` may be `fixext` or `ext`(with length 8).

```txt
+-------+----+----+----+----+----+----+----+----+----+
|     1 |  2 |  3 |  4 |  5 |  6 |  7 |  8 |  9 | 10 |
+-------+----+----+----+----+----+----+----+----+----+
|    D7 | 00 | second from epoch |     nanosecond    |
+-------+----+----+----+----+----+----+----+----+----+
|fixext8|type| 32bits integer BE | 32bits integer BE |
+-------+----+----+----+----+----+----+----+----+----+

+--------+----+----+----+----+----+----+----+----+----+----+
|      1 |  2 |  3 |  4 |  5 |  6 |  7 |  8 |  9 | 10 | 11 |
+--------+----+----+----+----+----+----+----+----+----+----+
|     C7 | 08 | 00 | second from epoch |     nanosecond    |
+--------+----+----+----+----+----+----+----+----+----+----+
|   ext8 | len|type| 32bits integer BE | 32bits integer BE |
+--------+----+----+----+----+----+----+----+----+----+----+
```

### Grammar

- `Name?` means that `Name` is optional.
- `Name*` means that `Name` can occur zero or more times.
- `<<Name>>` means binary msgpack representation of `Name`.
- `[ A, B, C ]` means an array.
- `nil`, `string`, `integer` and `object` means as it is.

```txt
Connection ::= <<Request>>*

Request ::= Message | Forward | PackedForward | nil

Message ::= [ Tag, Time, Record, Option? ]

Forward ::= [ Tag, MultiEventStream, Option? ]

MultiEventStream ::= [ Event* ]

PackedForward ::= [ Tag, MessagePackEventStream, Option? ]

MessagePackEventStream ::= <<Event>>*

Event ::= [ Time, Record ]

Tag ::= string

Time ::= integer | EventTime

Record ::= object

Option ::= object
```