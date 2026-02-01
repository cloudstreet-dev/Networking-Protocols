# Networking Protocols: A Developer's Guide

A comprehensive technical book covering network protocols from fundamentals to advanced patterns. Built with [mdbook](https://rust-lang.github.io/mdBook/).

**Read online:** https://cloudstreet-dev.github.io/Networking-Protocols/

## Topics Covered

- **Fundamentals** - OSI model, TCP/IP stack, encapsulation, ports and sockets
- **IP Layer** - IPv4, IPv6, subnetting, routing, fragmentation
- **TCP** - Three-way handshake, flow control, congestion control, retransmission
- **UDP** - Datagrams, use cases, trade-offs vs TCP
- **DNS** - Resolution process, record types, caching, DNSSEC
- **HTTP** - Evolution from HTTP/1.0 to HTTP/3
- **QUIC** - Connection establishment, multiplexing, connection migration
- **WebSockets** - Handshake, bidirectional communication, use cases
- **TLS/SSL** - Handshake, certificates, cipher suites, TLS 1.3
- **Application Protocols** - SMTP, FTP/SFTP, SSH
- **Protocol Design** - Versioning, backwards compatibility, performance
- **Real-World Patterns** - Load balancing, proxies, CDNs, connection pooling

## Building Locally

Install mdbook:

```bash
cargo install mdbook
```

Build and serve:

```bash
mdbook serve --open
```

The book will be available at http://localhost:3000.

## License

Copyright CloudStreet
