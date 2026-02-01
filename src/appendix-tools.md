# Appendix: Tools and Debugging

This appendix covers essential tools for network debugging, packet analysis, and protocol troubleshooting.

## Command Line Tools

### curl - HTTP Client

```bash
# Basic GET request
$ curl https://example.com

# Verbose output (see headers, TLS handshake)
$ curl -v https://example.com

# Show only response headers
$ curl -I https://example.com

# POST with JSON
$ curl -X POST https://api.example.com/data \
  -H "Content-Type: application/json" \
  -d '{"key": "value"}'

# Follow redirects
$ curl -L https://example.com

# Save response to file
$ curl -o output.html https://example.com

# Show timing breakdown
$ curl -w "@curl-timing.txt" -o /dev/null -s https://example.com

# Custom timing format
$ curl -w "DNS: %{time_namelookup}s\nConnect: %{time_connect}s\nTLS: %{time_appconnect}s\nTotal: %{time_total}s\n" \
  -o /dev/null -s https://example.com
```

### dig - DNS Queries

```bash
# Query A record
$ dig example.com

# Query specific record type
$ dig example.com AAAA
$ dig example.com MX
$ dig example.com TXT

# Use specific DNS server
$ dig @8.8.8.8 example.com

# Short output
$ dig +short example.com

# Trace resolution path
$ dig +trace example.com

# Reverse lookup
$ dig -x 93.184.216.34

# Show all records
$ dig example.com ANY
```

### nslookup - DNS Lookup (Alternative)

```bash
# Basic lookup
$ nslookup example.com

# Specify record type
$ nslookup -type=MX example.com

# Use specific server
$ nslookup example.com 8.8.8.8
```

### netstat / ss - Network Connections

```bash
# Show all TCP connections
$ netstat -ant     # Linux/Mac
$ ss -ant          # Linux (faster)

# Show listening ports
$ netstat -tlnp    # Linux
$ ss -tlnp         # Linux

# Show UDP sockets
$ ss -u

# Show process using port
$ ss -tlnp | grep :8080
$ lsof -i :8080    # Mac/Linux
```

### tcpdump - Packet Capture

```bash
# Capture all traffic on interface
$ sudo tcpdump -i eth0

# Capture specific port
$ sudo tcpdump -i eth0 port 80

# Capture specific host
$ sudo tcpdump -i eth0 host 192.168.1.100

# Save to file (for Wireshark)
$ sudo tcpdump -i eth0 -w capture.pcap

# Read from file
$ tcpdump -r capture.pcap

# Show packet contents (ASCII)
$ sudo tcpdump -i eth0 -A port 80

# Show packet contents (hex + ASCII)
$ sudo tcpdump -i eth0 -X port 80

# Capture only TCP SYN packets
$ sudo tcpdump -i eth0 'tcp[tcpflags] & tcp-syn != 0'

# Capture DNS queries
$ sudo tcpdump -i eth0 port 53
```

### ping - Connectivity Test

```bash
# Basic ping
$ ping example.com

# Specify count
$ ping -c 4 example.com

# Set interval
$ ping -i 0.5 example.com

# Set packet size
$ ping -s 1000 example.com

# IPv6 ping
$ ping6 example.com
```

### traceroute - Path Discovery

```bash
# Trace route to destination
$ traceroute example.com

# Use ICMP (like ping)
$ traceroute -I example.com    # Linux
$ traceroute example.com       # Mac (ICMP default)

# Use TCP
$ traceroute -T -p 80 example.com

# Use UDP (default on Linux)
$ traceroute -U example.com
```

### mtr - Combined Ping + Traceroute

```bash
# Interactive mode
$ mtr example.com

# Report mode (run 10 times, output)
$ mtr -r -c 10 example.com

# Show IP addresses only
$ mtr -n example.com
```

### openssl - TLS/SSL Testing

```bash
# Connect and show certificate
$ openssl s_client -connect example.com:443

# Show certificate details
$ openssl s_client -connect example.com:443 2>/dev/null | \
  openssl x509 -text -noout

# Check certificate expiration
$ openssl s_client -connect example.com:443 2>/dev/null | \
  openssl x509 -noout -dates

# Test specific TLS version
$ openssl s_client -connect example.com:443 -tls1_2
$ openssl s_client -connect example.com:443 -tls1_3

# Show supported ciphers
$ openssl s_client -connect example.com:443 -cipher 'ALL' 2>&1 | \
  grep "Cipher is"
```

### nc (netcat) - TCP/UDP Tool

```bash
# Connect to port
$ nc example.com 80

# Listen on port
$ nc -l 8080

# Send UDP packet
$ echo "test" | nc -u 192.168.1.1 53

# Port scanning
$ nc -zv example.com 20-25

# Transfer file
$ nc -l 8080 > received.txt       # Receiver
$ nc host 8080 < file.txt          # Sender
```

## Wireshark

Wireshark is the standard GUI tool for packet analysis.

### Capture Filters (BPF Syntax)

