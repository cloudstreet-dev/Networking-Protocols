# DNS Caching

Caching is what makes DNS fast. Without it, every web request would require multiple round trips to root servers, TLD servers, and authoritative servers. Understanding caching helps you balance performance against update speed.

## The Caching Hierarchy

DNS caches exist at multiple levels:

```
┌─────────────────────────────────────────────────────────────┐
│                    Caching Layers                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Browser Cache          (seconds to minutes)                │
│       ↓                                                     │
│  Operating System       (minutes to hours)                  │
│       ↓                                                     │
│  Local DNS Server       (minutes to hours)                  │
│  (home router, office)                                      │
│       ↓                                                     │
│  Recursive Resolver     (minutes to days)                   │
│  (8.8.8.8, ISP DNS)                                         │
│       ↓                                                     │
│  Authoritative Server   (source of truth)                   │
│                                                             │
└─────────────────────────────────────────────────────────────┘

Each level can serve cached responses.
Request only goes further if cache misses.
```

## TTL (Time To Live)

Every DNS record has a TTL that controls how long it can be cached:

```
Record with TTL:
  example.com.    3600    IN    A    93.184.216.34
                   │
                   └── Cache for 3600 seconds (1 hour)

When cached:
  - Resolver stores record with timestamp
  - Returns cached response for subsequent queries
  - After TTL expires, must re-query authoritative server
```

### How TTL Decrements

```
Authoritative server returns:
  example.com.    3600    IN    A    93.184.216.34

Resolver caches at T=0:
  TTL remaining: 3600

After 1000 seconds (T=1000):
  Client queries resolver
  Resolver returns from cache with TTL=2600

After 3600 seconds (T=3600):
  TTL=0, entry expired
  Next query goes to authoritative server
  Fresh record cached with new TTL
```

## Browser DNS Cache

Browsers maintain their own DNS cache:

```
Chrome: chrome://net-internals/#dns
Firefox: about:networking#dns
Safari: No direct viewer

Typical browser cache TTL: Capped at 1-60 seconds
(Shorter than OS cache to detect changes faster)

Clearing browser cache:
  - Chrome: Settings → Privacy → Clear browsing data
  - Firefox: Settings → Privacy → Clear Data
  - Or restart browser
```

## Operating System Cache

### Linux (systemd-resolved)

```bash
# View cache statistics
$ resolvectl statistics

# View cached entries (limited)
$ resolvectl query example.com

# Flush cache
$ sudo resolvectl flush-caches

# Alternative (older systems)
$ sudo systemctl restart systemd-resolved
```

### macOS

```bash
# Flush DNS cache
$ sudo dscacheutil -flushcache
$ sudo killall -HUP mDNSResponder

# View cached entries (limited visibility)
$ sudo killall -INFO mDNSResponder
# Check Console.app for output
```

### Windows

```powershell
# View cache
> ipconfig /displaydns

# Flush cache
> ipconfig /flushdns

# Check DNS client service
> Get-Service dnscache
```

## Recursive Resolver Cache

Public resolvers like 8.8.8.8 cache extensively:

```
Benefits:
  - Single query serves millions of users
  - Popular domains almost always cached
  - Reduced load on authoritative servers

Cache characteristics:
  - Respects TTL from authoritative
  - May apply minimum TTL (typically 60s)
  - May cap maximum TTL (typically 24-48h)
  - Huge cache (millions of entries)
```

### Cache Warming

Large resolvers "warm" their caches:

```
Popular domain (google.com):
  - Millions of queries per second
  - Always in cache
  - TTL never truly expires (refreshed constantly)

Obscure domain (your-small-site.com):
  - Few queries
  - May fall out of cache between queries
  - Each visitor might trigger fresh lookup
```

## Negative Caching

Failed lookups are also cached:

```
Query: nonexistent.example.com
Response: NXDOMAIN (doesn't exist)

Cached as negative response:
  - Saves repeated queries for invalid domains
  - TTL from SOA minimum field
  - Typically cached for minutes to hours

RFC 2308 defines negative caching behavior.
```

### Negative Cache Problems

```
Scenario:
  1. Query new domain before DNS propagates
  2. Get NXDOMAIN (not yet available)
  3. NXDOMAIN cached for 1 hour
  4. Domain IS available 5 minutes later
  5. Still getting NXDOMAIN from cache!

Solution:
  - Wait for negative cache to expire
  - Flush local DNS cache
  - Use different resolver temporarily
```

## TTL Strategies

### Long TTL (3600-86400 seconds)

