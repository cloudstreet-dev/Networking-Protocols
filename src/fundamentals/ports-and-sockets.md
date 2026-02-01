# Ports and Sockets

A single computer can run dozens of networked applications simultaneously—a web browser, email client, chat application, and more. How does the operating system route incoming data to the right application? The answer lies in **ports** and **sockets**.

## The Problem

Consider a server with IP address `192.168.1.100` running:
- A web server
- An SSH server
- A database
- An API service

When a packet arrives addressed to `192.168.1.100`, which application should receive it?

```
         Incoming Packets
              │
              ▼
    ┌─────────────────────┐
    │   IP: 192.168.1.100 │
    │                     │
    │   ??? Which app ??? │
    │                     │
    │   ┌───┐ ┌───┐ ┌───┐ │
    │   │Web│ │SSH│ │DB │ │
    │   └───┘ └───┘ └───┘ │
    └─────────────────────┘
```

## Ports: Application Addressing

**Ports** are 16-bit numbers (0-65535) that identify specific applications or services on a host. Combined with an IP address, a port uniquely identifies an application endpoint.

```
┌─────────────────────────────────────────────────────────────┐
│                    Port Number Space                        │
│                                                             │
│   0 ─────── 1023 ──────── 49151 ──────── 65535             │
│   │          │              │              │                │
│   │ Well-Known│  Registered │   Dynamic/   │                │
│   │   Ports   │    Ports    │   Private    │                │
│   │           │             │    Ports     │                │
│   │ (System)  │ (IANA reg)  │ (Ephemeral)  │                │
└─────────────────────────────────────────────────────────────┘
```

### Port Ranges

| Range | Name | Purpose |
|-------|------|---------|
| 0-1023 | Well-Known Ports | Reserved for standard services; require root/admin |
| 1024-49151 | Registered Ports | Can be registered with IANA for specific services |
| 49152-65535 | Dynamic/Private | Used for client-side ephemeral ports |

### Common Well-Known Ports

```
Port    Protocol    Service
────────────────────────────
20      TCP         FTP Data
21      TCP         FTP Control
22      TCP         SSH
23      TCP         Telnet
25      TCP         SMTP
53      TCP/UDP     DNS
67/68   UDP         DHCP
80      TCP         HTTP
110     TCP         POP3
143     TCP         IMAP
443     TCP         HTTPS
465     TCP         SMTPS
587     TCP         SMTP Submission
993     TCP         IMAPS
995     TCP         POP3S
3306    TCP         MySQL
5432    TCP         PostgreSQL
6379    TCP         Redis
27017   TCP         MongoDB
```

## How Ports Enable Multiplexing

With ports, our server can now direct traffic:

```
         Incoming Packets
              │
    ┌─────────┴─────────┐
    │    Check port     │
    └─────────┬─────────┘
              │
    ┌─────────┼─────────────────┬──────────────┐
    │         │                 │              │
    ▼         ▼                 ▼              ▼
┌───────┐ ┌───────┐       ┌───────┐      ┌───────┐
│Port 80│ │Port 22│       │Port   │      │Port   │
│ HTTP  │ │  SSH  │       │ 5432  │      │ 3000  │
│Server │ │Server │       │Postgre│      │  API  │
└───────┘ └───────┘       └───────┘      └───────┘
```

## Sockets: The Programming Interface

A **socket** is an endpoint for network communication. It's the API that applications use to send and receive data over the network.

### The Socket Tuple

A socket is uniquely identified by a 5-tuple:

```
┌─────────────────────────────────────────────────────────────┐
│                      Socket 5-Tuple                         │
├─────────────────────────────────────────────────────────────┤
│  1. Protocol        (TCP or UDP)                            │
│  2. Local IP        (192.168.1.100)                         │
│  3. Local Port      (80)                                    │
│  4. Remote IP       (10.0.0.50)                             │
│  5. Remote Port     (52431)                                 │
└─────────────────────────────────────────────────────────────┘

This combination uniquely identifies a connection.
```

### Why the Tuple Matters

Multiple connections can share the same local port:

