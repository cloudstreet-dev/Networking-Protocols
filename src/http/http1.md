# HTTP/1.0 and HTTP/1.1

HTTP/1.x established the patterns still used today: request-response over TCP, text-based headers, and the familiar verbs like GET and POST. Understanding these versions explains why later versions were needed.

## HTTP/1.0 (1996)

### Basic Request-Response

```
Client connects to server:
  1. TCP handshake (SYN, SYN-ACK, ACK)
  2. Send HTTP request
  3. Receive HTTP response
  4. Close connection

Every request = New TCP connection!
```

### Request Format

```http
GET /index.html HTTP/1.0
User-Agent: Mozilla/5.0
Accept: text/html

```

That's it—method, path, version, and optional headers. Blank line ends headers.

### Response Format

```http
HTTP/1.0 200 OK
Content-Type: text/html
Content-Length: 1234

<!DOCTYPE html>
<html>
...
</html>
```

Status line, headers, blank line, body.

### The Connection Problem

```
Loading a webpage with HTTP/1.0:

Page needs:
  - index.html (1 request)
  - style.css (1 request)
  - script.js (1 request)
  - logo.png (1 request)
  - header.png (1 request)

HTTP/1.0 timeline (sequential):
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                         │
│  ├─TCP─┤├───HTML────┤                                                   │
│                      ├─TCP─┤├───CSS────┤                                │
│                                         ├─TCP─┤├───JS────┤              │
│                                                          ├─TCP─┤├PNG1─┤ │
│                                                                   ...   │
│                                                                         │
│  Total: 5 TCP handshakes + 5 requests = Very slow!                      │
└─────────────────────────────────────────────────────────────────────────┘

Each resource requires:
  - TCP handshake (~1 RTT)
  - Request + response (~1 RTT)
  - TCP teardown

For 10 resources over 100ms RTT: ~2 seconds just for overhead!
```

## HTTP/1.1 (1997)

HTTP/1.1 addressed the connection overhead with several improvements.

### Persistent Connections

Connections stay open by default:

```
HTTP/1.0:
  Connection: close        (default, close after response)
  Connection: keep-alive   (optional, keep open)

HTTP/1.1:
  Connection: keep-alive   (default, keep open)
  Connection: close        (optional, close after response)
```

### Connection Reuse

```
HTTP/1.1 timeline (persistent connection):
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                         │
│  ├─TCP─┤├─HTML─┤├─CSS─┤├─JS─┤├─PNG1─┤├─PNG2─┤                           │
│                                                                         │
│  One TCP handshake, multiple requests!                                  │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘

Savings: 4 fewer TCP handshakes = ~400ms on 100ms RTT link
```

### Pipelining

Send multiple requests without waiting for responses:

```
Without pipelining:
  Request 1 → Response 1 → Request 2 → Response 2

With pipelining:
  Request 1 → Request 2 → Request 3 → Response 1 → Response 2 → Response 3

┌─────────────────────────────────────────────────────────────────────────┐
│                                                                         │
│  Client:  [Req1][Req2][Req3]                                            │
│  Server:                    [Resp1][Resp2][Resp3]                       │
│                                                                         │
│  Server processes in parallel (potentially)                             │
│  But responses MUST be in request order!                                │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Pipelining's Fatal Flaw: Head-of-Line Blocking

```
Pipelining problem:

Requests sent:  [HTML][CSS][JS]
Server ready:   JS(10ms), CSS(20ms), HTML(500ms)

Must respond in order:
  ├────────HTML (500ms)────────┤├CSS┤├JS┤

JS is ready instantly but waits 500ms for HTML!
This is "head-of-line blocking."

Reality: Pipelining rarely used
  - Complex to implement correctly
  - Many proxies don't support it
  - HOL blocking negates benefits
  - Browsers disabled it by default
```

### Multiple Connections Workaround

Browsers work around HTTP/1.1 limitations:

```
Browser opens 6 parallel connections per domain:

Connection 1: [HTML]─────────[Image5]──────────
Connection 2: [CSS]─────[Image1]──────[Image6]─
Connection 3: [JS1]─────[Image2]───────────────
Connection 4: [JS2]─────[Image3]───────────────
Connection 5: [Font]────[Image4]───────────────
Connection 6: [Icon]────[Image7]───────────────

Parallel downloads without pipelining!

But: 6 TCP connections = 6× overhead
     6× congestion control windows
     Not efficient
