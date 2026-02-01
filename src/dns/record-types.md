# DNS Record Types

DNS stores more than just IP addresses. Different **record types** serve different purposes—from pointing domain names to servers, to routing email, to verifying domain ownership.

## Common Record Types

```
┌───────────────────────────────────────────────────────────────────────┐
│ Type  │ Name                   │ Purpose                              │
├───────┼────────────────────────┼──────────────────────────────────────┤
│  A    │ Address                │ Maps name to IPv4 address            │
│ AAAA  │ IPv6 Address           │ Maps name to IPv6 address            │
│ CNAME │ Canonical Name         │ Alias to another name                │
│  MX   │ Mail Exchange          │ Email server for domain              │
│  TXT  │ Text                   │ Arbitrary text (verification, SPF)   │
│  NS   │ Name Server            │ Authoritative servers for zone       │
│ SOA   │ Start of Authority     │ Zone metadata and parameters         │
│  PTR  │ Pointer                │ Reverse DNS (IP to name)             │
│  SRV  │ Service                │ Service location (port, priority)    │
│ CAA   │ Cert. Authority Auth.  │ Which CAs can issue certificates     │
└───────┴────────────────────────┴──────────────────────────────────────┘
```

## A Record (Address)

Maps a domain name to an IPv4 address.

```
Record:
  Name:  example.com
  Type:  A
  TTL:   3600
  Value: 93.184.216.34

Lookup:
$ dig example.com A
example.com.    3600    IN    A    93.184.216.34
```

### Multiple A Records

```
Load balancing via DNS:

example.com.    300    IN    A    192.0.2.1
example.com.    300    IN    A    192.0.2.2
example.com.    300    IN    A    192.0.2.3

Client picks one (often randomly or round-robin).
Simple load distribution, no dedicated load balancer.

Drawbacks:
  - No health checking
  - Uneven distribution possible
  - Cached entries persist after server failure
```

## AAAA Record (IPv6 Address)

Maps a domain name to an IPv6 address.

```
Record:
  Name:  example.com
  Type:  AAAA
  TTL:   3600
  Value: 2606:2800:220:1:248:1893:25c8:1946

Lookup:
$ dig example.com AAAA
example.com.    3600    IN    AAAA    2606:2800:220:1:248:1893:25c8:1946

"AAAA" = four times "A" = four times the address size (32 → 128 bits)
```

### Dual Stack

```
Many domains have both A and AAAA:

example.com.    A       93.184.216.34
example.com.    AAAA    2606:2800:220:1:248:1893:25c8:1946

Client chooses based on connectivity:
  - Happy Eyeballs algorithm prefers IPv6
  - Falls back to IPv4 if IPv6 fails
```

## CNAME Record (Canonical Name)

Creates an alias pointing to another domain name.

```
Record:
  Name:  www.example.com
  Type:  CNAME
  TTL:   3600
  Value: example.com

Lookup:
$ dig www.example.com
www.example.com.    3600    IN    CNAME    example.com.
example.com.        3600    IN    A        93.184.216.34

Resolver follows the chain:
  www.example.com → example.com → 93.184.216.34
```

### CNAME Use Cases

```
1. WWW alias:
   www.example.com → example.com

2. CDN integration:
   cdn.example.com → d1234.cloudfront.net

3. Service endpoints:
   api.example.com → api-prod.company.internal

4. Environment switching:
   app.example.com → staging.example.com  (during testing)
   app.example.com → production.example.com  (in production)
```

### CNAME Restrictions

```
Cannot coexist with other records at same name:

INVALID:
  example.com    CNAME    other.com
  example.com    A        1.2.3.4      ← Conflict!

INVALID:
  example.com    CNAME    other.com
  example.com    MX       mail.example.com  ← Conflict!

Therefore: Cannot use CNAME at zone apex (example.com)
           Must use A/AAAA records there

Workarounds:
  - ALIAS records (provider-specific, not standard DNS)
  - ANAME records (draft standard)
```

## MX Record (Mail Exchange)

Specifies email servers for a domain.

```
Record:
  Name:     example.com
  Type:     MX
  TTL:      3600
  Priority: 10
  Value:    mail.example.com

Lookup:
$ dig example.com MX
example.com.    3600    IN    MX    10 mail.example.com.
example.com.    3600    IN    MX    20 backup.example.com.
```

### MX Priority

```
Lower number = higher priority

example.com.    MX    10 primary.mail.example.com.
example.com.    MX    20 secondary.mail.example.com.
example.com.    MX    30 backup.mail.example.com.

Email delivery attempts:
  1. Try primary (priority 10)
  2. If unavailable, try secondary (priority 20)
  3. Last resort: backup (priority 30)
```

### Email Flow with MX

```
Sending email to user@example.com:

1. Sender's MTA queries: example.com MX
2. Gets: mail.example.com (priority 10)
3. Queries: mail.example.com A
4. Gets: 93.184.216.100
5. Connects to 93.184.216.100:25 (SMTP)
6. Delivers email
```

## TXT Record

Stores arbitrary text. Used for verification and email security.

```
Record:
  Name:  example.com
  Type:  TXT
  TTL:   3600
  Value: "v=spf1 include:_spf.google.com ~all"

Lookup:
$ dig example.com TXT
example.com.    3600    IN    TXT    "v=spf1 include:_spf.google.com ~all"
```

### Common TXT Uses

