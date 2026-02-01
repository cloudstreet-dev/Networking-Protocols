# Subnetting

**Subnetting** is the practice of dividing a larger network into smaller, logical segments. It's a fundamental skill for network design and one of the most practical topics in IP networking.

## Why Subnet?

Without subnetting, you'd have one flat network for all devices:

```
Single Network (No Subnetting):

┌─────────────────────────────────────────────────────────────┐
│   All 65,534 possible hosts on one network segment         │
│                                                             │
│   ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐ ... thousands │
│   │PC1│ │PC2│ │PC3│ │Srv│ │Dev│ │IoT│ │...│     more      │
│   └───┘ └───┘ └───┘ └───┘ └───┘ └───┘ └───┘               │
│                                                             │
│   Problems:                                                 │
│   - Broadcast traffic reaches everyone                      │
│   - No isolation between departments                        │
│   - Security is harder to manage                            │
│   - Single failure can affect everyone                      │
└─────────────────────────────────────────────────────────────┘
```

With subnetting:

```
Subnetted Network:

┌────────────────────┐ ┌────────────────────┐ ┌────────────────────┐
│   Engineering      │ │    Marketing       │ │      Servers       │
│   192.168.1.0/26   │ │   192.168.1.64/26  │ │  192.168.1.128/26  │
│                    │ │                    │ │                    │
│  ┌───┐ ┌───┐ ┌───┐ │ │  ┌───┐ ┌───┐      │ │  ┌───┐ ┌───┐      │
│  │PC1│ │PC2│ │PC3│ │ │  │PC4│ │PC5│      │ │  │Web│ │DB │      │
│  └───┘ └───┘ └───┘ │ │  └───┘ └───┘      │ │  └───┘ └───┘      │
│   62 usable hosts  │ │   62 usable hosts  │ │   62 usable hosts  │
└─────────┬──────────┘ └─────────┬──────────┘ └─────────┬──────────┘
          │                      │                      │
          └──────────────────────┼──────────────────────┘
                                 │
                            ┌────┴────┐
                            │ Router  │
                            └─────────┘
```

Benefits:
- **Broadcast containment**: Broadcasts stay within subnet
- **Security**: Apply different policies to different subnets
- **Organization**: Logical grouping of related systems
- **Performance**: Less broadcast traffic per segment
- **Troubleshooting**: Easier to isolate issues

## Understanding CIDR Notation

**CIDR (Classless Inter-Domain Routing)** notation specifies how many bits are used for the network portion:

```
192.168.1.0/24
            └── 24 network bits, 8 host bits

Binary breakdown:
Address:    11000000.10101000.00000001.00000000
            └────────── 24 bits ──────────┘└ 8 ┘
                     Network           Host

Subnet mask: 255.255.255.0
             11111111.11111111.11111111.00000000
```

### Common CIDR Blocks

| CIDR | Subnet Mask | Network Bits | Host Bits | Usable Hosts |
|------|-------------|--------------|-----------|--------------|
| /8 | 255.0.0.0 | 8 | 24 | 16,777,214 |
| /16 | 255.255.0.0 | 16 | 16 | 65,534 |
| /24 | 255.255.255.0 | 24 | 8 | 254 |
| /25 | 255.255.255.128 | 25 | 7 | 126 |
| /26 | 255.255.255.192 | 26 | 6 | 62 |
| /27 | 255.255.255.224 | 27 | 5 | 30 |
| /28 | 255.255.255.240 | 28 | 4 | 14 |
| /29 | 255.255.255.248 | 29 | 3 | 6 |
| /30 | 255.255.255.252 | 30 | 2 | 2 |
| /31 | 255.255.255.254 | 31 | 1 | 2* |
| /32 | 255.255.255.255 | 32 | 0 | 1 |

**/31 is special**: Used for point-to-point links (no broadcast needed).

### Why "Usable Hosts" Is Less Than 2^n

Two addresses in every subnet are reserved:
- **Network address**: All host bits = 0 (identifies the subnet)
- **Broadcast address**: All host bits = 1 (reaches all hosts)

```
192.168.1.0/24:
  Network:   192.168.1.0     (first address)
  Hosts:     192.168.1.1 - 192.168.1.254
  Broadcast: 192.168.1.255   (last address)

Usable = 2^(host bits) - 2 = 256 - 2 = 254
```

## Calculating Subnets

### Method 1: Binary Calculation

Given 192.168.1.0/24, create 4 subnets:

