# TCP Header and Segments

Understanding the TCP header is essential for debugging network issues, interpreting packet captures, and grasping how TCP works. Every TCP segment starts with a header containing control information.

## TCP Segment Structure

A TCP **segment** is the unit of data at the transport layer:

```
┌─────────────────────────────────────────────────────────────┐
│                      TCP Segment                            │
├──────────────────────────────┬──────────────────────────────┤
│         TCP Header           │         TCP Payload          │
│        (20-60 bytes)         │     (0 to MSS bytes)         │
└──────────────────────────────┴──────────────────────────────┘
```

Segments are encapsulated in IP packets:

```
┌─────────────────────────────────────────────────────────────┐
│  IP Header   │  TCP Header   │        TCP Payload           │
│  (20 bytes)  │ (20-60 bytes) │       (application data)     │
└─────────────────────────────────────────────────────────────┘
```

## The TCP Header

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
├─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┤
│          Source Port          │       Destination Port        │
├───────────────────────────────┴───────────────────────────────┤
│                        Sequence Number                        │
├───────────────────────────────────────────────────────────────┤
│                    Acknowledgment Number                      │
├───────┬───────┬─┬─┬─┬─┬─┬─┬─┬─┬───────────────────────────────┤
│  Data │       │C│E│U│A│P│R│S│F│                               │
│ Offset│ Rsrvd │W│C│R│C│S│S│Y│I│            Window             │
│       │       │R│E│G│K│H│T│N│N│                               │
├───────┴───────┴─┴─┴─┴─┴─┴─┴─┴─┼───────────────────────────────┤
│           Checksum            │        Urgent Pointer         │
├───────────────────────────────┴───────────────────────────────┤
│                    Options (if Data Offset > 5)               │
│                             ...                               │
├───────────────────────────────────────────────────────────────┤
│                           Payload                             │
│                             ...                               │
└───────────────────────────────────────────────────────────────┘
```

## Header Fields Explained

### Source Port and Destination Port (16 bits each)

```
Source Port:      The sender's port (often ephemeral)
Destination Port: The receiver's port (often well-known)

Together with IP addresses, these form the connection 5-tuple:
(Protocol, Source IP, Source Port, Dest IP, Dest Port)

Example:
  Client → Server HTTP request:
    Source: 192.168.1.100:52431
    Dest:   93.184.216.34:80
```

### Sequence Number (32 bits)

```
Identifies the position of data in the byte stream.

If Sequence = 1000 and Payload = 100 bytes:
  This segment contains bytes 1000-1099

First SYN: Sequence = ISN (Initial Sequence Number)
           Subsequent: ISN + bytes sent

32 bits → wraps around after ~4GB
  (Timestamps help disambiguate in fast networks)
```

### Acknowledgment Number (32 bits)

```
"I've received all bytes up to this number - 1"
"Send me this byte next"

If ACK = 1100:
  "I have bytes 0-1099, expecting 1100"

Only valid when ACK flag is set.
```

### Data Offset (4 bits)

```
Length of TCP header in 32-bit words.

Minimum: 5 (5 × 4 = 20 bytes, no options)
Maximum: 15 (15 × 4 = 60 bytes, 40 bytes of options)

Tells receiver where the payload begins.
```

### Reserved (4 bits)

```
Reserved for future use. Must be zero.
(Historically 6 bits, 2 repurposed for CWR/ECE)
```

### Control Flags (8 bits)

Each flag is 1 bit:

```
┌─────┬─────────────────────────────────────────────────────────┐
│ CWR │ Congestion Window Reduced                               │
│     │ Sender reduced congestion window                        │
├─────┼─────────────────────────────────────────────────────────┤
│ ECE │ ECN-Echo                                                │
│     │ Congestion notification echo                            │
├─────┼─────────────────────────────────────────────────────────┤
│ URG │ Urgent pointer field is valid                           │
│     │ (Rarely used today)                                     │
├─────┼─────────────────────────────────────────────────────────┤
│ ACK │ Acknowledgment field is valid                           │
│     │ Set on almost every segment after SYN                   │
├─────┼─────────────────────────────────────────────────────────┤
│ PSH │ Push - deliver data immediately to application          │
│     │ Don't buffer waiting for more data                      │
├─────┼─────────────────────────────────────────────────────────┤
│ RST │ Reset - abort the connection immediately                │
│     │ Something went wrong                                    │
├─────┼─────────────────────────────────────────────────────────┤
│ SYN │ Synchronize - connection establishment                  │
│     │ Only set during handshake                               │
├─────┼─────────────────────────────────────────────────────────┤
│ FIN │ Finish - sender is done sending                         │
│     │ Graceful connection termination                         │
└─────┴─────────────────────────────────────────────────────────┘
```

Common flag combinations:

```
SYN           = Connection request
SYN + ACK     = Connection accepted
ACK           = Data or acknowledgment
PSH + ACK     = Push data (common for requests)
FIN + ACK     = Done sending, acknowledging
RST           = Connection abort
RST + ACK     = Reset with acknowledgment
```

### Window (16 bits)

```
Receive window size: "I can accept this many more bytes"

