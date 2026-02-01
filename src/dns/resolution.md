# DNS Resolution Process

When your browser needs to find `example.com`, a complex but elegant process unfolds. Understanding this process helps you debug DNS issues and optimize performance.

## The Query Journey

A full DNS resolution involves multiple servers:

```
┌─────────────┐    ┌───────────────┐    ┌───────────────┐
│Your Computer│───>│   Recursive   │───>│ Root Servers  │
│             │    │   Resolver    │    │ (13 clusters) │
│  (Stub      │    │  (8.8.8.8)    │    │               │
│   Resolver) │    │               │    └───────┬───────┘
└─────────────┘    │               │            │
                   │               │<───────────┘
                   │               │    ┌───────────────┐
                   │               │───>│ TLD Servers   │
                   │               │    │ (.com, .org)  │
                   │               │    │               │
                   │               │<───┴───────────────┘
                   │               │    ┌───────────────┐
                   │               │───>│ Authoritative │
                   │               │    │   Nameserver  │
                   └───────┬───────┘    │ (example.com) │
                           │            └───────────────┘
                           │
                    Answer returned
                    to your computer
```

## Step-by-Step Resolution

Let's trace a query for `www.example.com`:

### Step 1: Local Stub Resolver

```
Your computer checks (in order):
  1. Local cache (recently resolved names)
  2. /etc/hosts file (manual overrides)
  3. If not found → Query configured DNS server

$ cat /etc/hosts
127.0.0.1   localhost
192.168.1.10  myserver.local

$ cat /etc/resolv.conf  # Linux
nameserver 8.8.8.8
nameserver 8.8.4.4

If not in cache or hosts → Send UDP query to 8.8.8.8
```

### Step 2: Recursive Resolver Check Cache

```
Recursive resolver (8.8.8.8) checks its cache:

Cache might have:
  - www.example.com → 93.184.216.34 (exact match!)
  - example.com NS → ns1.example.com (partial help)
  - .com NS → a.gtld-servers.net (partial help)

Cache hit? Return immediately!
Cache miss? Start the recursive lookup.
```

### Step 3: Query Root Servers

```
Resolver → Root Server (a.root-servers.net)

Q: "What's the IP for www.example.com?"

Root server response:
  "I don't know www.example.com, but .com is handled by:
   a.gtld-servers.net (192.5.6.30)
   b.gtld-servers.net (192.33.14.30)
   ... (and others)

   This is a REFERRAL, not an answer.
   Go ask them."

Type: NS (Name Server) referral
```

### Step 4: Query TLD Servers

```
Resolver → .com TLD Server (a.gtld-servers.net)

Q: "What's the IP for www.example.com?"

TLD server response:
  "I don't know www.example.com, but example.com is handled by:
   ns1.example.com (93.184.216.34)
   ns2.example.com (93.184.216.34)

   Go ask them."

Type: NS referral + glue records (IPs of nameservers)
```

### Step 5: Query Authoritative Server

```
Resolver → Authoritative NS (ns1.example.com)

Q: "What's the IP for www.example.com?"

Authoritative response:
  "www.example.com has address 93.184.216.34"

Type: A record (the actual answer!)

This server IS authoritative for example.com.
The answer is definitive, not a referral.
```

### Step 6: Return to Client

```
Recursive resolver:
  1. Caches the answer (and intermediate results)
  2. Returns answer to your computer

Your computer:
  1. Caches the answer
  2. Uses IP to connect

Total time: 50-200ms (uncached)
Cached lookup: <1ms
```

## Query Types

### Recursive Query

```
Client → Recursive Resolver:
"Get me the answer, do whatever it takes"

Resolver must:
  - Return the answer, OR
  - Return an error

Client doesn't do iterative lookups itself.
```

### Iterative Query

```
Resolver → Authoritative Servers:
"Tell me what you know"

Server response can be:
  - The answer (if authoritative)
  - A referral (try somewhere else)
  - Error (doesn't exist)

Resolver follows referrals iteratively.
```

## DNS Message Format

```
DNS Query/Response Structure:

┌────────────────────────────────────────────────────────────┐
│                        Header                              │
│  - Query ID (match responses to queries)                   │
│  - Flags (query/response, recursion desired, etc.)         │
│  - Question count, Answer count, Authority count, etc.     │
├────────────────────────────────────────────────────────────┤
│                       Question                             │
│  - Name: www.example.com                                   │
│  - Type: A (or AAAA, MX, etc.)                             │
│  - Class: IN (Internet)                                    │
├────────────────────────────────────────────────────────────┤
│                        Answer                              │
│  - Name: www.example.com                                   │
│  - Type: A                                                 │
│  - TTL: 3600                                               │
│  - Data: 93.184.216.34                                     │
├────────────────────────────────────────────────────────────┤
│                      Authority                             │
│  (Nameservers for the zone)                                │
├────────────────────────────────────────────────────────────┤
│                      Additional                            │
│  (Extra helpful records, like NS IP addresses)             │
└────────────────────────────────────────────────────────────┘
```