```
1. SPF (Sender Policy Framework):
   "v=spf1 include:_spf.google.com ~all"
   Specifies authorized email senders

2. DKIM (DomainKeys Identified Mail):
   selector._domainkey.example.com TXT "v=DKIM1; k=rsa; p=..."
   Public key for email signing

3. DMARC (Domain-based Message Authentication):
   _dmarc.example.com TXT "v=DMARC1; p=reject; rua=mailto:..."
   Policy for handling authentication failures

4. Domain verification:
   example.com TXT "google-site-verification=abc123..."
   Proves domain ownership to services

5. Custom data:
   example.com TXT "contact=admin@example.com"
```

## NS Record (Name Server)

Specifies authoritative nameservers for a zone.

```
Record:
  Name:  example.com
  Type:  NS
  TTL:   86400
  Value: ns1.example.com

Lookup:
$ dig example.com NS
example.com.    86400    IN    NS    ns1.example.com.
example.com.    86400    IN    NS    ns2.example.com.
```

### Delegation

```
NS records delegate subdomains:

example.com zone has:
  subdomain.example.com.    NS    ns1.subdomain-hosting.com.

Now subdomain.example.com has its own nameservers.
The parent zone "delegates" authority.
```

## SOA Record (Start of Authority)

Contains zone metadata and parameters.

```
$ dig example.com SOA
example.com.    3600    IN    SOA    ns1.example.com. admin.example.com. (
                            2024011501 ; Serial
                            7200       ; Refresh
                            3600       ; Retry
                            1209600    ; Expire
                            3600 )     ; Minimum TTL

Fields:
  Primary NS:     ns1.example.com
  Admin email:    admin@example.com (@ replaced with .)
  Serial:         Version number (often YYYYMMDDNN)
  Refresh:        How often secondaries check for updates
  Retry:          Retry interval after failed refresh
  Expire:         When secondary data becomes invalid
  Minimum:        Negative caching TTL
```

## PTR Record (Pointer)

Maps IP addresses back to names (reverse DNS).

```
IP to name lookup:

IP: 93.184.216.34
Reverse zone: 34.216.184.93.in-addr.arpa

Record:
  34.216.184.93.in-addr.arpa.    PTR    example.com.

Lookup:
$ dig -x 93.184.216.34
34.216.184.93.in-addr.arpa. 3600 IN PTR example.com.
```

### Reverse DNS Uses

```
1. Email server verification:
   Receiving servers check if sender IP has valid PTR
   Missing/mismatched PTR → likely spam

2. Logging and auditing:
   Convert IPs to names for readable logs

3. Security analysis:
   Quick identification of attacking IPs
```

## SRV Record (Service)

Specifies location of services with port and priority.

```
Record format:
  _service._proto.name    SRV    priority weight port target

Example:
  _sip._tcp.example.com.    SRV    10 60 5060 sipserver.example.com.
  _xmpp._tcp.example.com.   SRV    10 50 5222 xmpp.example.com.

Fields:
  Priority: Lower = preferred (like MX)
  Weight:   For load balancing among same priority
  Port:     Service port number
  Target:   Server hostname
```

### SRV Use Cases

```
1. VoIP/SIP:
   _sip._tcp.example.com → voip.example.com:5060

2. XMPP/Jabber:
   _xmpp-client._tcp.example.com → chat.example.com:5222

3. LDAP:
   _ldap._tcp.example.com → ldap.example.com:389

4. Kubernetes services:
   _http._tcp.myservice.namespace → pod-ip:port
```

## CAA Record (Certificate Authority Authorization)

Controls which Certificate Authorities can issue SSL certificates.

```
Record:
  example.com.    CAA    0 issue "letsencrypt.org"
  example.com.    CAA    0 issuewild ";"
  example.com.    CAA    0 iodef "mailto:security@example.com"

Meanings:
  issue:     Which CA can issue regular certs
  issuewild: Which CA can issue wildcard certs (";" = none)
  iodef:     Where to report violations

$ dig example.com CAA
example.com.    3600    IN    CAA    0 issue "letsencrypt.org"
```

## Querying Different Record Types

```bash
# A record (IPv4)
$ dig example.com A

# AAAA record (IPv6)
$ dig example.com AAAA

# MX record (mail servers)
$ dig example.com MX

# TXT record (text)
$ dig example.com TXT

# All records
$ dig example.com ANY  # Note: Many servers don't support ANY

# Specific nameserver
$ dig @8.8.8.8 example.com A

# Short output
$ dig +short example.com A
93.184.216.34

# Trace resolution path
$ dig +trace example.com
```

## Record TTL Considerations

```
TTL (Time To Live) controls caching:

Long TTL (86400 = 24 hours):
  + Fewer queries, lower load
  + Faster lookups (cached)
  - Slow to update, changes take time

Short TTL (60 = 1 minute):
  + Quick updates
  + Fast failover
  - More queries
  - Higher load on nameservers

Recommendations:
  Stable records: 3600-86400 (1-24 hours)
  Dynamic/failover: 60-300 (1-5 minutes)
  During migration: Reduce before, restore after
```

## Summary

| Record | Purpose | Example Value |
|--------|---------|---------------|
| A | IPv4 address | 93.184.216.34 |
| AAAA | IPv6 address | 2606:2800:220:1::1 |
| CNAME | Alias | www → example.com |
| MX | Mail server | 10 mail.example.com |
| TXT | Text/verification | "v=spf1 ..." |
| NS | Nameservers | ns1.example.com |
| SOA | Zone metadata | Serial, timers |
| PTR | Reverse lookup | IP → name |
| SRV | Service location | priority weight port target |
| CAA | CA authorization | 0 issue "letsencrypt.org" |

Understanding record types helps you:
- Configure domains correctly
- Debug email delivery issues
- Set up SSL certificates
- Implement service discovery

Next, we'll explore DNS caching and how TTLs affect performance.