Range: 0 - 65535 bytes

With window scaling (negotiated in SYN):
  Actual window = Window × 2^scale

Example with scale=7:
  Window=512 means 512 × 128 = 65536 bytes
```

### Checksum (16 bits)

```
Covers header, data, and a pseudo-header:

┌─────────────────────────────────────────────────────────────┐
│                     Pseudo-Header                           │
├─────────────────────────────────────────────────────────────┤
│  Source IP Address (from IP header)                         │
│  Destination IP Address (from IP header)                    │
│  Zero | Protocol (6 for TCP) | TCP Length                   │
└─────────────────────────────────────────────────────────────┘

Why pseudo-header?
- Ensures segment reaches correct destination
- Protects against IP address spoofing
```

### Urgent Pointer (16 bits)

```
Offset to end of urgent data (if URG flag set).

Largely obsolete - rarely used in modern applications.
Was intended for out-of-band signaling.
```

## TCP Options

Options extend the header beyond 20 bytes:

```
Option Format:
┌─────────┬────────┬──────────────────────┐
│  Kind   │ Length │        Data          │
│ (1 byte)│(1 byte)│   (Length-2 bytes)   │
└─────────┴────────┴──────────────────────┘

Single-byte options: Kind only (no length/data)
  Kind 0: End of Options
  Kind 1: NOP (padding)
```

### Common Options

```
┌──────┬────────┬────────────────────────────────────────────────┐
│ Kind │ Length │ Description                                    │
├──────┼────────┼────────────────────────────────────────────────┤
│  0   │   -    │ End of Options List                            │
│  1   │   -    │ NOP (No Operation) - padding                   │
│  2   │   4    │ MSS (Maximum Segment Size)                     │
│  3   │   3    │ Window Scale                                   │
│  4   │   2    │ SACK Permitted                                 │
│  5   │  var   │ SACK (Selective Acknowledgment)                │
│  8   │  10    │ Timestamps (TSval, TSecr)                      │
└──────┴────────┴────────────────────────────────────────────────┘
```

### MSS Option (Kind 2)

```
Maximum Segment Size - largest payload sender can receive.

┌─────────┬────────┬─────────────────────┐
│ Kind=2  │ Len=4  │   MSS Value (16b)   │
└─────────┴────────┴─────────────────────┘

Typical: 1460 (Ethernet) or 1440 (with timestamps)
Only in SYN and SYN-ACK segments.
```

### Window Scale Option (Kind 3)

```
Multiplier for window field: Window × 2^scale

┌─────────┬────────┬────────────┐
│ Kind=3  │ Len=3  │ Shift (8b) │
└─────────┴────────┴────────────┘

Shift range: 0-14
Max window: 65535 × 2^14 = ~1GB

Only in SYN and SYN-ACK.
```

### SACK Option (Kind 5)

```
Reports non-contiguous received blocks:

┌─────────┬────────┬──────────┬──────────┬─────┐
│ Kind=5  │ Length │ Left Edge│Right Edge│ ... │
└─────────┴────────┴──────────┴──────────┴─────┘

Example: SACK 1001-1500, 2001-3000
  "I have bytes 1001-1500 and 2001-3000,
   missing 1501-2000"
```

### Timestamps Option (Kind 8)

```
┌─────────┬────────┬────────────────┬────────────────┐
│ Kind=8  │ Len=10 │ TSval (32 bit) │ TSecr (32 bit) │
└─────────┴────────┴────────────────┴────────────────┘