```
Step 1: Determine bits needed
  4 subnets = 2^2, so we need 2 additional network bits
  New prefix: /24 + 2 = /26

Step 2: Calculate subnet size
  Host bits = 32 - 26 = 6
  Addresses per subnet = 2^6 = 64
  Usable hosts = 64 - 2 = 62

Step 3: List subnets (increment by 64)
  Subnet 0: 192.168.1.0/26    (hosts .1-.62,   broadcast .63)
  Subnet 1: 192.168.1.64/26   (hosts .65-.126, broadcast .127)
  Subnet 2: 192.168.1.128/26  (hosts .129-.190, broadcast .191)
  Subnet 3: 192.168.1.192/26  (hosts .193-.254, broadcast .255)
```

### Method 2: The "Magic Number" Method

The "magic number" is 256 minus the last non-zero octet of the subnet mask:

```
For /26: Mask = 255.255.255.192
Magic number = 256 - 192 = 64

Subnets start at multiples of 64:
  192.168.1.0, 192.168.1.64, 192.168.1.128, 192.168.1.192
```

### Subnet Calculation Chart

```
┌──────────────────────────────────────────────────────────────────────┐
│  CIDR    Mask           Magic#   Subnets(from /24)  Hosts  Range    │
├──────────────────────────────────────────────────────────────────────┤
│  /25     255.255.255.128  128         2              126    /2       │
│  /26     255.255.255.192   64         4               62    /4       │
│  /27     255.255.255.224   32         8               30    /8       │
│  /28     255.255.255.240   16        16               14    /16      │
│  /29     255.255.255.248    8        32                6    /32      │
│  /30     255.255.255.252    4        64                2    /64      │
└──────────────────────────────────────────────────────────────────────┘
```

## Practical Examples

### Example 1: Office Network Design

**Requirement**: Design a network for a small office with:
- 50 employees (workstations)
- 10 servers
- 5 network devices
- Room for 50% growth

**Given**: 192.168.10.0/24

```
Solution:

Department        Hosts Needed   Subnet         Usable Range
────────────────────────────────────────────────────────────────
Workstations      50 (→75)       /25 (126)      192.168.10.0/25
                                                .1 - .126
Servers           10 (→15)       /27 (30)       192.168.10.128/27
                                                .129 - .158
Network Devices   5 (→8)         /28 (14)       192.168.10.160/28
                                                .161 - .174
Future Use        -              /28 (14)       192.168.10.176/28
Management        -              /28 (14)       192.168.10.192/28

Remaining: 192.168.10.208 - 192.168.10.255 (/28 + partial)
```

### Example 2: Finding Subnet for an IP

**Question**: What subnet does 192.168.1.147/26 belong to?

```
Step 1: Find the magic number
  /26 mask = 255.255.255.192
  Magic = 256 - 192 = 64

Step 2: Find which multiple of 64 contains .147
  0, 64, 128, 192...
  128 ≤ 147 < 192

Step 3: Answer
  Network: 192.168.1.128/26
  Range: 192.168.1.128 - 192.168.1.191
  Broadcast: 192.168.1.191
```

### Example 3: Are Two IPs on Same Subnet?

**Question**: Are 10.1.1.50/28 and 10.1.1.60/28 on the same subnet?

```
For /28: Magic number = 256 - 240 = 16

10.1.1.50: Falls in 10.1.1.48/28 (48 ≤ 50 < 64)
10.1.1.60: Falls in 10.1.1.48/28 (48 ≤ 60 < 64)

Answer: Yes, same subnet (10.1.1.48/28)
```

## VLSM (Variable Length Subnet Mask)

**VLSM** allows different subnets to have different sizes, optimizing address usage:

```
Without VLSM (Fixed /26):
┌──────────────────────────────────────────────────────────────┐
│  Dept A: 60 hosts    │  Dept B: 10 hosts   │  Links: 2 hosts │
│  /26 (62 usable) ✓   │  /26 (62 usable)    │  /26 (62 usable)│
│                      │  52 wasted!          │  60 wasted!     │
└──────────────────────────────────────────────────────────────┘

With VLSM:
┌──────────────────────────────────────────────────────────────┐
│  Dept A: 60 hosts    │  Dept B: 10 hosts   │  Links: 2 hosts │
│  /26 (62 usable) ✓   │  /28 (14 usable) ✓  │  /30 (2 usable)✓│
│                      │  4 spare             │  0 wasted       │
└──────────────────────────────────────────────────────────────┘
```

### VLSM Planning Process

1. **List requirements** from largest to smallest
2. **Assign subnets** starting with largest
3. **Use remaining space** for smaller subnets

