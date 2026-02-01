# IPv4 Addressing

**IPv4** (Internet Protocol version 4) has been the backbone of the internet since 1981. Despite its age and limitations, it still carries the majority of internet traffic.

## The IPv4 Address

An IPv4 address is a 32-bit number, typically written as four decimal numbers separated by dots (dotted-decimal notation):

```
Binary:    11000000 10101000 00000001 01100100
           └──┬───┘ └──┬───┘ └──┬───┘ └──┬───┘
Decimal:     192   .  168   .   1    .  100

Each number (octet) ranges from 0-255 (8 bits)
Total: 4 octets × 8 bits = 32 bits
```

### Address Space Size

32 bits gives us 2³² = **4,294,967,296** addresses. Sounds like a lot, but:
- Many are reserved for special purposes
- Allocation was historically wasteful
- Every device needs an address (phones, IoT, servers...)

We ran out of new IPv4 blocks in 2011.

## The IPv4 Header

Every IP packet starts with a header containing routing and handling information:

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
├─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┤
│Version│  IHL  │    DSCP   │ECN│         Total Length          │
├───────┴───────┴───────────┴───┼───────┬───────────────────────┤
│         Identification        │ Flags │    Fragment Offset    │
├───────────────┬───────────────┼───────┴───────────────────────┤
│      TTL      │   Protocol    │        Header Checksum        │
├───────────────┴───────────────┴───────────────────────────────┤
│                       Source IP Address                       │
├───────────────────────────────────────────────────────────────┤
│                    Destination IP Address                     │
├───────────────────────────────────────────────────────────────┤
│                    Options (if IHL > 5)                       │
└───────────────────────────────────────────────────────────────┘

Minimum header size: 20 bytes (no options)
Maximum header size: 60 bytes (with options)
```

### Key Header Fields

| Field | Size | Purpose |
|-------|------|---------|
| Version | 4 bits | IP version (4 for IPv4) |
| IHL | 4 bits | Header length in 32-bit words |
| DSCP/ECN | 8 bits | Quality of Service hints |
| Total Length | 16 bits | Packet size (header + data) |
| Identification | 16 bits | Unique ID for fragmentation |
| Flags | 3 bits | Fragmentation control |
| Fragment Offset | 13 bits | Position in fragmented packet |
| TTL | 8 bits | Hop limit (prevents loops) |
| Protocol | 8 bits | Upper layer protocol (TCP=6, UDP=17) |
| Header Checksum | 16 bits | Error detection for header |
| Source IP | 32 bits | Sender's address |
| Destination IP | 32 bits | Receiver's address |

## Address Classes (Historical)

Originally, IPv4 used a classful addressing scheme:

```
Class A: 0xxxxxxx.xxxxxxxx.xxxxxxxx.xxxxxxxx
         │└──────────────┬───────────────────┘
         Network (8 bits)    Host (24 bits)
         Range: 1.0.0.0 - 126.255.255.255
         Networks: 126    Hosts/Network: 16 million

Class B: 10xxxxxx.xxxxxxxx.xxxxxxxx.xxxxxxxx
         └───────┬────────┘└───────┬────────┘
         Network (16 bits)   Host (16 bits)
         Range: 128.0.0.0 - 191.255.255.255
         Networks: 16,384  Hosts/Network: 65,534

Class C: 110xxxxx.xxxxxxxx.xxxxxxxx.xxxxxxxx
         └──────────┬──────────────┘└───┬───┘
         Network (24 bits)         Host (8 bits)
         Range: 192.0.0.0 - 223.255.255.255
         Networks: 2 million  Hosts/Network: 254

Class D: 1110xxxx.xxxxxxxx.xxxxxxxx.xxxxxxxx
         Multicast addresses (224.0.0.0 - 239.255.255.255)

Class E: 1111xxxx.xxxxxxxx.xxxxxxxx.xxxxxxxx
         Reserved/Experimental (240.0.0.0 - 255.255.255.255)
```

**This system is obsolete.** It was too inflexible—an organization needing 300 addresses had to get a Class B (65,534 addresses) because Class C was too small (254). This wasted addresses. Modern networks use CIDR (classless addressing) instead.

## Special and Reserved Addresses

Several address ranges have special meanings:

```
┌──────────────────────────────────────────────────────────────┐
│  Address Range        │  Purpose                             │
├──────────────────────────────────────────────────────────────┤
│  0.0.0.0/8            │  "This network" / unspecified        │
│  10.0.0.0/8           │  Private network (Class A)           │
│  127.0.0.0/8          │  Loopback (localhost)                │
│  169.254.0.0/16       │  Link-local (auto-config)            │
│  172.16.0.0/12        │  Private network (Class B range)     │
│  192.168.0.0/16       │  Private network (Class C range)     │
│  224.0.0.0/4          │  Multicast                           │
│  255.255.255.255      │  Broadcast                           │
└──────────────────────────────────────────────────────────────┘
```

### Private Addresses (RFC 1918)

Three ranges are designated for private use—they're not routable on the public internet:

```
10.0.0.0     - 10.255.255.255    (10.0.0.0/8)      16 million addresses
172.16.0.0   - 172.31.255.255    (172.16.0.0/12)   1 million addresses
192.168.0.0  - 192.168.255.255   (192.168.0.0/16)  65,536 addresses
```

Your home network almost certainly uses one of these ranges (typically 192.168.x.x). To reach the internet, your router performs NAT (Network Address Translation).

### Loopback Address

`127.0.0.1` (or any 127.x.x.x) is the **loopback address**. Traffic sent here never leaves your machine—it's used for local testing:

```bash
$ ping 127.0.0.1
PING 127.0.0.1: 64 bytes, seq=0 time=0.054 ms