```
Pros:
  + Fewer queries to authoritative servers
  + Faster lookups (usually cached)
  + Less DNS infrastructure needed

Cons:
  - Slow propagation of changes
  - Failover takes time
  - Users may hit stale data

Best for:
  - Stable infrastructure
  - Rarely-changing records
  - Cost/performance optimization
```

### Short TTL (60-300 seconds)

```
Pros:
  + Quick propagation of changes
  + Fast failover
  + More control over traffic

Cons:
  - More queries (higher load)
  - Slightly higher latency on cache miss
  - More authoritative server capacity needed

Best for:
  - Dynamic infrastructure
  - Traffic management
  - Disaster recovery scenarios
```

### TTL Strategy by Record Type

```
┌──────────────────────────────────────────────────────────────┐
│ Record Type     │ Recommended TTL    │ Rationale             │
├─────────────────┼────────────────────┼───────────────────────┤
│ NS              │ 86400 (24 hours)   │ Rarely change         │
│ MX              │ 3600-14400         │ Email can retry       │
│ A/AAAA (stable) │ 3600-86400         │ Usually cached anyway │
│ A/AAAA (dynamic)│ 60-300             │ Need quick updates    │
│ CNAME           │ 3600               │ Depends on target     │
│ TXT (SPF/DKIM)  │ 3600               │ Reasonable balance    │
└──────────────────────────────────────────────────────────────┘
```

## TTL and DNS Migrations

When changing DNS records, manage TTL proactively:

```
Timeline for IP change:

T-24h: Reduce TTL
  example.com.    300    IN    A    93.184.216.34
  (Old IP, short TTL now)

T-0: Make the change
  example.com.    300    IN    A    198.51.100.50
  (New IP)

T+1h: Verify traffic shifted

T+24h: Restore normal TTL
  example.com.    3600    IN    A    198.51.100.50
  (New IP, normal TTL)

The "reduce before, restore after" pattern minimizes
stale cache impact during changes.
```

## Cache Debugging

### Check What's Cached

```bash
# Query specific resolver (bypass local cache)
$ dig @8.8.8.8 example.com

# Check TTL remaining
$ dig example.com | grep -A1 "ANSWER SECTION"
example.com.    2847    IN    A    93.184.216.34
                 │
                 └── 2847 seconds remaining in cache

# Compare different resolvers
$ dig @8.8.8.8 example.com +short
$ dig @1.1.1.1 example.com +short
$ dig @9.9.9.9 example.com +short

# Different results = propagation in progress
```

### Force Fresh Lookup

```bash
# Query authoritative directly
$ dig @ns1.example.com example.com

# Trace (bypasses cache, queries authoritatively)
$ dig +trace example.com

# No recursion (only ask one server)
$ dig +norecurse @a.root-servers.net example.com
```

## Caching Issues

### Inconsistent Results

```
Problem:
  dig @8.8.8.8 example.com → 1.2.3.4
  dig @1.1.1.1 example.com → 5.6.7.8

Causes:
  - Recent change, propagation in progress
  - Different servers have different cache ages
  - Anycast resolvers hit different instances

Solution:
  Wait for TTL to expire everywhere
  Typically resolves within max(TTL) time
```

### Cached Failure

```
Problem:
  DNS change made, but users still see old/error

Causes:
  - Negative caching (NXDOMAIN cached)
  - Old positive record still valid
  - Client-side cache not flushed

Debug:
  1. Check TTL of cached record
  2. Check negative TTL (SOA minimum)
  3. Flush caches at multiple levels
  4. Wait for TTL expiration
```

### Cache Poisoning (Security)

```
Attack:
  Attacker injects fake record into resolver cache
  Users sent to malicious server

Mitigations:
  - DNSSEC (cryptographic validation)
  - Source port randomization
  - Query ID randomization
  - Response validation (0x20 encoding)
```

## Summary

DNS caching is hierarchical and TTL-controlled:

| Cache Location | Typical TTL Cap | Flush Method |
|----------------|-----------------|--------------|
| Browser | 60s | Restart or clear |
| OS | varies | System-specific |
| Local resolver | varies | Restart service |
| Recursive resolver | Respects record TTL | Wait |

TTL guidelines:
- **Stable records**: 3600-86400 seconds
- **Dynamic records**: 60-300 seconds
- **Before changes**: Reduce TTL in advance
- **After changes**: Wait for old TTL to expire

Caching makes DNS fast but requires understanding for:
- Planning DNS changes
- Debugging resolution issues
- Balancing freshness vs. performance

Next, we'll explore DNSSEC—how DNS responses can be cryptographically validated.
