# The OSI Model

The **Open Systems Interconnection (OSI) model** is a conceptual framework that standardizes how network communication should be organized. Created by the International Organization for Standardization (ISO) in 1984, it divides networking into seven distinct layers.

## The Seven Layers

```
┌───────────────────────────────────────────────────────────────┐
│  Layer 7: Application     │  HTTP, FTP, SMTP, DNS            │
├───────────────────────────────────────────────────────────────┤
│  Layer 6: Presentation    │  Encryption, Compression, Format │
├───────────────────────────────────────────────────────────────┤
│  Layer 5: Session         │  Session Management, RPC         │
├───────────────────────────────────────────────────────────────┤
│  Layer 4: Transport       │  TCP, UDP                        │
├───────────────────────────────────────────────────────────────┤
│  Layer 3: Network         │  IP, ICMP, Routing               │
├───────────────────────────────────────────────────────────────┤
│  Layer 2: Data Link       │  Ethernet, WiFi, MAC addresses   │
├───────────────────────────────────────────────────────────────┤
│  Layer 1: Physical        │  Cables, Radio waves, Voltages   │
└───────────────────────────────────────────────────────────────┘
```

### Layer 1: Physical Layer

The physical layer deals with the actual transmission of raw bits over a physical medium.

**Responsibilities:**
- Defining physical connectors and cables
- Encoding bits as electrical signals, light pulses, or radio waves
- Specifying transmission rates (bandwidth)
- Managing physical topology (how devices connect)

**Examples:**
- Ethernet cables (Cat5, Cat6)
- Fiber optic cables
- WiFi radio signals
- USB connections

**What it looks like:**
```
Bit stream: 10110010 01001101 11010010 ...
            ↓
Physical:   ▁▁▔▔▁▔▁▁ ▁▔▁▁▔▔▁▔ ▔▔▁▔▁▁▔▁
            (voltage levels on copper wire)
```

### Layer 2: Data Link Layer

The data link layer handles communication between directly connected devices on the same network segment.

**Responsibilities:**
- Framing: Organizing bits into frames
- MAC (Media Access Control) addressing
- Error detection (not correction)
- Flow control between adjacent nodes

**Key Concepts:**
- **MAC Address**: A 48-bit hardware address (e.g., `00:1A:2B:3C:4D:5E`)
- **Frame**: The unit of data at this layer

**Ethernet Frame Structure:**
```
┌──────────┬──────────┬──────┬─────────────────┬─────┐
│ Dest MAC │ Src MAC  │ Type │     Payload     │ FCS │
│ (6 bytes)│ (6 bytes)│(2 B) │  (46-1500 B)    │(4 B)│
└──────────┴──────────┴──────┴─────────────────┴─────┘

FCS = Frame Check Sequence (error detection)
```

### Layer 3: Network Layer

The network layer enables communication across different networks—it's what makes "inter-networking" (the Internet) possible.

**Responsibilities:**
- Logical addressing (IP addresses)
- Routing packets between networks
- Fragmentation and reassembly
- Quality of Service (QoS)

**Key Protocols:**
- **IP** (Internet Protocol): The primary protocol
- **ICMP** (Internet Control Message Protocol): Error reporting and diagnostics
- **ARP** (Address Resolution Protocol): Maps IP to MAC addresses

```
Routing Decision:

   Source: 192.168.1.100
   Destination: 8.8.8.8

   Is destination on local network? NO
   ↓
   Send to default gateway (router)
   ↓
   Router examines destination, forwards to next hop
   ↓
   Process repeats until packet reaches destination
```

### Layer 4: Transport Layer

The transport layer provides end-to-end communication services, handling the complexities of reliable data transfer.

**Responsibilities:**
- Segmentation and reassembly
- Connection management
- Reliability (for TCP)
- Flow control
- Multiplexing via ports

**Key Protocols:**
- **TCP** (Transmission Control Protocol): Reliable, ordered delivery
- **UDP** (User Datagram Protocol): Fast, connectionless delivery

```
Port Multiplexing:

Single IP address, multiple applications:

   IP: 192.168.1.100
   ├── Port 80:   Web Server
   ├── Port 443:  HTTPS Server
   ├── Port 22:   SSH Server
   └── Port 3000: Development Server
```