```
Server listening on port 80 (192.168.1.100:80)

Connection 1: (TCP, 192.168.1.100, 80, 10.0.0.50, 52431)
Connection 2: (TCP, 192.168.1.100, 80, 10.0.0.50, 52432)
Connection 3: (TCP, 192.168.1.100, 80, 10.0.0.99, 41000)
              └─┬─┘ └──────┬─────┘ └┬┘ └────┬────┘ └──┬──┘
              Proto   Local IP   Local  Remote IP  Remote
                                 Port               Port

All three connections go to the same server port, but
each is a unique connection due to different remote endpoints.
```

This is how a web server can handle thousands of simultaneous connections on port 80.

## Socket Types

### Stream Sockets (SOCK_STREAM)

Used with TCP:
- Connection-oriented
- Reliable, ordered byte stream
- Most common for applications

```
Client                              Server
   │                                   │
   │────── connect() ─────────────────>│
   │                                   │ accept()
   │<──────────────────────────────────│
   │                                   │
   │═══════ Bidirectional Stream ══════│
   │                                   │
   │────── send(data) ────────────────>│
   │<───── send(response) ─────────────│
   │                                   │
   │────── close() ───────────────────>│
```

### Datagram Sockets (SOCK_DGRAM)

Used with UDP:
- Connectionless
- Individual messages (datagrams)
- No guarantee of delivery or order

```
Client                              Server
   │                                   │
   │────── sendto(data, addr) ────────>│
   │<───── sendto(response, addr) ─────│
   │                                   │
   │      (No connection state)        │
   │                                   │
   │────── sendto(data, addr) ────────>│
```

## Socket Programming Example

Here's a simple TCP server and client in Python:

### TCP Server

```python
import socket

# Create socket
server_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

# Allow address reuse (helpful during development)
server_socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)

# Bind to address and port
server_socket.bind(('0.0.0.0', 8080))

# Listen for connections (backlog of 5)
server_socket.listen(5)

print("Server listening on port 8080...")

while True:
    # Accept incoming connection
    client_socket, client_address = server_socket.accept()
    print(f"Connection from {client_address}")

    # Receive data
    data = client_socket.recv(1024)
    print(f"Received: {data.decode()}")

    # Send response
    client_socket.send(b"Hello from server!")

    # Close connection
    client_socket.close()
```

### TCP Client

```python
import socket

# Create socket
client_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

# Connect to server
client_socket.connect(('localhost', 8080))

# Send data
client_socket.send(b"Hello from client!")

# Receive response
response = client_socket.recv(1024)
print(f"Received: {response.decode()}")

# Close connection
client_socket.close()
```

## The Socket Lifecycle

### Server Side

```
┌─────────────────────────────────────────────────────────────┐
│                    Server Socket Lifecycle                   │
└─────────────────────────────────────────────────────────────┘

    socket()         Create the socket
        │
        ▼
    bind()           Assign local address and port
        │
        ▼
    listen()         Mark socket as passive (accepting connections)
        │
        ▼
    ┌──────────────────────────────────────┐
    │              accept()                │◄────┐
    │   (blocks until client connects)     │     │
    └──────────────┬───────────────────────┘     │
                   │                             │
                   ▼                             │
           New connected socket                  │
                   │                             │
        ┌──────────┴──────────┐                  │
        │                     │                  │
        ▼                     ▼                  │
    recv()/send()        spawn thread/          │
        │                handle async            │
        ▼                     │                  │
    close()                   │                  │
        │                     └──────────────────┘
        │
    (Handle next connection)
```

### Client Side

```
┌─────────────────────────────────────────────────────────────┐
│                    Client Socket Lifecycle                   │
└─────────────────────────────────────────────────────────────┘

    socket()         Create the socket
        │
        ▼
    connect()        Connect to remote server
        │            (OS assigns ephemeral local port)
        ▼
    send()/recv()    Exchange data
        │
        ▼
    close()          Terminate connection
```

## Ephemeral Ports

When a client connects to a server, the OS automatically assigns a **ephemeral port** (temporary port) for the client side:

