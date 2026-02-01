# Protocol Design Principles

When building networked systems, you often need custom protocols or must extend existing ones. This chapter covers principles for designing protocols that are robust, evolvable, and performant.

## Design Considerations

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Protocol Design Questions                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Communication Pattern:                                             │
│    Request-response? Streaming? Pub-sub? Full-duplex?               │
│                                                                     │
│  Reliability Requirements:                                          │
│    Every message must arrive? Some loss acceptable?                 │
│                                                                     │
│  Latency Requirements:                                              │
│    Real-time? Best-effort? Batch acceptable?                        │
│                                                                     │
│  Message Size:                                                      │
│    Small fixed? Variable? Very large?                               │
│                                                                     │
│  Security:                                                          │
│    Authentication? Encryption? Integrity?                           │
│                                                                     │
│  Compatibility:                                                     │
│    Must work with existing systems? Future evolution?               │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

## Key Topics

1. **Versioning Strategies**: How to evolve protocols over time
2. **Backwards Compatibility**: Supporting old and new clients
3. **Performance Considerations**: Optimizing for speed and efficiency
