# HTTP Evolution

**HTTP (Hypertext Transfer Protocol)** is the foundation of the web. What started as a simple protocol for retrieving hypertext documents has evolved into a sophisticated system powering everything from websites to APIs to real-time applications.

## The Journey

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        HTTP Timeline                                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  1991    HTTP/0.9    One-line protocol, GET only                        │
│    │                                                                    │
│  1996    HTTP/1.0    Headers, methods, status codes                     │
│    │                 Problem: One request per connection                │
│    │                                                                    │
│  1997    HTTP/1.1    Persistent connections, pipelining                 │
│    │                 Problem: Head-of-line blocking                     │
│    │                                                                    │
│  2015    HTTP/2      Binary, multiplexing, server push                  │
│    │                 Problem: TCP head-of-line blocking                 │
│    │                                                                    │
│  2022    HTTP/3      QUIC transport, UDP-based                          │
│                      Eliminates transport-level blocking                │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

## Why HTTP Keeps Evolving

Each HTTP version addressed limitations of its predecessor:

```
HTTP/1.0 → HTTP/1.1
  Problem: Opening new TCP connection per request is slow
  Solution: Keep connections open (persistent connections)

HTTP/1.1 → HTTP/2
  Problem: Requests must wait in line, even on persistent connections
  Solution: Multiplex requests over single connection

HTTP/2 → HTTP/3
  Problem: TCP packet loss blocks ALL streams
  Solution: Use QUIC (UDP-based), independent stream delivery
```

## Request-Response Model

Despite version differences, HTTP maintains its fundamental model:

```
┌────────────────────────────────────────────────────────────────┐
│                        HTTP Transaction                        │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  Client                                Server                  │
│     │                                     │                    │
│     │─────────── Request ────────────────>│                    │
│     │                                     │                    │
│     │  GET /index.html HTTP/1.1           │                    │
│     │  Host: example.com                  │                    │
│     │  Accept: text/html                  │                    │
│     │                                     │                    │
│     │                                     │                    │
│     │<──────────── Response ──────────────│                    │
│     │                                     │                    │
│     │  HTTP/1.1 200 OK                    │                    │
│     │  Content-Type: text/html            │                    │
│     │  Content-Length: 1234               │                    │
│     │                                     │                    │
│     │  <!DOCTYPE html>...                 │                    │
│     │                                     │                    │
└────────────────────────────────────────────────────────────────┘

Request  = Method + Path + Headers + (optional) Body
Response = Status + Headers + (optional) Body
```

## Key HTTP Concepts

### Methods

```
GET      Retrieve a resource
POST     Submit data, create resource
PUT      Replace a resource
PATCH    Partially modify a resource
DELETE   Remove a resource
HEAD     GET without body (metadata only)
OPTIONS  Describe communication options
```

### Status Codes

```
1xx  Informational    100 Continue, 101 Switching Protocols
2xx  Success          200 OK, 201 Created, 204 No Content
3xx  Redirection      301 Moved, 302 Found, 304 Not Modified
4xx  Client Error     400 Bad Request, 401 Unauthorized, 404 Not Found
5xx  Server Error     500 Internal Error, 502 Bad Gateway, 503 Unavailable
```

### Headers

```
Request headers:
  Host: example.com           (required in HTTP/1.1+)
  Accept: application/json    (preferred response type)
  Authorization: Bearer xyz   (credentials)
  Cookie: session=abc123      (state)

Response headers:
  Content-Type: text/html     (body format)
  Content-Length: 1234        (body size)
  Cache-Control: max-age=3600 (caching rules)
  Set-Cookie: session=abc123  (set state)
```

## What You'll Learn

In this chapter:

1. **HTTP/1.0 and HTTP/1.1**: The text-based foundation
2. **HTTP/2**: Binary framing and multiplexing
3. **HTTP/3 and QUIC**: The modern, UDP-based protocol

Understanding HTTP evolution helps you:
- Choose appropriate protocol versions
- Optimize web performance
- Debug connection issues
- Design efficient APIs