```

### Domain Sharding (Historical)

```
Workaround for 6-connection limit:

Instead of:
  example.com/style.css
  example.com/script.js
  example.com/image1.png

Use:
  example.com/style.css
  static1.example.com/script.js
  static2.example.com/image1.png

Browser sees different domains:
  6 connections to example.com
  6 connections to static1.example.com
  6 connections to static2.example.com
  = 18 parallel connections!

Downsides:
  - More TCP overhead
  - More TLS handshakes (if HTTPS)
  - DNS lookups for each domain
  - Cache fragmentation

Note: Harmful with HTTP/2! (multiplexing is better)
```

### Host Header (Required)

HTTP/1.1 requires the Host header:

```http
GET /page.html HTTP/1.1
Host: www.example.com

```

Enables virtual hosting—multiple sites on one IP:

```
Server at 192.168.1.100 hosts:
  - www.example.com
  - www.another-site.com
  - api.example.com

Host header tells server which site is requested.
Without it: Server doesn't know which site you want!
```

### Chunked Transfer Encoding

Send response without knowing size upfront:

```http
HTTP/1.1 200 OK
Transfer-Encoding: chunked

1a
This is the first chunk.
1b
This is the second chunk.
0

```

Format: Size (hex) + CRLF + Data + CRLF, ending with 0

Use cases:
- Streaming responses
- Server-generated content
- Live data feeds

### Additional HTTP/1.1 Features

**100 Continue**
```http
Client: POST /upload HTTP/1.1
        Content-Length: 10000000
        Expect: 100-continue

Server: HTTP/1.1 100 Continue

Client: (sends 10MB body)

Server: HTTP/1.1 200 OK
```
Avoids sending large body if server will reject it.

**Range Requests**
```http
GET /large-file.zip HTTP/1.1
Range: bytes=1000-1999

HTTP/1.1 206 Partial Content
Content-Range: bytes 1000-1999/50000
```
Resume interrupted downloads, video seeking.

**Cache Control**
```http
Cache-Control: max-age=3600, must-revalidate
ETag: "abc123"
Last-Modified: Wed, 21 Oct 2015 07:28:00 GMT
```
Sophisticated caching for performance.

## HTTP/1.1 Example Session

```
$ telnet example.com 80
Trying 93.184.216.34...
Connected to example.com.

GET / HTTP/1.1
Host: example.com
Connection: keep-alive

HTTP/1.1 200 OK
Content-Type: text/html; charset=UTF-8
Content-Length: 1256
Connection: keep-alive
Cache-Control: max-age=604800

<!doctype html>
<html>
<head>
    <title>Example Domain</title>
...

GET /favicon.ico HTTP/1.1
Host: example.com
Connection: close

HTTP/1.1 404 Not Found
Content-Length: 0
Connection: close

Connection closed by foreign host.
```

## HTTP/1.x Limitations Summary

```
┌─────────────────────────────────────────────────────────────────────────┐
│               HTTP/1.x Limitations                                      │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  1. Head-of-Line Blocking                                               │
│     Responses must be in request order                                  │
│     One slow response blocks all others                                 │
│                                                                         │
│  2. Textual Protocol Overhead                                           │
│     Headers are uncompressed text                                       │
│     Same headers sent repeatedly                                        │
│                                                                         │
│  3. No Request Prioritization                                           │
│     Can't indicate which resources are critical                         │
│     Server processes arbitrarily                                        │
│                                                                         │
│  4. Client-Initiated Only                                               │
│     Server can't push resources proactively                             │
│     Client must request everything explicitly                           │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

## When HTTP/1.1 Is Still Used

```
HTTP/1.1 remains common for:
  - Simple APIs (few requests per connection)
  - Internal services (low latency networks)
  - Legacy system compatibility
  - Debugging (human-readable)
  - When HTTP/2 isn't supported

Modern web: HTTP/2 or HTTP/3 preferred
  Better performance with no application changes
```

## Summary

| Feature | HTTP/1.0 | HTTP/1.1 |
|---------|----------|----------|
| Persistent connections | Optional | Default |
| Host header | Optional | Required |
| Chunked transfer | No | Yes |
| Pipelining | No | Yes (rarely used) |
| Cache-Control | Limited | Full support |
| Range requests | No | Yes |
| 100 Continue | No | Yes |

HTTP/1.1 significantly improved on 1.0 but still suffers from head-of-line blocking. HTTP/2 was designed to solve this—which we'll explore next.
