# SSH: Secure Shell

**SSH (Secure Shell)** provides encrypted remote access, replacing insecure protocols like Telnet and rlogin. Beyond shell access, SSH enables secure file transfer, port forwarding, and tunneling.

## SSH Capabilities

```
┌─────────────────────────────────────────────────────────────────────┐
│                      SSH Use Cases                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Remote Shell:     Interactive command line on remote server        │
│  File Transfer:    SFTP, SCP over encrypted channel                 │
│  Port Forwarding:  Tunnel any TCP connection through SSH            │
│  X11 Forwarding:   Run graphical apps remotely                      │
│  Agent Forwarding: Use local keys on remote servers                 │
│  Git Transport:    Secure repository access                         │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

## Authentication Methods

### Password Authentication

```bash
$ ssh user@server.example.com
user@server.example.com's password: ********

Simple but:
  - Vulnerable to brute force
  - Requires typing password
  - Can't be automated safely
```

### Public Key Authentication

```bash
# Generate key pair
$ ssh-keygen -t ed25519 -C "alice@laptop"
# Creates: ~/.ssh/id_ed25519 (private)
#          ~/.ssh/id_ed25519.pub (public)

# Copy public key to server
$ ssh-copy-id user@server.example.com
# Or manually add to ~/.ssh/authorized_keys

# Login (no password!)
$ ssh user@server.example.com
```

### Key Types

```
Ed25519:     Modern, fast, secure (recommended)
RSA:         Widely compatible (4096-bit minimum)
ECDSA:       Elliptic curve (P-256, P-384)

Avoid:
  DSA:       Deprecated, weak
  RSA <2048: Too short
```

## SSH Configuration

### Client Config (~/.ssh/config)

```
# Default settings
Host *
    AddKeysToAgent yes
    IdentityFile ~/.ssh/id_ed25519

# Named host
Host myserver
    HostName server.example.com
    User alice
    Port 22
    IdentityFile ~/.ssh/work_key

# Now just:
$ ssh myserver
```

### Server Config (/etc/ssh/sshd_config)

```bash
# Secure settings
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
AllowUsers alice bob
Protocol 2
```

## Port Forwarding

### Local Forwarding (-L)

```
Access remote service through local port:

$ ssh -L 8080:localhost:80 user@server

  Local:8080  ──────SSH Tunnel──────>  Server ────> localhost:80
     │                                      (server's port 80)
     └── Your browser connects here

Use case: Access web app behind firewall
```

### Remote Forwarding (-R)

```
Expose local service to remote:

$ ssh -R 9000:localhost:3000 user@server

  Server:9000  <─────SSH Tunnel───────  Local:3000
     │                                      │
     └── Internet can access          Your dev server

Use case: Share local development server
```

### Dynamic Forwarding (-D)

```
SOCKS proxy through SSH:

$ ssh -D 1080 user@server

Configure browser to use SOCKS proxy localhost:1080
All browser traffic goes through server.

Use case: Bypass network restrictions, privacy
```

## SSH Tunnels for Security

```
┌─────────────────────────────────────────────────────────────────────┐
│                    SSH Tunnel Example                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Scenario: Connect to database behind firewall                      │
│                                                                     │
│  ┌────────┐        ┌────────────┐        ┌──────────┐              │
│  │ Laptop │──SSH──>│ Jump Host  │        │ Database │              │
│  │        │        │ (bastion)  │──────>│ :5432    │              │
│  └────────┘        └────────────┘        └──────────┘              │
│                                                                     │
│  $ ssh -L 5432:db.internal:5432 user@bastion                        │
│  $ psql -h localhost -p 5432 mydb                                   │
│                                                                     │
│  Database connection encrypted through SSH tunnel.                  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

## Best Practices

```
Key Management:
  ✓ Use Ed25519 keys
  ✓ Protect private key with passphrase
  ✓ Use ssh-agent to avoid retyping passphrase
  ✓ Rotate keys periodically

Server Security:
  ✓ Disable password authentication
  ✓ Disable root login
  ✓ Use fail2ban for brute force protection
  ✓ Keep SSH updated
  ✓ Use non-standard port (security through obscurity, minor)

Access Control:
  ✓ Limit allowed users
  ✓ Use bastion/jump hosts
  ✓ Audit authorized_keys regularly
```

## Troubleshooting

```bash
# Verbose output
$ ssh -v user@server      # Basic
$ ssh -vvv user@server    # Maximum verbosity

# Check key permissions
$ ls -la ~/.ssh/
# id_ed25519 should be 600
# authorized_keys should be 600

# Test authentication
$ ssh -T git@github.com

# Check server logs (on server)
$ sudo tail -f /var/log/auth.log
```
