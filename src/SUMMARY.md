# Summary

[Introduction](./introduction.md)

# Part I: Foundations

- [Network Fundamentals](./fundamentals/README.md)
  - [The OSI Model](./fundamentals/osi-model.md)
  - [The TCP/IP Stack](./fundamentals/tcp-ip-stack.md)
  - [Encapsulation](./fundamentals/encapsulation.md)
  - [Ports and Sockets](./fundamentals/ports-and-sockets.md)

- [The IP Layer](./ip-layer/README.md)
  - [IPv4 Addressing](./ip-layer/ipv4.md)
  - [IPv6 and the Future](./ip-layer/ipv6.md)
  - [Subnetting](./ip-layer/subnetting.md)
  - [Routing Fundamentals](./ip-layer/routing.md)
  - [IP Fragmentation](./ip-layer/fragmentation.md)

# Part II: Transport Layer Protocols

- [TCP Deep Dive](./tcp/README.md)
  - [The Three-Way Handshake](./tcp/handshake.md)
  - [TCP Header and Segments](./tcp/segments.md)
  - [Flow Control](./tcp/flow-control.md)
  - [Congestion Control](./tcp/congestion-control.md)
  - [Retransmission Mechanisms](./tcp/retransmission.md)
  - [TCP States and Lifecycle](./tcp/states.md)

- [UDP: The Simple Protocol](./udp/README.md)
  - [UDP Header and Datagrams](./udp/datagrams.md)
  - [When to Use UDP](./udp/use-cases.md)
  - [UDP vs TCP Trade-offs](./udp/tradeoffs.md)

# Part III: Application Layer Protocols

- [DNS: The Internet's Directory](./dns/README.md)
  - [DNS Resolution Process](./dns/resolution.md)
  - [Record Types](./dns/record-types.md)
  - [DNS Caching](./dns/caching.md)
  - [DNSSEC](./dns/dnssec.md)

- [HTTP Evolution](./http/README.md)
  - [HTTP/1.0 and HTTP/1.1](./http/http1.md)
  - [HTTP/2: Multiplexing Revolution](./http/http2.md)
  - [HTTP/3 and QUIC](./http/http3.md)

- [QUIC Protocol](./quic/README.md)
  - [Why QUIC Exists](./quic/motivation.md)
  - [Connection Establishment and 0-RTT](./quic/connection.md)
  - [Multiplexing Without Head-of-Line Blocking](./quic/multiplexing.md)
  - [Connection Migration](./quic/migration.md)

- [WebSockets](./websockets/README.md)
  - [The Upgrade Handshake](./websockets/handshake.md)
  - [Full-Duplex Communication](./websockets/communication.md)
  - [WebSocket Use Cases](./websockets/use-cases.md)

# Part IV: Security

- [TLS/SSL](./tls/README.md)
  - [The TLS Handshake](./tls/handshake.md)
  - [Certificates and PKI](./tls/certificates.md)
  - [Cipher Suites](./tls/cipher-suites.md)
  - [TLS 1.3 Improvements](./tls/tls13.md)

# Part V: Practical Applications

- [Application Protocols](./application-protocols/README.md)
  - [SMTP: Email Delivery](./application-protocols/smtp.md)
  - [FTP and Secure Alternatives](./application-protocols/ftp.md)
  - [SSH: Secure Shell](./application-protocols/ssh.md)

- [Protocol Design Principles](./protocol-design/README.md)
  - [Versioning Strategies](./protocol-design/versioning.md)
  - [Backwards Compatibility](./protocol-design/compatibility.md)
  - [Performance Considerations](./protocol-design/performance.md)

- [Real-World Patterns](./real-world/README.md)
  - [Load Balancing](./real-world/load-balancing.md)
  - [Proxies and Reverse Proxies](./real-world/proxies.md)
  - [CDNs](./real-world/cdns.md)
  - [Connection Pooling](./real-world/connection-pooling.md)

[Conclusion](./conclusion.md)
[Appendix: Tools and Debugging](./appendix-tools.md)
