# CDNs

**Content Delivery Networks (CDNs)** cache content at edge locations worldwide, reducing latency by serving users from nearby servers.

## How CDNs Work

```
┌─────────────────────────────────────────────────────────────────────┐
│                      CDN Architecture                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│           Origin Server (Your Server)                               │
│                    │                                                │
│        ┌──────────┼──────────┐                                      │
│        │          │          │                                      │
│        ▼          ▼          ▼                                      │
│   ┌────────┐ ┌────────┐ ┌────────┐                                  │
│   │  Edge  │ │  Edge  │ │  Edge  │                                  │
│   │  US    │ │  EU    │ │  Asia  │                                  │
│   └────┬───┘ └────┬───┘ └────┬───┘                                  │
│        │          │          │                                      │
│        ▼          ▼          ▼                                      │
│      Users      Users      Users                                    │
│                                                                     │
│   User requests → Nearest Edge → Cached response (fast!)            │
│   Cache miss → Edge fetches from Origin → Caches → Responds         │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

## Benefits

```
Performance:
  - Reduced latency (content closer to users)
  - Faster page loads
  - Better user experience

Scalability:
  - Offload traffic from origin
  - Handle traffic spikes
  - Global reach without global infrastructure

Availability:
  - DDoS protection
  - Origin failover
  - Always-on edge presence
```

## What to Put on CDN

```
Ideal for CDN:
  ✓ Static files (JS, CSS, images)
  ✓ Videos and media
  ✓ Downloadable files
  ✓ Public API responses (if cacheable)

Not for CDN (usually):
  ✗ User-specific content
  ✗ Real-time data
  ✗ Authenticated endpoints
  ✗ Frequently changing data
```

## Cache Control

```http
# Cache for 1 day, revalidate after
Cache-Control: public, max-age=86400, must-revalidate

# Cache for 1 year (immutable assets)
Cache-Control: public, max-age=31536000, immutable

# No caching
Cache-Control: no-store

# Private only (not CDN)
Cache-Control: private, max-age=3600
```

## CDN Configuration Concepts

```
TTL (Time To Live):
  How long edge caches content
  Balance freshness vs. origin load

Cache Keys:
  What makes requests "same" for caching
  URL, headers, cookies, query strings

Purge/Invalidation:
  Force refresh of cached content
  By URL, tag, or entire cache

Edge Functions:
  Run code at edge (Cloudflare Workers, Lambda@Edge)
  Customize responses, A/B testing, auth
```

## Popular CDNs

```
Global CDNs:
  - Cloudflare: Free tier, security focus
  - Fastly: Real-time purging, edge compute
  - Akamai: Enterprise, largest network
  - AWS CloudFront: AWS integration
  - Google Cloud CDN: GCP integration

Specialized:
  - Bunny CDN: Cost-effective
  - KeyCDN: Simple pricing
  - imgix: Image optimization focus
```

## Setting Up CDN

```
Basic setup:
  1. Sign up with CDN provider
  2. Configure origin (your server)
  3. Get CDN domain (e.g., cdn.example.com)
  4. Update DNS or reference CDN URLs
  5. Configure cache rules

DNS example:
  cdn.example.com CNAME example.cdn-provider.net

Or full site through CDN:
  example.com → CDN → origin.example.com
```

## Debugging CDN

```bash
# Check cache status
$ curl -I https://cdn.example.com/image.png
X-Cache: HIT          # Served from edge
X-Cache: MISS         # Fetched from origin
CF-Cache-Status: HIT  # Cloudflare specific

# Check which edge served request
$ curl -I https://cdn.example.com/image.png
CF-RAY: 123abc-SJC    # Cloudflare San Jose

# Bypass cache
$ curl -H "Cache-Control: no-cache" https://cdn.example.com/image.png
```
