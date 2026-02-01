# Load Balancing

Load balancers distribute traffic across multiple servers, improving availability and performance. Understanding load balancing helps you design scalable systems.

## Why Load Balance?

```
Without load balancing:
  All traffic → Single server
  Problems: Single point of failure, limited capacity

With load balancing:
  Traffic → Load Balancer → Multiple servers
  Benefits: Redundancy, scalability, maintenance flexibility
```

## Load Balancing Algorithms

### Round Robin

```
Request 1 → Server A
Request 2 → Server B
Request 3 → Server C
Request 4 → Server A (repeat)

Pros: Simple, even distribution
Cons: Ignores server capacity, session state
```

### Weighted Round Robin

```
Server A (weight 3): Gets 3x traffic
Server B (weight 1): Gets 1x traffic

Request pattern: A, A, A, B, A, A, A, B, ...

Use case: Servers with different capacities
```

### Least Connections

```
Route to server with fewest active connections.

Server A: 10 connections
Server B: 5 connections
Server C: 8 connections

New request → Server B

Better for variable request durations.
```

### IP Hash

```
Hash(Client IP) → Server selection

Same client always hits same server.
Useful for session affinity without cookies.

hash("192.168.1.100") % 3 = Server B
```

### Least Response Time

```
Route to server with fastest response.

Combines: Connection count + response time
Best for: Heterogeneous backends
Requires: Active health monitoring
```

## Layer 4 vs Layer 7

```
Layer 4 (Transport):
  - Routes based on IP/port
  - Faster (less inspection)
  - Protocol-agnostic
  - No content-based routing

Layer 7 (Application):
  - Routes based on content (URL, headers, cookies)
  - Can modify requests/responses
  - SSL termination
  - More flexible, more overhead

Example Layer 7 rules:
  /api/*     → API servers
  /static/*  → CDN
  /admin/*   → Admin servers
```

## Health Checks

```
Load balancer monitors backends:

Active checks:
  - Periodic HTTP requests to /health
  - TCP connection attempts
  - Custom scripts

Passive checks:
  - Monitor real request success/failure
  - Track response times

Unhealthy server:
  - Remove from rotation
  - Continue checking
  - Return when healthy
```

## Session Persistence

```
Problem: User state spread across servers
  Login on Server A
  Next request hits Server B
  "Please login again" 😞

Solutions:

Sticky Sessions (affinity):
  Set-Cookie: SERVERID=A
  Load balancer routes by cookie

Shared Session Store:
  All servers use Redis/Memcached for sessions
  Any server can handle any request

Stateless Design:
  JWT tokens contain user state
  No server-side session needed (best!)
```

## Common Load Balancers

```
Software:
  - HAProxy: High performance, Layer 4/7
  - nginx: Web server + load balancer
  - Envoy: Modern, service mesh focused
  - Traefik: Cloud-native, auto-discovery

Cloud:
  - AWS ALB/NLB: Layer 7/4
  - GCP Load Balancing: Global, anycast
  - Azure Load Balancer: Layer 4
  - Cloudflare: CDN + load balancing

Hardware (legacy):
  - F5 BIG-IP
  - Citrix NetScaler
```

## Configuration Example (nginx)

```nginx
upstream backend {
    least_conn;
    server 10.0.0.1:8080 weight=3;
    server 10.0.0.2:8080 weight=2;
    server 10.0.0.3:8080 backup;

    keepalive 32;
}

server {
    listen 80;

    location / {
        proxy_pass http://backend;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
    }

    location /health {
        return 200 "OK";
    }
}
```