```
Client                                      Server
┌─────────────┐                        ┌─────────────┐
│ 10.0.0.50   │                        │192.168.1.100│
│             │                        │             │
│ Port: ???   │── connect() ──────────>│ Port: 80    │
└─────────────┘                        └─────────────┘

OS assigns ephemeral port (e.g., 52431)

┌─────────────┐                        ┌─────────────┐
│ 10.0.0.50   │                        │192.168.1.100│
│             │                        │             │
│ Port: 52431 │<═══════════════════════│ Port: 80    │
└─────────────┘     Connection         └─────────────┘
```

### Ephemeral Port Range

Different systems use different ranges:

| OS | Default Range |
|----|---------------|
| Linux | 32768-60999 |
| Windows | 49152-65535 |
| macOS | 49152-65535 |

You can check and modify this on Linux:
```bash
$ cat /proc/sys/net/ipv4/ip_local_port_range
32768   60999

$ sudo sysctl -w net.ipv4.ip_local_port_range="10000 65535"
```

## Port Exhaustion

Each outbound connection uses an ephemeral port. If your application makes many outbound connections, you can exhaust available ports:

```
Problem Scenario:
─────────────────
Application makes 50,000 connections to an API server.
Each connection uses one ephemeral port.
Default range: 32768-60999 = ~28,000 ports

If connections aren't closed properly (lingering in TIME_WAIT),
you run out of ports!

Solutions:
─────────────────
1. Expand ephemeral port range
2. Enable TCP reuse options (SO_REUSEADDR, tcp_tw_reuse)
3. Use connection pooling
4. Properly close connections
```

## Viewing Port Usage

### Linux/macOS

```bash
# List all listening ports
$ netstat -tlnp
Proto Local Address    Foreign Address  State   PID/Program
tcp   0.0.0.0:22       0.0.0.0:*        LISTEN  1234/sshd
tcp   0.0.0.0:80       0.0.0.0:*        LISTEN  5678/nginx

# Or with ss (modern replacement)
$ ss -tlnp

# List all connections
$ netstat -anp | grep ESTABLISHED

# Show which process owns a port
$ lsof -i :80
```

### Windows

```powershell
# List all listening ports
netstat -an | findstr LISTENING

# Show process IDs
netstat -ano | findstr :80
```

## Special Port Behaviors

### Binding to 0.0.0.0

Binding to `0.0.0.0` means "all interfaces":

```
┌─────────────────────────────────────────────────────────────┐
│  Server with multiple interfaces                            │
│                                                             │
│  eth0: 192.168.1.100                                        │
│  eth1: 10.0.0.50                                            │
│  lo:   127.0.0.1                                            │
│                                                             │
│  bind('0.0.0.0', 80) → accepts on ALL interfaces            │
│  bind('192.168.1.100', 80) → accepts only on eth0           │
│  bind('127.0.0.1', 80) → accepts only on localhost          │
└─────────────────────────────────────────────────────────────┘
```

### Port 0

Binding to port 0 asks the OS to assign any available port:

```python
server_socket.bind(('0.0.0.0', 0))
actual_port = server_socket.getsockname()[1]
print(f"Assigned port: {actual_port}")  # e.g., 54321
```

### Reserved Ports (< 1024)

On Unix systems, ports below 1024 require root privileges:

```bash
$ python -c "import socket; s=socket.socket(); s.bind(('',80))"
PermissionError: [Errno 13] Permission denied

$ sudo python -c "import socket; s=socket.socket(); s.bind(('',80))"
# Works
```

This prevents unprivileged users from impersonating system services.

## Summary

- **Ports** (0-65535) identify applications on a host
- **Sockets** are the programming interface for network I/O
- A connection is uniquely identified by the **5-tuple**: (protocol, local IP, local port, remote IP, remote port)
- **Ephemeral ports** are automatically assigned for outbound connections
- Multiple connections can share a server port because remote endpoints differ

Understanding ports and sockets is essential for:
- Writing networked applications
- Debugging connectivity issues
- Understanding firewall rules
- Diagnosing port exhaustion problems

With the fundamentals covered, we're ready to dive into the IP layer—how data finds its way across the internet.
