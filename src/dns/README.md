# DNS: The Internet's Directory

**DNS (Domain Name System)** is the internet's phone book. It translates human-readable domain names like `example.com` into IP addresses like `93.184.216.34`. Without DNS, we'd have to memorize IP addresses for every website—the internet would be unusable.

## Why DNS Matters

Every network connection starts with DNS:

```
You type: https://github.com
Browser needs: IP address

1. Browser → DNS: "What's the IP for github.com?"
2. DNS → Browser: "140.82.114.3"
3. Browser → 140.82.114.3: "GET / HTTP/1.1"
4. Server → Browser: "Here's the page!"

DNS lookup happens before any connection.
DNS performance affects EVERY request.
```

## The Hierarchical Design

DNS is a distributed database organized as a tree:

```
                              . (root)
                              │
         ┌────────────────────┼────────────────────┐
         │                    │                    │
        com                  org                  net
         │                    │                    │
    ┌────┼────┐          ┌────┼────┐              ...
    │    │    │          │    │
example google ...     wikipedia ...
    │
   www

Domain: www.example.com
  - "." is the root (usually implicit)
  - "com" is the Top-Level Domain (TLD)
  - "example" is the Second-Level Domain
  - "www" is a subdomain
```

## Key Concepts

### Domain Names

```
Fully Qualified Domain Name (FQDN):
  www.example.com.
                 └── Trailing dot means "this is complete"
                     (Usually omitted in browsers)

Labels: Separated by dots
  - Each label: 1-63 characters
  - Total FQDN: max 253 characters
  - Case-insensitive (Example.COM = example.com)
```

### DNS Servers

```
┌─────────────────────────────────────────────────────────────┐
│                  Types of DNS Servers                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Recursive Resolver (Caching Nameserver)                    │
│    - What your computer typically talks to                  │
│    - Does the heavy lifting of finding answers              │
│    - Caches results for faster subsequent queries           │
│    - Examples: 8.8.8.8 (Google), 1.1.1.1 (Cloudflare)       │
│                                                             │
│  Authoritative Nameserver                                   │
│    - Holds the actual DNS records for a zone                │
│    - Is the "source of truth" for that domain               │
│    - Responds to queries about its zones                    │
│                                                             │
│  Root Nameservers                                           │
│    - 13 root server clusters (a.root-servers.net, etc.)     │
│    - Know where to find TLD servers                         │
│    - Foundation of the entire DNS system                    │
│                                                             │
│  TLD Nameservers                                            │
│    - Manage .com, .org, .net, country codes, etc.           │
│    - Know authoritative servers for each domain             │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### The Resolution Process (Preview)

```
Query: "What's the IP for www.example.com?"

Your Computer → Recursive Resolver
                        │
                        ├──> Root Server: "Who handles .com?"
                        │    "Go ask a.gtld-servers.net"
                        │
                        ├──> .com TLD: "Who handles example.com?"
                        │    "Go ask ns1.example.com"
                        │
                        ├──> example.com NS: "What's www.example.com?"
                        │    "It's 93.184.216.34"
                        │
                        └──> Returns answer to your computer

Multiple round trips, but caching makes it fast.
```

## Why DNS Uses UDP (Mostly)

```
Traditional DNS:
  - Small queries (~50 bytes)
  - Small responses (~100-500 bytes)
  - Single request-response
  - Speed matters (affects every page load)

UDP advantages:
  - No connection overhead
  - Faster resolution
  - Lower server load

When TCP is used:
  - Responses > 512 bytes (EDNS extends this)
  - Zone transfers between servers
  - DNS over TLS (DoT)
  - DNS over HTTPS (DoH)
```

## What You'll Learn

In this chapter:

1. **DNS Resolution Process**: How lookups actually work
2. **Record Types**: A, AAAA, CNAME, MX, and more
3. **DNS Caching**: How TTLs and caching improve performance
4. **DNSSEC**: Securing DNS against tampering

Understanding DNS helps you:
- Debug "cannot resolve hostname" errors
- Configure domains correctly
- Understand CDN and load balancing behavior
- Recognize DNS-based attacks