```
# Capture specific host
host 192.168.1.100

# Capture specific port
port 80

# Capture range of ports
portrange 8000-9000

# Capture TCP only
tcp

# Combine filters
host 192.168.1.100 and port 443
tcp and not port 22
```

### Display Filters

```
# Filter by IP
ip.addr == 192.168.1.100
ip.src == 192.168.1.100
ip.dst == 10.0.0.1

# Filter by port
tcp.port == 80
tcp.dstport == 443

# Filter by protocol
http
dns
tls
tcp
udp

# HTTP specific
http.request.method == "GET"
http.response.code == 200

# TCP flags
tcp.flags.syn == 1
tcp.flags.fin == 1
tcp.flags.reset == 1

# TLS specific
tls.handshake.type == 1    # Client Hello
tls.handshake.type == 2    # Server Hello

# DNS specific
dns.qry.name == "example.com"

# Combine filters
ip.addr == 192.168.1.100 && tcp.port == 443
http.request || http.response
```

### Useful Wireshark Features

```
Follow TCP Stream:
  Right-click packet → Follow → TCP Stream
  Shows complete conversation in readable format

Flow Graph:
  Statistics → Flow Graph
  Visualizes packet flow between hosts

Protocol Hierarchy:
  Statistics → Protocol Hierarchy
  Shows breakdown of protocols in capture

Expert Info:
  Analyze → Expert Information
  Highlights anomalies, retransmissions, errors

I/O Graph:
  Statistics → I/O Graph
  Visualizes traffic over time
```

## HTTP-Specific Tools

### httpie - Modern HTTP Client

```bash
# GET request
$ http example.com

# POST with JSON (automatic)
$ http POST api.example.com/users name=john age:=25

# Custom headers
$ http example.com Authorization:"Bearer token123"

# Form data
$ http -f POST example.com/login user=john pass=secret
```

### wget - Download Tool

```bash
# Download file
$ wget https://example.com/file.zip

# Continue interrupted download
$ wget -c https://example.com/large-file.zip

# Mirror website
$ wget -m https://example.com

# Download with custom filename
$ wget -O output.zip https://example.com/file.zip
```

### ab (Apache Bench) - Load Testing

```bash
# 1000 requests, 10 concurrent
$ ab -n 1000 -c 10 https://example.com/

# With keep-alive
$ ab -n 1000 -c 10 -k https://example.com/
```

### wrk - Modern Load Testing

```bash
# 30 second test, 12 threads, 400 connections
$ wrk -t12 -c400 -d30s https://example.com/

# With Lua script for custom requests
$ wrk -t12 -c400 -d30s -s script.lua https://example.com/
```

## Debugging Common Issues

### Connection Refused

```
$ curl https://example.com:8080
curl: (7) Failed to connect: Connection refused

Causes:
  - Service not running
  - Wrong port
  - Firewall blocking

Debug:
  $ ss -tlnp | grep 8080         # Is anything listening?
  $ sudo iptables -L -n           # Check firewall
  $ systemctl status service      # Check service
```

### Connection Timeout

```
$ curl --connect-timeout 5 https://example.com
curl: (28) Connection timed out

Causes:
  - Host unreachable
  - Firewall dropping packets (not rejecting)
  - Network routing issue

Debug:
  $ ping example.com              # Basic connectivity
  $ traceroute example.com        # Where does it stop?
  $ tcpdump -i eth0 host example.com  # See outgoing packets
```

### DNS Resolution Failure

```
$ curl https://example.com
curl: (6) Could not resolve host: example.com

Debug:
  $ dig example.com               # Query DNS directly
  $ dig @8.8.8.8 example.com      # Try different DNS
  $ cat /etc/resolv.conf          # Check DNS config
```

### TLS/SSL Errors

```
$ curl https://example.com
curl: (60) SSL certificate problem

Debug:
  $ openssl s_client -connect example.com:443
  # Check for:
  #   - Certificate chain
  #   - Expiration date
  #   - Common name / SAN matching

  $ curl -v https://example.com 2>&1 | grep -i ssl
```

### Slow Connections

```
Debug with timing:
  $ curl -w "DNS: %{time_namelookup}s
  TCP: %{time_connect}s
  TLS: %{time_appconnect}s
  TTFB: %{time_starttransfer}s
  Total: %{time_total}s\n" -o /dev/null -s https://example.com

High DNS time: DNS resolver issue
High TCP time: Network latency
High TLS time: TLS negotiation slow
High TTFB: Server processing slow
```

## Quick Reference

```
┌────────────────────────────────────────────────────────────────┐
│                    Tool Quick Reference                        │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  What you need               Tool to use                       │
│  ─────────────────────────────────────────────────────────     │
│  HTTP debugging              curl -v, httpie                   │
│  DNS lookup                  dig, nslookup                     │
│  Connectivity test           ping, nc                          │
│  Path tracing                traceroute, mtr                   │
│  Port checking               ss, netstat, lsof                 │
│  Packet capture              tcpdump, Wireshark                │
│  TLS/Certificate check       openssl s_client                  │
│  Load testing                ab, wrk                           │
│  File download               curl, wget                        │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```
