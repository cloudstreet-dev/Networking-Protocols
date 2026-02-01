# Full-Duplex Communication

After the handshake, WebSocket communication happens through frames—small packets that can carry text, binary data, or control messages.

## Frame Format

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
├─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┤
│F│R│R│R│  Opcode │M│         Payload Length                   │
│I│S│S│S│  (4)    │A│             (7)                          │
│N│V│V│V│         │S│                                          │
│ │1│2│3│         │K│                                          │
├─┴─┴─┴─┴─────────┴─┴───────────────────────────────────────────┤
│     Extended payload length (16/64 bits, if needed)          │
├───────────────────────────────────────────────────────────────┤
│     Masking key (32 bits, if MASK=1)                         │
├───────────────────────────────────────────────────────────────┤
│     Payload Data                                             │
└───────────────────────────────────────────────────────────────┘

Minimum frame: 2 bytes (header only)
Typical small message: 6-8 bytes overhead
```

### Frame Fields

| Field | Bits | Description |
|-------|------|-------------|
| FIN | 1 | Final fragment of message |
| RSV1-3 | 3 | Reserved for extensions |
| Opcode | 4 | Frame type |
| MASK | 1 | Payload is masked (required from client) |
| Payload Length | 7+ | Size of payload |

### Opcodes

```
0x0  Continuation   Part of fragmented message
0x1  Text          UTF-8 text data
0x2  Binary        Binary data
0x8  Close         Connection close request
0x9  Ping          Heartbeat request
0xA  Pong          Heartbeat response
```

## Message Types

### Text Messages

```javascript
// Send
ws.send("Hello, World!");

// Frame created:
// FIN=1, Opcode=0x1 (text), MASK=1, Payload="Hello, World!"

// Receive
ws.onmessage = (event) => {
  console.log(event.data);  // "Hello, World!" (string)
};
```

### Binary Messages

```javascript
// Send ArrayBuffer
const buffer = new ArrayBuffer(8);
const view = new DataView(buffer);
view.setFloat64(0, 3.14159);
ws.send(buffer);

// Send Blob
const blob = new Blob(['Binary data'], {type: 'application/octet-stream'});
ws.send(blob);

// Receive
ws.binaryType = 'arraybuffer';  // or 'blob'
ws.onmessage = (event) => {
  const data = event.data;  // ArrayBuffer or Blob
};
```

## Control Frames

### Ping/Pong (Heartbeat)

```
Ping: "Are you still there?"
Pong: "Yes, I'm here."

Server sends:  Ping (opcode 0x9)
Client sends:  Pong (opcode 0xA) with same payload

Purpose:
  - Detect dead connections
  - Keep NAT mappings alive
  - Measure latency

Client browser handles Pong automatically.
```

### Close Frame

```
Graceful shutdown:

1. Initiator sends Close frame
   - Opcode 0x8
   - Optional: status code (2 bytes) + reason (text)

2. Recipient sends Close frame back

3. TCP connection closed

Status codes:
  1000  Normal closure
  1001  Endpoint going away
  1002  Protocol error
  1003  Unsupported data type
  1006  Abnormal closure (no close frame)
  1011  Server error
```

## Masking

Client-to-server frames must be masked:

```
Why masking?
  Cache poisoning attack prevention.
  Proxies might cache WebSocket data as HTTP.
  Masking makes data look random, prevents caching.

Masking process:
  1. Generate random 32-bit masking key
  2. XOR each byte of payload with key (rotating)

  masked[i] = payload[i] XOR key[i % 4]

Server-to-client: No masking required.
```

## Fragmentation

Large messages can be split into fragments:

```
Large message (1MB):

Fragment 1: FIN=0, Opcode=0x1 (text), data[0:64KB]
Fragment 2: FIN=0, Opcode=0x0 (continuation), data[64KB:128KB]
Fragment 3: FIN=0, Opcode=0x0 (continuation), data[128KB:192KB]
...
Fragment N: FIN=1, Opcode=0x0 (continuation), data[last portion]

Receiver reassembles before delivering to application.
Allows interleaving with control frames (Ping/Pong).
```

## Full-Duplex in Action

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Simultaneous Communication                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Client                                            Server           │
│     │                                                 │             │
│     │──── "Hello" ────────────────────────────────────│             │
│     │────────────────────────────── "World" ──────────│             │
│     │                        X                        │             │
│     │──── "How are you?" ─────────────────────────────│             │
│     │────────────────────── "Message for you" ────────│             │
│     │                                                 │             │
│     │ Messages cross "in flight"                      │             │
│     │ No waiting for response                         │             │
│     │ Both sides send whenever ready                  │             │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

## Implementation Patterns

### Message Protocol

```javascript
// Define message format
const message = {
  type: 'chat',
  payload: {
    user: 'alice',
    text: 'Hello everyone!'
  },
  timestamp: Date.now()
};

ws.send(JSON.stringify(message));

// Receiver
ws.onmessage = (event) => {
  const msg = JSON.parse(event.data);
  switch (msg.type) {
    case 'chat':
      displayChat(msg.payload);
      break;
    case 'notification':
      showNotification(msg.payload);
      break;
  }
};
```

### Reconnection Logic

```javascript
function connect() {
  const ws = new WebSocket('wss://example.com/socket');

  ws.onopen = () => {
    console.log('Connected');
    reconnectAttempts = 0;
  };

  ws.onclose = (event) => {
    if (event.code !== 1000) {  // Not normal close
      // Exponential backoff
      const delay = Math.min(1000 * 2 ** reconnectAttempts, 30000);
      reconnectAttempts++;
      setTimeout(connect, delay);
    }
  };

  return ws;
}
```

## Summary

WebSocket communication features:

| Aspect | Details |
|--------|---------|
| Frame overhead | 2-14 bytes (vs 100+ for HTTP) |
| Message types | Text (UTF-8), Binary |
| Control frames | Ping, Pong, Close |
| Direction | Full-duplex (simultaneous both ways) |
| Fragmentation | Large messages split across frames |
| Masking | Required for client→server |
