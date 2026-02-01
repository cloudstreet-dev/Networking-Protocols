# Application Protocols

Beyond HTTP, many other application protocols power essential internet services. Understanding them provides insight into protocol design and helps when integrating with these systems.

## Common Application Protocols

```
┌─────────────────────────────────────────────────────────────────────┐
│               Major Application Protocols                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Email:                                                             │
│    SMTP (25, 587)     Sending email between servers                 │
│    IMAP (143, 993)    Accessing mailbox, server stores mail         │
│    POP3 (110, 995)    Downloading mail, client stores               │
│                                                                     │
│  File Transfer:                                                     │
│    FTP (21, 20)       Classic file transfer (insecure)              │
│    SFTP (22)          SSH-based file transfer (secure)              │
│    SCP (22)           Secure copy over SSH                          │
│                                                                     │
│  Remote Access:                                                     │
│    SSH (22)           Secure shell, tunneling, file transfer        │
│    Telnet (23)        Insecure remote access (legacy)               │
│    RDP (3389)         Windows remote desktop                        │
│                                                                     │
│  Name Resolution:                                                   │
│    DNS (53)           Domain name → IP address                      │
│    mDNS (5353)        Multicast DNS (local discovery)               │
│                                                                     │
│  Time:                                                              │
│    NTP (123)          Network time synchronization                  │
│                                                                     │
│  Directory:                                                         │
│    LDAP (389, 636)    Directory services (Active Directory)         │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

## Protocol Characteristics

Most application protocols share common traits:

```
Request-Response:
  Client sends command/request
  Server sends response
  Back and forth until done

Text vs Binary:
  Text:   Human-readable (SMTP, HTTP/1.1)
  Binary: Machine-efficient (HTTP/2, Protocol Buffers)

Stateful vs Stateless:
  Stateful:  Server remembers session (SMTP, FTP)
  Stateless: Each request independent (HTTP, DNS)
```

## What You'll Learn

1. **SMTP**: How email travels across the internet
2. **FTP and Alternatives**: File transfer evolution
3. **SSH**: Secure remote access and more