## DNS Query in Action

Using `dig` to see the resolution:

```bash
$ dig www.example.com +trace

; <<>> DiG 9.16.1 <<>> www.example.com +trace
;; global options: +cmd
.                       518400  IN  NS  a.root-servers.net.
.                       518400  IN  NS  b.root-servers.net.
;; Received 262 bytes from 8.8.8.8#53(8.8.8.8) in 12 ms

com.                    172800  IN  NS  a.gtld-servers.net.
com.                    172800  IN  NS  b.gtld-servers.net.
;; Received 828 bytes from 192.58.128.30#53(a.root-servers.net) in 24 ms

example.com.            172800  IN  NS  ns1.example.com.
example.com.            172800  IN  NS  ns2.example.com.
;; Received 268 bytes from 192.5.6.30#53(a.gtld-servers.net) in 32 ms

www.example.com.        3600    IN  A   93.184.216.34
;; Received 56 bytes from 93.184.216.34#53(ns1.example.com) in 16 ms
```

## Negative Responses

What if the domain doesn't exist?

### NXDOMAIN

```
Query: thisdomaindoesnotexist.com

Response:
  Status: NXDOMAIN (Non-Existent Domain)
  Meaning: Domain doesn't exist at all

This is authoritative - the domain really doesn't exist.
Can be cached (negative caching).
```

### NODATA

```
Query: example.com (type AAAA for IPv6)

Response:
  Status: NODATA
  Meaning: Domain exists but no record of this type

example.com has A records but no AAAA records.
Also cached negatively.
```

## Resolver Behavior

### Timeouts and Retries

```
Resolver query to server times out:

Default behavior:
  Timeout: ~2 seconds
  Retries: 2-3 attempts
  Tries alternate servers in list

Total resolution might take:
  Best case: <50ms (cached)
  Typical: 50-200ms (uncached)
  Worst case: Several seconds (timeouts)
```

### Server Selection

```
Multiple nameservers for redundancy:

ns1.example.com
ns2.example.com

Resolver tracks:
  - Response times per server
  - Failure counts
  - Prefers faster/more reliable servers

"Smoothed Round Trip Time" (SRTT) helps pick fastest.
```

## Common Resolution Issues

### "Could not resolve hostname"

```
Causes:
  1. DNS server unreachable (network issue)
  2. Domain doesn't exist (NXDOMAIN)
  3. DNS server returning errors
  4. Local resolver misconfigured

Debug:
  $ nslookup example.com
  $ dig example.com
  $ ping 8.8.8.8  # Can you reach DNS server?
```

### Slow Resolution

```
Causes:
  1. Cache empty (first lookup is slow)
  2. DNS server far away
  3. DNS server overloaded
  4. Network latency

Solutions:
  - Use closer DNS server
  - Increase local cache size/TTL
  - Pre-resolve critical domains
```

### Stale Cache

```
Situation:
  Website changed IP
  Your cache still has old IP
  Connection fails

Solutions:
  $ sudo systemd-resolve --flush-caches  # Linux systemd
  $ sudo dscacheutil -flushcache          # macOS
  $ ipconfig /flushdns                    # Windows

  Or wait for TTL to expire.
```

## Programming with DNS

### Basic Lookup (Python)

```python
import socket

# Simple lookup
ip = socket.gethostbyname('example.com')
print(ip)  # 93.184.216.34

# Get all addresses (IPv4 + IPv6)
infos = socket.getaddrinfo('example.com', 80)
for info in infos:
    family, socktype, proto, canonname, sockaddr = info
    print(f"{family.name}: {sockaddr[0]}")
```

### Using dnspython Library

```python
import dns.resolver

# A record lookup
answers = dns.resolver.resolve('example.com', 'A')
for rdata in answers:
    print(f"IP: {rdata}")

# MX record lookup
answers = dns.resolver.resolve('example.com', 'MX')
for rdata in answers:
    print(f"Mail server: {rdata.exchange} (priority {rdata.preference})")

# Tracing (like dig +trace)
import dns.query
import dns.zone

# ... more advanced queries
```

## Summary

DNS resolution follows a hierarchical pattern:

```
Your Computer
    │
    ▼
Recursive Resolver (does the work)
    │
    ├──> Root Servers (.com? .org? .net?)
    │
    ├──> TLD Servers (example.com? github.com?)
    │
    └──> Authoritative Servers (www? mail? api?)
            │
            ▼
         Answer!
```

Key points:
- **Stub resolvers** on your computer do minimal work
- **Recursive resolvers** (like 8.8.8.8) do the heavy lifting
- **Caching** at every level makes it fast
- **Authoritative servers** are the source of truth
- **TTL values** control cache duration

Next, we'll explore the different types of DNS records and their uses.