# Same as:
$ ping localhost
```

### Broadcast Address

`255.255.255.255` is the **limited broadcast** address. Packets sent here go to all devices on the local network segment.

Each network also has a **directed broadcast** address (the highest address in the range). For 192.168.1.0/24, the broadcast is 192.168.1.255.

## Network vs. Host Portions

An IP address has two parts:

```
       192.168.1.100
       └───┬───┘└┬┘
       Network  Host
       Portion  Portion

The division is determined by the subnet mask.
```

The **network portion** identifies which network a host belongs to. The **host portion** identifies the specific device on that network.

### Subnet Mask

A **subnet mask** indicates how many bits are network vs. host:

```
IP Address:     192.168.1.100   = 11000000.10101000.00000001.01100100
Subnet Mask:    255.255.255.0   = 11111111.11111111.11111111.00000000
                                  └────────── Network ─────────────┘└ Host ┘

AND them together to get the network address:
Network:        192.168.1.0     = 11000000.10101000.00000001.00000000
```

### CIDR Notation

**CIDR (Classless Inter-Domain Routing)** notation appends a slash and the number of network bits:

```
192.168.1.100/24
             └── 24 bits for network = 255.255.255.0 mask

Common CIDR blocks:
/8   = 255.0.0.0       = 16,777,214 hosts
/16  = 255.255.0.0     = 65,534 hosts
/24  = 255.255.255.0   = 254 hosts
/32  = 255.255.255.255 = 1 host (single address)
```

## Determining If Two Hosts Are on the Same Network

Hosts on the same network can communicate directly. Hosts on different networks need a router.

```
Host A: 192.168.1.100/24
Host B: 192.168.1.200/24
Host C: 192.168.2.50/24

Apply mask to each:
A network: 192.168.1.100 AND 255.255.255.0 = 192.168.1.0
B network: 192.168.1.200 AND 255.255.255.0 = 192.168.1.0
C network: 192.168.2.50  AND 255.255.255.0 = 192.168.2.0

A and B: Same network (192.168.1.0) → Direct communication
A and C: Different networks → Need router
```

## NAT (Network Address Translation)

With private addresses and limited IPv4 space, **NAT** lets many devices share one public IP:

```
Private Network (192.168.1.0/24)          Internet
┌─────────────────────────────────┐
│  ┌─────────┐                    │     ┌─────────────────┐
│  │ Laptop  │                    │     │                 │
│  │ .100    ├──┐                 │     │   Web Server    │
│  └─────────┘  │    ┌─────────┐  │     │  93.184.216.34  │
│               ├────┤ Router  ├──┼────>│                 │
│  ┌─────────┐  │    │  NAT    │  │     │                 │
│  │  Phone  ├──┘    │         │  │     └─────────────────┘
│  │ .101    │       │ Public: │  │
│  └─────────┘       │73.45.2.1│  │
│                    └─────────┘  │
└─────────────────────────────────┘

Laptop sends: src=192.168.1.100:52000 dst=93.184.216.34:80
NAT rewrites: src=73.45.2.1:40123    dst=93.184.216.34:80

Response comes back to 73.45.2.1:40123
NAT looks up mapping, forwards to 192.168.1.100:52000
```

NAT is why billions of devices can use the internet with only ~4 billion addresses.

## Working with IP Addresses in Code

### Python

```python
import ipaddress

# Parse an address
ip = ipaddress.ip_address('192.168.1.100')
print(ip.is_private)      # True
print(ip.is_loopback)     # False

# Work with networks
network = ipaddress.ip_network('192.168.1.0/24')
print(network.num_addresses)  # 256
print(network.netmask)        # 255.255.255.0

# Check if address is in network
ip = ipaddress.ip_address('192.168.1.100')
print(ip in network)          # True

# Iterate over hosts
for host in network.hosts():
    print(host)  # 192.168.1.1 through 192.168.1.254
```

### Bash

```bash
# Get your IP addresses
$ ip addr show
# or
$ ifconfig

# Check if you can reach an IP
$ ping -c 3 192.168.1.1

# Trace route to destination
$ traceroute 8.8.8.8

# Look up your public IP
$ curl ifconfig.me
```

## Practical Tips

### Finding Your IP Address

```bash
# Linux/Mac - local IP
$ hostname -I
192.168.1.100

# Windows - local IP
> ipconfig

# Public IP (what the internet sees)
$ curl ifconfig.me
```

### Common Issues

**"Network is unreachable"**
- Check if you have an IP (DHCP may have failed)
- Check subnet mask is correct
- Check default gateway is set

**"No route to host"**
- Destination may be down
- Firewall may be blocking
- ARP resolution may have failed

**"Connection refused"**
- You reached the host, but no service is listening
- This is a good sign for network debugging—networking works!

## Summary

IPv4's 32-bit addressing scheme, while showing its age, remains the internet's foundation:

- Addresses are written as four octets (e.g., 192.168.1.100)
- Network and host portions are determined by the subnet mask
- Private ranges (10.x, 172.16-31.x, 192.168.x) are for internal use
- NAT allows address sharing but adds complexity
- CIDR replaced wasteful classful addressing

The address shortage led to IPv6, which we'll cover next.
