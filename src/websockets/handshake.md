# The Upgrade Handshake

WebSocket connections begin as HTTP requests, then "upgrade" to the WebSocket protocol. This allows WebSockets to work through HTTP infrastructure (proxies, load balancers) while establishing a different communication pattern.

## Client Request

```http
GET /chat HTTP/1.1
Host: server.example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
Origin: http://example.com
Sec-WebSocket-Protocol: chat, superchat
```

### Required Headers

| Header | Purpose |
|--------|---------|
| `Upgrade: websocket` | Request protocol upgrade |
| `Connection: Upgrade` | Indicates upgrade requested |
| `Sec-WebSocket-Key` | Random base64 value for handshake validation |
| `Sec-WebSocket-Version: 13` | WebSocket protocol version |

### Optional Headers

| Header | Purpose |
|--------|---------|
| `Origin` | Where request originates (CORS) |
| `Sec-WebSocket-Protocol` | Subprotocol preferences (application-defined) |
| `Sec-WebSocket-Extensions` | Extension negotiation (e.g., compression) |

## Server Response

```http
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
Sec-WebSocket-Protocol: chat
```

### Sec-WebSocket-Accept Calculation

Server proves it received the client's key:

```
1. Take client's Sec-WebSocket-Key:
   "dGhlIHNhbXBsZSBub25jZQ=="

2. Append magic GUID:
   "dGhlIHNhbXBsZSBub25jZQ==" + "258EAFA5-E914-47DA-95CA-C5AB0DC85B11"

3. SHA-1 hash the result

4. Base64 encode the hash:
   "s3pPLMBiTxaQ9kYGzzhZRbK+xOo="

Client verifies this matches expected value.
Prevents accidental connections or caching issues.
```

## After the Handshake

```
HTTP/1.1 101 Switching Protocols
...

─── HTTP ENDS HERE ───

┌─────────────────────────────────────────┐
│     WebSocket Frames (Binary)           │
│                                         │
│  [Frame Header][Payload]                │
│  [Frame Header][Payload]                │
│  ...                                    │
└─────────────────────────────────────────┘

Same TCP connection, different protocol.
No more HTTP until connection closes.
```

## Handshake Implementation

### JavaScript (Browser)

```javascript
const ws = new WebSocket('wss://server.example.com/chat');

ws.onopen = () => {
  console.log('Connected!');
  ws.send('Hello Server!');
};

ws.onmessage = (event) => {
  console.log('Received:', event.data);
};

ws.onclose = (event) => {
  console.log('Closed:', event.code, event.reason);
};

ws.onerror = (error) => {
  console.error('Error:', error);
};
```

### Python Server (websockets library)

```python
import asyncio
import websockets

async def handler(websocket, path):
    async for message in websocket:
        print(f"Received: {message}")
        await websocket.send(f"Echo: {message}")

async def main():
    async with websockets.serve(handler, "localhost", 8765):
        await asyncio.Future()  # Run forever

asyncio.run(main())
```

### Node.js Server (ws library)

```javascript
const WebSocket = require('ws');
const wss = new WebSocket.Server({ port: 8080 });

wss.on('connection', (ws) => {
  console.log('Client connected');

  ws.on('message', (message) => {
    console.log('Received:', message.toString());
    ws.send(`Echo: ${message}`);
  });

  ws.on('close', () => {
    console.log('Client disconnected');
  });
});
```

## Subprotocols

Subprotocols define application-level meaning:

```
Client: Sec-WebSocket-Protocol: graphql, json, protobuf
Server: Sec-WebSocket-Protocol: graphql

Agreement: Use GraphQL over WebSocket.

Common subprotocols:
  - graphql-ws (GraphQL subscriptions)
  - mqtt (IoT messaging)
  - wamp (Web Application Messaging)
  - soap (legacy)
```

## Extensions

Extensions modify the protocol (typically for compression):

```
Client: Sec-WebSocket-Extensions: permessage-deflate
Server: Sec-WebSocket-Extensions: permessage-deflate

permessage-deflate:
  - Compresses message payloads
  - Significant bandwidth savings for text
  - Supported by most implementations
```

## Error Cases

```
Server doesn't support WebSocket:
  HTTP/1.1 400 Bad Request
  (or 404, or no Upgrade response)

Wrong Sec-WebSocket-Accept:
  Client: Abort connection
  Prevents man-in-the-middle returning wrong accept

Origin not allowed (CORS-like):
  HTTP/1.1 403 Forbidden
  Server rejects based on Origin header
```

## Secure WebSockets (WSS)

```
ws://  - Unencrypted WebSocket (like HTTP)
wss:// - Encrypted WebSocket over TLS (like HTTPS)

Process for wss://:
  1. TCP connection
  2. TLS handshake (certificate validation)
  3. HTTP Upgrade request (encrypted)
  4. WebSocket frames (encrypted)

Always use wss:// in production.
```

## Summary

The WebSocket handshake:

1. Client sends HTTP GET with `Upgrade: websocket`
2. Server validates and responds `101 Switching Protocols`
3. Server sends `Sec-WebSocket-Accept` derived from client's key
4. Connection becomes bidirectional WebSocket

After upgrade, it's no longer HTTP—just WebSocket frames on TCP.