```
Given: 172.16.0.0/16
Requirements:
  - Engineering: 500 hosts
  - Sales: 100 hosts
  - HR: 50 hosts
  - Point-to-point links: 4 (need 2 hosts each)

Allocation:
  Engineering:  172.16.0.0/23    (510 hosts)   172.16.0.1 - 172.16.1.254
  Sales:        172.16.2.0/25    (126 hosts)   172.16.2.1 - 172.16.2.126
  HR:           172.16.2.128/26  (62 hosts)    172.16.2.129 - 172.16.2.190
  Link 1:       172.16.2.192/30  (2 hosts)     172.16.2.193 - 172.16.2.194
  Link 2:       172.16.2.196/30  (2 hosts)     172.16.2.197 - 172.16.2.198
  Link 3:       172.16.2.200/30  (2 hosts)     172.16.2.201 - 172.16.2.202
  Link 4:       172.16.2.204/30  (2 hosts)     172.16.2.205 - 172.16.2.206

  Remaining:    172.16.2.208 - 172.16.255.255 (available for future)
```

## Supernetting (Route Aggregation)

**Supernetting** (or CIDR aggregation) combines multiple smaller networks into one larger route:

```
Before aggregation (4 routes):
  192.168.0.0/24
  192.168.1.0/24
  192.168.2.0/24
  192.168.3.0/24

After aggregation (1 route):
  192.168.0.0/22

This reduces routing table size and improves router efficiency.

Binary visualization:
  192.168.0.0  = 11000000.10101000.000000|00.00000000
  192.168.1.0  = 11000000.10101000.000000|01.00000000
  192.168.2.0  = 11000000.10101000.000000|10.00000000
  192.168.3.0  = 11000000.10101000.000000|11.00000000
                                  └──────┘
                                  These bits vary

  Common prefix: 22 bits → /22 covers all four
```

## IPv6 Subnetting

IPv6 subnetting is conceptually similar but the numbers are larger:

```
Standard allocation:
  ISP receives:     /32 or /48 from registry
  Organization gets: /48 from ISP
  Site/Subnet:      /64 (standard LAN)

/48 to /64 gives: 16 bits = 65,536 subnets
Each /64 has:     64 bits for hosts = 2^64 addresses

Example:
  Organization: 2001:db8:abcd::/48

  Subnets:
    2001:db8:abcd:0000::/64  - HQ Floor 1
    2001:db8:abcd:0001::/64  - HQ Floor 2
    2001:db8:abcd:0002::/64  - HQ Servers
    ...
    2001:db8:abcd:ffff::/64  - 65,536th subnet
```

## Tools for Subnetting

### Command Line

```bash
# ipcalc (Linux)
$ ipcalc 192.168.1.0/26
Address:   192.168.1.0          11000000.10101000.00000001.00 000000
Netmask:   255.255.255.192 = 26 11111111.11111111.11111111.11 000000
Wildcard:  0.0.0.63             00000000.00000000.00000000.00 111111
Network:   192.168.1.0/26       11000000.10101000.00000001.00 000000
HostMin:   192.168.1.1          11000000.10101000.00000001.00 000001
HostMax:   192.168.1.62         11000000.10101000.00000001.00 111110
Broadcast: 192.168.1.63         11000000.10101000.00000001.00 111111
Hosts/Net: 62

# sipcalc (more features)
$ sipcalc 192.168.1.0/24 -s 26
```

### Python

```python
import ipaddress

# Create network
network = ipaddress.ip_network('192.168.1.0/24')

# Get subnet info
print(f"Network: {network.network_address}")
print(f"Netmask: {network.netmask}")
print(f"Broadcast: {network.broadcast_address}")
print(f"Hosts: {network.num_addresses - 2}")

# Divide into subnets
subnets = list(network.subnets(new_prefix=26))
for subnet in subnets:
    print(f"  {subnet}")
# Output:
#   192.168.1.0/26
#   192.168.1.64/26
#   192.168.1.128/26
#   192.168.1.192/26

# Check if IP is in network
ip = ipaddress.ip_address('192.168.1.100')
print(ip in network)  # True
```

## Common Mistakes

1. **Forgetting reserved addresses**
   - Always subtract 2 from total for usable hosts

2. **Overlapping subnets**
   - 192.168.1.0/25 and 192.168.1.64/26 overlap!
   - Plan carefully, especially with VLSM

3. **Not planning for growth**
   - Networks grow; leave room for expansion

4. **Using /30 for LANs**
   - /30 is for point-to-point links only
   - LANs need room for multiple hosts

## Summary

Subnetting divides networks for better organization, security, and efficiency:

- **CIDR notation** (/24) indicates network vs. host bits
- **Subnet mask** shows the network boundary
- **Magic number** (256 - mask octet) gives subnet size
- **VLSM** allows different-sized subnets for efficiency
- **Supernetting** aggregates routes for simpler routing

Practice is key—work through examples until it becomes intuitive.

Next, we'll explore how packets actually find their way across networks: routing fundamentals.
