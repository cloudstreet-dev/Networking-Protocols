# SMTP: Email Delivery

**SMTP (Simple Mail Transfer Protocol)** is how email moves between servers. Despite being from 1982, it remains the backbone of email delivery.

## How Email Flows

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Email Delivery Path                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  alice@gmail.com sends to bob@example.com                           │
│                                                                     │
│  ┌────────────┐                                                     │
│  │   Alice    │                                                     │
│  │  (Gmail)   │                                                     │
│  └─────┬──────┘                                                     │
│        │ 1. Compose & Send                                          │
│        ▼                                                            │
│  ┌────────────┐                                                     │
│  │Gmail Server│                                                     │
│  │    MTA     │                                                     │
│  └─────┬──────┘                                                     │
│        │ 2. DNS lookup: example.com MX                              │
│        │ 3. SMTP to mail.example.com                                │
│        ▼                                                            │
│  ┌────────────┐                                                     │
│  │Example.com │                                                     │
│  │Mail Server │                                                     │
│  └─────┬──────┘                                                     │
│        │ 4. Store in Bob's mailbox                                  │
│        ▼                                                            │
│  ┌────────────┐                                                     │
│  │    Bob     │ 5. Retrieve via IMAP/POP3                           │
│  └────────────┘                                                     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

## SMTP Conversation

```
S: 220 mail.example.com ESMTP ready
C: EHLO gmail.com
S: 250-mail.example.com
S: 250-SIZE 35882577
S: 250-STARTTLS
S: 250 OK

C: STARTTLS
S: 220 Ready to start TLS
   (TLS handshake happens)

C: EHLO gmail.com
S: 250 OK

C: MAIL FROM:<alice@gmail.com>
S: 250 OK

C: RCPT TO:<bob@example.com>
S: 250 OK

C: DATA
S: 354 Start mail input
C: From: alice@gmail.com
C: To: bob@example.com
C: Subject: Hello!
C:
C: Hi Bob, how are you?
C: .
S: 250 OK, message queued

C: QUIT
S: 221 Bye
```

## Ports and Security

```
Port 25:   Server-to-server (MTA to MTA)
           Often blocked by ISPs for end users

Port 587:  Client submission (with authentication)
           Modern email clients use this

Port 465:  SMTPS (implicit TLS)
           Deprecated but re-standardized

Security:
  STARTTLS: Upgrade plain connection to TLS
  AUTH:     Login with username/password
  SPF:      Verify sender IP authorized
  DKIM:     Cryptographic message signature
  DMARC:    Policy for SPF/DKIM failures
```

## Email Authentication (SPF, DKIM, DMARC)

```
SPF (DNS TXT record):
  example.com TXT "v=spf1 include:_spf.google.com -all"
  "Only Google's servers can send as @example.com"

DKIM (signature in header):
  DKIM-Signature: v=1; a=rsa-sha256; d=example.com; s=selector;
    h=from:to:subject:date; bh=...; b=...
  Receiver fetches public key from DNS, verifies signature.

DMARC (policy):
  _dmarc.example.com TXT "v=DMARC1; p=reject; rua=mailto:..."
  "If SPF/DKIM fail, reject the message and report."
```

## Common Issues

```
Rejected as spam:
  - Missing SPF/DKIM/DMARC
  - IP on blocklist
  - Poor sending reputation

Connection refused:
  - Port 25 blocked (use 587)
  - Firewall rules
  - Server down

Authentication failed:
  - Wrong credentials
  - App-specific password needed
  - TLS required but not enabled
```