TSval:  Sender's current timestamp
TSecr:  Echo of peer's last timestamp

Uses:
1. RTT measurement (TSecr shows when original was sent)
2. PAWS - detect old duplicates by timestamp
```

## Example TCP Segments

### SYN Segment

```
Client initiating connection to web server:

Source Port:      52431
Dest Port:        80
Sequence:         2837465182 (random ISN)
Acknowledgment:   0 (not used)
Data Offset:      8 (32 bytes header = options)
Flags:            SYN
Window:           65535
Checksum:         0x1a2b
Urgent:           0

Options:
  MSS: 1460
  SACK Permitted
  Timestamps: TSval=1234567, TSecr=0
  NOP (padding)
  Window Scale: 7
```

### Data Segment

```
HTTP request being sent:

Source Port:      52431
Dest Port:        80
Sequence:         2837465183
Acknowledgment:   948271635
Data Offset:      8
Flags:            PSH, ACK
Window:           502
Checksum:         0x3c4d
Urgent:           0

Options:
  NOP, NOP
  Timestamps: TSval=1234590, TSecr=9876543

Payload (95 bytes):
  GET / HTTP/1.1\r\n
  Host: example.com\r\n
  \r\n
```

### ACK-only Segment

```
Acknowledging received data (no payload):

Source Port:      80
Dest Port:        52431
Sequence:         948272000
Acknowledgment:   2837465278
Data Offset:      8
Flags:            ACK
Window:           1024
Checksum:         0x5e6f
Urgent:           0

Options:
  NOP, NOP
  Timestamps: TSval=9876600, TSecr=1234590

Payload: (empty)
```

## Segment Size Considerations

### Maximum Segment Size (MSS)

```
MSS = MTU - IP Header - TCP Header
MSS = 1500 - 20 - 20 = 1460 bytes (typical Ethernet)

With timestamps (common):
MSS = 1500 - 20 - 32 = 1448 bytes

The actual payload in a segment ≤ MSS
```

### Why Segment Size Matters

```
Too small:
- More packets = more overhead
- More ACKs needed
- Less efficient

Too large:
- IP fragmentation (bad for performance)
- Higher chance of loss requiring retransmit

Optimal: Just under MTU (Path MTU Discovery helps)
```

## Viewing TCP Headers

### Using tcpdump

```bash
$ tcpdump -i eth0 -nn tcp port 80 -vvX

15:30:45.123456 IP 192.168.1.100.52431 > 93.184.216.34.80:
    Flags [S], cksum 0x1a2b (correct),
    seq 2837465182, win 65535,
    options [mss 1460,sackOK,TS val 1234567 ecr 0,
             nop,wscale 7],
    length 0
```

### Using Wireshark

Wireshark provides a graphical view with all fields decoded:

```
Transmission Control Protocol, Src Port: 52431, Dst Port: 80
    Source Port: 52431
    Destination Port: 80
    Sequence Number: 2837465182
    Acknowledgment Number: 0
    Header Length: 32 bytes (8)
    Flags: 0x002 (SYN)
        .... .... ..0. = Reserved
        .... .... ...0 = Nonce
        .... ...0 .... = Congestion Window Reduced
        .... ..0. .... = ECN-Echo
        .... .0.. .... = Urgent
        .... 0... .... = Acknowledgment
        ...0 .... .... = Push
        ..0. .... .... = Reset
        .0.. .... .... = Syn: Set
        0... .... .... = Fin
    Window: 65535
    Options: (12 bytes)
```

## Summary

The TCP header contains everything needed for reliable, ordered delivery:

| Field | Size | Purpose |
|-------|------|---------|
| Source/Dest Port | 16 bits each | Identify applications |
| Sequence Number | 32 bits | Track byte position |
| Acknowledgment | 32 bits | Confirm receipt |
| Data Offset | 4 bits | Header length |
| Flags | 8 bits | Control (SYN, ACK, FIN, etc.) |
| Window | 16 bits | Flow control |
| Checksum | 16 bits | Error detection |
| Options | Variable | MSS, SACK, timestamps, etc. |

Understanding these fields helps you:
- Debug connection problems
- Interpret packet captures
- Tune TCP performance
- Recognize attacks (SYN floods, RST attacks)

Next, we'll explore flow control—how TCP prevents senders from overwhelming receivers.
