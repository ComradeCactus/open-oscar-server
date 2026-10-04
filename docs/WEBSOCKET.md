# OSCAR WebSocket bridge

The OSCAR WebSocket bridge lets a browser-based JavaScript client exchange the
existing OSCAR protocol with the server. The protocol data is unchanged: send
and receive FLAP bytes as binary WebSocket messages. The bridge presents the
WebSocket as a byte stream to the existing OSCAR handlers, so it works for
authentication, BOS, and chat connections.

## Configuration

The bridge is disabled unless both WebSocket listener variables are configured.
Add a matching listener and browser-reachable address using the same listener
name (names match case-insensitively):

```dotenv
OSCAR_WEBSOCKET_LISTENERS=LOCAL://0.0.0.0:8090
OSCAR_WEBSOCKET_ADVERTISED_LISTENERS=LOCAL://aim.example.com:443
```

`OSCAR_WEBSOCKET_LISTENERS` is the address the server binds. The advertised
setting is the public host and port sent in OSCAR reconnect fields; it does not
include a scheme or path. Configure the client URL as
`wss://aim.example.com/oscar` (or `ws://<host>:<port>/oscar` for local
development). The example listener accepts plaintext HTTP/WebSocket on port
`8090`; put it behind a TLS-terminating reverse proxy for browser clients.

When a page is served over HTTPS, use `wss://`; browsers block insecure `ws://`
connections from secure pages. Configure the proxy to forward `/oscar` to the
listener and support HTTP/1.1 WebSocket upgrades. Continue to configure
`OSCAR_LISTENERS` and `OSCAR_ADVERTISED_LISTENERS_PLAIN` for native TCP clients.
The WebSocket settings add a transport; they do not replace the TCP listeners.

## JavaScript client transport contract

This is a transport for the existing OSCAR binary protocol, not a JSON API.
Use a WebSocket for each OSCAR connection, set `binaryType` to `"arraybuffer"`,
send binary data, and treat each received binary message as bytes. No
application-level message envelope is added.

WebSocket message boundaries are independent of OSCAR frame boundaries. A
single WebSocket message may contain part of a FLAP frame, one frame, or several
frames. Keep a byte buffer and parse complete FLAP frames from it; do not assume
one WebSocket message equals one FLAP frame. A standard FLAP frame starts with
`0x2a`, then has a one-byte frame type, a two-byte big-endian sequence number,
a two-byte big-endian payload length, and that many payload bytes. Wait for all
six header bytes and the full declared payload before consuming a frame.

The initial exchange on every WebSocket connection is:

1. Open `ws(s)://<advertised-host>:<advertised-port>/oscar`.
2. Wait for the server's FLAP signon frame (frame type `0x01`). It contains
   FLAP version `1` and any server signon TLVs.
3. Send the client's FLAP signon frame (also type `0x01`) with the appropriate
   login or service-connection TLVs for the OSCAR flow.
4. Continue parsing server FLAP frames and sending client FLAP frames as
   binary WebSocket messages.

For authentication, follow the existing OSCAR login flow and credentials
supported by the chosen client. The login response includes an authorization
cookie and a reconnect host. The host is an OSCAR `host:port`, not a URL: use
the same `ws`/`wss` scheme and `/oscar` path to open the next WebSocket, then
send a FLAP signon containing that cookie in the login-cookie TLV (`0x0006`).
Likewise, when a BOS service response redirects to another service (for
example, chat), open another WebSocket to the advertised host using the same
scheme and path, and provide the returned service cookie in the same TLV. These
are OSCAR-level reconnects, not HTTP redirects. Do not try to open a raw TCP
socket from the browser.

The bridge advertises the configured WebSocket host and port for reconnects
from WebSocket connections; it does not redirect a WebSocket client to the
configured raw TCP listener. The regular OSCAR protocol still determines which
service to connect to and what cookie to send. Implement normal FLAP sequence
handling, SNAC/TLV encoding, server-initiated frames, keepalives, and signoff
handling as required by the OSCAR client. See the [OSCAR protocol reference](https://devinsmith.net/backups/OSCAR/)
and the protocol and wire-format implementation in this repository for those
details.

Minimal browser setup:

```js
const socket = new WebSocket("wss://aim.example.com/oscar");
socket.binaryType = "arraybuffer";

socket.addEventListener("open", () => {
  // Send a complete or partial FLAP byte sequence as binary data.
  socket.send(flapBytes); // flapBytes is an ArrayBuffer or typed-array view
});

socket.addEventListener("message", ({ data }) => {
  // Append this ArrayBuffer to the client's FLAP input buffer and parse
  // every complete frame; do not assume one event contains one frame.
  appendAndParseFlapBytes(new Uint8Array(data));
});
```

`appendAndParseFlapBytes` above is client-side protocol parsing, not a bridge
helper provided by this server. A complete OSCAR client must also implement the
authentication and service flows; opening the WebSocket alone does not sign a
user in.