### Layer 5: Session Layer

The session layer manages sessions—ongoing dialogues between applications.

**Responsibilities:**
- Establishing, maintaining, and terminating sessions
- Session checkpointing and recovery
- Synchronization

**In Practice:**
This layer is often merged with the application layer in real implementations. Few protocols exist purely at this layer.

**Examples:**
- NetBIOS
- RPC (Remote Procedure Call)
- Session tokens in web applications (conceptually)

### Layer 6: Presentation Layer

The presentation layer handles data representation—how information is formatted, encoded, and encrypted.

**Responsibilities:**
- Data translation between formats
- Encryption and decryption
- Compression and decompression
- Character encoding (ASCII, UTF-8)

**In Practice:**
Like the session layer, this is often absorbed into the application layer. TLS can be considered a presentation layer protocol.

**Examples:**
- SSL/TLS (encryption)
- JPEG, GIF (image formatting)
- MIME types

### Layer 7: Application Layer

The application layer is where network applications and their protocols operate. This is the layer developers interact with most directly.

**Responsibilities:**
- Providing network services to applications
- User authentication
- Resource sharing

**Examples:**
- HTTP/HTTPS (web)
- SMTP, POP3, IMAP (email)
- FTP, SFTP (file transfer)
- DNS (name resolution)
- SSH (secure shell)

## How Data Flows Through Layers

When you send data, it travels down the stack on your machine, across the network, and up the stack on the destination:

```
   Sender                                    Receiver
┌───────────┐                            ┌───────────┐
│Application│ ──────────────────────────>│Application│
├───────────┤                            ├───────────┤
│Presentation─────────────────────────────Presentation
├───────────┤                            ├───────────┤
│  Session  │ ────────────────────────────  Session  │
├───────────┤                            ├───────────┤
│ Transport │ ────────────────────────────│ Transport │
├───────────┤                            ├───────────┤
│  Network  │ ────────────────────────────│  Network  │
├───────────┤      ┌─────────┐           ├───────────┤
│ Data Link │ ─────│ Router  │───────────│ Data Link │
├───────────┤      └─────────┘           ├───────────┤
│ Physical  │ ═══════════════════════════│ Physical  │
└───────────┘     Physical Medium        └───────────┘
```

Each layer adds its own header (and sometimes trailer) to the data—a process called encapsulation.

## OSI in the Real World

Here's an important truth: **the OSI model is a teaching tool, not a strict blueprint**.

The internet wasn't built on OSI—it was built on TCP/IP, which predates OSI and uses a simpler four-layer model. Real protocols often don't fit neatly into single layers:

- **TLS** spans presentation and session layers
- **HTTP** is application layer but handles some session concerns
- **TCP** handles some session-layer functions

The OSI model is valuable for:
- Learning and discussing networking concepts
- Troubleshooting ("Is this a Layer 2 or Layer 3 problem?")
- Understanding where protocols fit conceptually

But don't expect real-world protocols to follow it rigidly.

## Memorization Tricks

Many people use mnemonics to remember the layers. From Layer 1 to 7:

- **P**lease **D**o **N**ot **T**hrow **S**ausage **P**izza **A**way
- **P**hysical, **D**ata Link, **N**etwork, **T**ransport, **S**ession, **P**resentation, **A**pplication

Or from 7 to 1:
- **A**ll **P**eople **S**eem **T**o **N**eed **D**ata **P**rocessing

## Summary

The OSI model provides a framework for understanding network communication:

| Layer | Name | Key Function | Example |
|-------|------|--------------|---------|
| 7 | Application | User interface to network | HTTP, DNS |
| 6 | Presentation | Data formatting | TLS, JPEG |
| 5 | Session | Dialog management | RPC |
| 4 | Transport | End-to-end delivery | TCP, UDP |
| 3 | Network | Routing between networks | IP |
| 2 | Data Link | Local network delivery | Ethernet |
| 1 | Physical | Bit transmission | Cables, WiFi |

In the next section, we'll look at the TCP/IP model—what the internet actually uses.
