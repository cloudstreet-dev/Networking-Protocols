# Connection Pooling

Connection pooling reuses established connections instead of creating new ones for each request. This is essential for performance in database access, HTTP clients, and service communication.

## Why Pool Connections?

```
Without pooling (connection per request):
  Request 1: [TCP handshake][TLS handshake][Query][Response][Close]
  Request 2: [TCP handshake][TLS handshake][Query][Response][Close]
  Request 3: [TCP handshake][TLS handshake][Query][Response][Close]

  Each request pays full connection overhead!

With pooling (reuse connections):
  [TCP][TLS] ← Once
  Request 1: [Query][Response]
  Request 2: [Query][Response]
  Request 3: [Query][Response]

  Connection overhead paid once, amortized across requests.
```

## Performance Impact

```
Connection setup costs:
  TCP handshake:  1 RTT (~50ms intercontinental)
  TLS handshake:  1-2 RTT (~50-100ms)
  Auth/setup:     Varies

Without pooling (100ms RTT):
  1000 requests × 150ms setup = 150 seconds in overhead alone!

With pooling:
  10 connections × 150ms setup = 1.5 seconds
  Requests reuse existing connections.

10-100x improvement in connection overhead.
```

## Pool Configuration

```
Key parameters:

Min connections:
  Connections kept open even when idle.
  Ready for immediate use.

Max connections:
  Upper limit on concurrent connections.
  Prevents resource exhaustion.

Idle timeout:
  Close connections unused for this long.
  Frees resources, reduces stale connections.

Max lifetime:
  Close connections older than this.
  Prevents issues with long-lived connections.

Connection timeout:
  How long to wait for new connection.
  Fails fast if pool exhausted.
```

## Database Connection Pooling

```python
# Python with SQLAlchemy
from sqlalchemy import create_engine

engine = create_engine(
    'postgresql://user:pass@host/db',
    pool_size=5,          # Maintained connections
    max_overflow=10,      # Extra connections allowed
    pool_timeout=30,      # Wait for connection
    pool_recycle=3600,    # Recreate after 1 hour
    pool_pre_ping=True    # Test connections before use
)

# Each request borrows from pool
with engine.connect() as conn:
    result = conn.execute("SELECT 1")
# Connection returned to pool automatically
```

## HTTP Connection Pooling

```python
# Python requests with session (pooled)
import requests

# BAD: New connection per request
for url in urls:
    response = requests.get(url)  # New connection each time

# GOOD: Reuse connections via session
session = requests.Session()
adapter = requests.adapters.HTTPAdapter(
    pool_connections=10,
    pool_maxsize=20,
    max_retries=3
)
session.mount('https://', adapter)

for url in urls:
    response = session.get(url)  # Reuses connections
```

## Common Pooling Issues

### Pool Exhaustion

```
All connections in use, new requests must wait.

Symptoms:
  - Requests timeout waiting for connection
  - "Connection pool exhausted" errors
  - Latency spikes

Solutions:
  - Increase pool size
  - Reduce connection hold time
  - Add timeouts for borrowing
  - Monitor pool usage
```

### Connection Leaks

```
Connections borrowed but never returned.

Causes:
  - Exception before returning connection
  - Forgot to close/return connection
  - Infinite loop holding connection

Solutions:
  - Always use try-finally or context managers
  - Set connection timeouts
  - Monitor active vs. available connections
  - Implement leak detection
```

### Stale Connections

```
Connection in pool is dead (server closed it).

Causes:
  - Server timeout (closed idle connection)
  - Network issue
  - Server restart

Solutions:
  - Connection validation before use (pool_pre_ping)
  - Maximum connection lifetime
  - Proper error handling with retry
```

## Pool Sizing Guidelines

```
Too few connections:
  - Requests queue up
  - Increased latency
  - Underutilized backend

Too many connections:
  - Memory waste
  - May exceed server limits
  - Connection thrashing

Starting point:
  connections = (requests_per_second × avg_request_duration) × 1.5

Example:
  100 req/s × 0.1s duration = 10 concurrent
  Pool size: 15 (10 × 1.5)

Adjust based on monitoring!
```

## Monitoring Pool Health

```
Key metrics:
  - Active connections (in use)
  - Idle connections (available)
  - Wait time for connections
  - Connection creation rate
  - Timeout/exhaustion errors

Alerts:
  - Pool utilization > 80% sustained
  - Connection wait time > threshold
  - Pool exhaustion events
```
