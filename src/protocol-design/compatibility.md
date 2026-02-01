# Backwards Compatibility

Maintaining backwards compatibility lets you evolve protocols without breaking existing deployments. It's often the difference between smooth upgrades and painful migrations.

## Compatibility Types

```
Backwards Compatible:
  New servers work with old clients.
  Client v1 ──────> Server v2 ✓

Forwards Compatible:
  Old servers handle new clients gracefully.
  Client v2 ──────> Server v1 ✓ (degraded)

Full Compatibility:
  Both directions work.
  Ideal but not always achievable.
```

## Techniques for Compatibility

### Ignore Unknown Fields

```json
// Client v1 sends:
{ "name": "Alice", "age": 30 }

// Server v2 expects:
{ "name": "Alice", "age": 30, "email": "?" }

// Server should:
// - Accept missing email (use default or null)
// - Not reject the request

// Client v2 sends:
{ "name": "Bob", "age": 25, "email": "bob@example.com" }

// Server v1 should:
// - Ignore unknown "email" field
// - Process name and age normally
```

### Optional Fields with Defaults

```protobuf
// Protocol Buffers example
message User {
  string name = 1;
  int32 age = 2;
  optional string email = 3;  // Added in v2
}

// Missing optional fields get default values.
// Old messages work with new code.
// New messages work with old code (email ignored).
```

### Extensible Enums

```
Bad: Fixed enum, no room to grow
  enum Status { OK = 0, ERROR = 1 }

Good: Reserve unknown handling
  enum Status {
    UNKNOWN = 0,  // Default for unrecognized
    OK = 1,
    ERROR = 2
    // Future: PENDING = 3
  }

Old code receiving new status → UNKNOWN (handled gracefully)
```

### Reserved Fields

```protobuf
message User {
  string name = 1;
  reserved 2;  // Was 'age', removed in v3
  string email = 3;
  reserved "age";  // Prevent reuse of name
}

// Prevents accidentally reusing field numbers
// which would cause data corruption.
```

## Breaking Changes

Sometimes breaking changes are necessary:

```
What's Breaking:
  - Removing required fields
  - Changing field types
  - Renaming fields (in JSON)
  - Changing semantics of existing fields
  - Removing supported message types

Mitigation Strategies:
  1. New endpoint/message type (keep old working)
  2. Deprecation period with warnings
  3. Version bump (v1 → v2)
  4. Feature flags during transition
```

## Postel's Law (Robustness Principle)

```
"Be conservative in what you send,
 be liberal in what you accept."

Send: Strictly conform to spec
Accept: Handle variations gracefully

This enables interoperability between
implementations with slight differences.
```

## Testing Compatibility

```bash
# Test old client against new server
$ old-client --server=new-server --test-suite

# Test new client against old server
$ new-client --server=old-server --test-suite

# Fuzz testing with version mixing
$ compatibility-fuzzer --versions=v1,v2,v3

# Contract testing
$ pact-verify --provider=server --consumer=client-v1
```

## Real-World Examples

```
JSON (excellent compatibility):
  - Unknown fields ignored
  - Missing fields → null/default
  - Easy to extend

Protocol Buffers (good compatibility):
  - Field numbers provide stability
  - Unknown fields preserved
  - Wire format stable

HTTP (exceptional compatibility):
  - 20+ years of evolution
  - HTTP/1.1 still works everywhere
  - Headers ignore unknown values
```
