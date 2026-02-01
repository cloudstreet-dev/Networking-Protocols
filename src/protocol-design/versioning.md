# Versioning Strategies

Protocols evolve. New features are added, bugs are fixed, and requirements change. Good versioning makes this evolution manageable.

## Why Version?

```
Without versioning:

  Client (v1): Send message type A
  Server (v2): Expects message type B
  Result: Confusion, errors, failures

With versioning:

  Client (v1): "I speak version 1"
  Server (v2): "I understand v1 and v2, let's use v1"
  Result: Graceful interoperability
```

## Versioning Approaches

### Explicit Version Numbers

```
In protocol header:
┌──────────────────────────────────────────────────────────────┐
│ Version │ Message Type │ Length │ Payload...                 │
│   (1)   │     (2)      │  (4)   │                            │
└──────────────────────────────────────────────────────────────┘

HTTP:
  GET / HTTP/1.1
  GET / HTTP/2

Pros: Clear, explicit
Cons: Major versions can break compatibility
```

### Feature Negotiation

```
Instead of single version, negotiate capabilities:

Client: "I support: compression, encryption, batch"
Server: "I support: encryption, streaming"
Both: "We'll use: encryption"

TLS does this with cipher suites.
HTTP/2 does this with SETTINGS frames.

Pros: Granular, flexible
Cons: Complex negotiation
```

### Semantic Versioning

```
MAJOR.MINOR.PATCH

Major: Breaking changes (v1 → v2)
Minor: New features, backwards compatible (v1.1 → v1.2)
Patch: Bug fixes only (v1.1.0 → v1.1.1)

For APIs:
  v1 clients work with v1.x servers
  v2 might require migration

Pros: Clear expectations
Cons: Major bumps still painful
```

### Wire Format Versioning

```
Message format evolution:

Version 1:
  { "name": "Alice", "age": 30 }

Version 2 (additive):
  { "name": "Alice", "age": 30, "email": "alice@example.com" }

Old clients ignore unknown fields.
New clients handle missing fields.
No version number needed if done carefully.
```

## Version in Different Layers

```
URL versioning (REST APIs):
  /api/v1/users
  /api/v2/users

Header versioning:
  Accept: application/vnd.myapi.v2+json

Query parameter:
  /api/users?version=2

Content negotiation:
  Accept: application/json; version=2
```

## Best Practices

```
1. Include version from day one
   Adding versioning later is painful.

2. Plan for evolution
   Reserve bits/fields for future use.

3. Support multiple versions
   Don't force immediate upgrades.

4. Deprecation timeline
   v1 supported until 2025-01-01.

5. Version at right granularity
   API version? Message version? Both?
```
