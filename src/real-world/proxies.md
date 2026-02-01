# Proxies and Reverse Proxies

Proxies act as intermediaries in network communication. Understanding them helps you design secure architectures and debug connectivity issues.

## Forward Proxy

Client-side proxy: Client → Proxy → Internet

```
┌─────────────────────────────────────────────────────────────────────┐
│                       Forward Proxy                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   ┌────────┐       ┌───────┐       ┌──────────────┐                │
│   │ Client │──────>│ Proxy │──────>│   Internet   │                │
│   └────────┘       └───────┘       │   (Server)   │                │
│                        │           └──────────────┘                │
│                        │                                           │
│   Proxy hides client identity from server.                         │
│   Server sees proxy's IP, not client's.                            │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘

Use cases:
  - Corporate content filtering
  - Caching (reduce bandwidth)
  - Anonymity
  - Access control
```

## Reverse Proxy

Server-side proxy: Internet → Reverse Proxy → Servers

```
┌─────────────────────────────────────────────────────────────────────┐
│                       Reverse Proxy                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   ┌──────────────┐       ┌───────────┐       ┌────────┐            │
│   │   Internet   │──────>│  Reverse  │──────>│ Server │            │
│   │   (Client)   │       │   Proxy   │──────>│ Server │            │
│   └──────────────┘       └───────────┘──────>│ Server │            │
│                               │              └────────┘            │
│                               │                                    │
│   Clients don't know backend servers exist.                        │
│   Single entry point to multiple backends.                         │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘

Use cases:
  - SSL termination
  - Load balancing
  - Caching
  - Compression
  - Security (hide backend)
  - A/B testing
```

## Reverse Proxy Functions

### SSL Termination

```
Client ──HTTPS──> Reverse Proxy ──HTTP──> Backend

Proxy handles TLS:
  - Certificate management in one place
  - Offloads crypto from backends
  - Backends get plain HTTP (simpler)
  - Internal traffic often trusted network
```

### Request Routing

```
Based on URL path:
  /api/*     → API servers
  /images/*  → Image servers
  /          → Web servers

Based on header:
  Host: api.example.com → API servers
  Host: www.example.com → Web servers

Based on cookie:
  beta_user=true → Beta servers
```

### Caching

```
Cache responses at proxy level:

Request 1: GET /logo.png
  Proxy → Backend → Response (cached at proxy)

Request 2: GET /logo.png
  Proxy → Cache hit → Response (no backend call)

Reduces backend load significantly.
```

## Common Proxy Headers

```
X-Forwarded-For: Client IP (through proxy chain)
  X-Forwarded-For: 203.0.113.195, 70.41.3.18, 150.172.238.178

X-Forwarded-Proto: Original protocol
  X-Forwarded-Proto: https

X-Forwarded-Host: Original Host header
  X-Forwarded-Host: www.example.com

X-Real-IP: Single client IP (nginx convention)
  X-Real-IP: 203.0.113.195
```

## Proxy Protocols

### HTTP CONNECT (Forward Proxy)

```
Client → Proxy: CONNECT example.com:443 HTTP/1.1
Proxy → Client: HTTP/1.1 200 Connection Established
Client → (tunnel) → Server

Proxy creates TCP tunnel.
Used for HTTPS through forward proxies.
```

### PROXY Protocol (Reverse Proxy)

```
HAProxy-style protocol:
  Passes original client info to backend.
  Binary or text header prepended to connection.

PROXY TCP4 192.168.1.1 10.0.0.1 56789 80\r\n
(Then normal HTTP traffic)

Backend sees real client IP.
```

## nginx Reverse Proxy Config

```nginx
server {
    listen 443 ssl;
    server_name example.com;

    ssl_certificate /etc/nginx/cert.pem;
    ssl_certificate_key /etc/nginx/key.pem;

    location / {
        proxy_pass http://backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # WebSocket support
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

## Debugging Through Proxies

```bash
# Check what proxy sees
$ curl -v -x http://proxy:8080 https://example.com

# See forwarded headers
$ curl -s https://httpbin.org/headers

# Trace proxy chain
$ curl -s https://httpbin.org/ip
# Returns visible IP (proxy's if through proxy)
```
