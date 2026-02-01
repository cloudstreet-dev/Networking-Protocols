# FTP and Secure Alternatives

**FTP (File Transfer Protocol)** is one of the oldest internet protocols (1971). While still used, security concerns have led to better alternatives.

## How FTP Works

```
FTP uses two connections:

Control Connection (Port 21):
  - Commands and responses
  - Stays open during session
  - Text-based protocol

Data Connection (Port 20 or ephemeral):
  - Actual file transfer
  - Opened per transfer
  - Closed after each file

┌────────────┐                    ┌────────────┐
│   Client   │                    │   Server   │
├────────────┤                    ├────────────┤
│ Control ───┼────── Port 21 ─────┼─── Control │
│            │                    │            │
│   Data  ◄──┼─── Port 20/high ───┼──►  Data   │
└────────────┘                    └────────────┘
```

## Active vs Passive Mode

```
Active Mode:
  1. Client opens control connection to server:21
  2. Client tells server: "Connect to me on port 5000"
  3. Server connects FROM port 20 TO client:5000

  Problem: Client firewalls block incoming connections

Passive Mode (PASV):
  1. Client opens control connection to server:21
  2. Client: "PASV" (I'll connect to you)
  3. Server: "227 Entering Passive (192,168,1,100,195,149)"
     (Connect to 192.168.1.100 port 50069)
  4. Client connects to server's data port

  Better: Client initiates both connections (firewall-friendly)
```

## FTP Session Example

```
$ ftp ftp.example.com
220 Welcome to Example FTP
Name: alice
331 Password required
Password: ********
230 Login successful

ftp> pwd
257 "/" is current directory

ftp> ls
227 Entering Passive Mode (192,168,1,100,195,149)
150 Here comes the directory listing
drwxr-xr-x    2 alice  staff   68 Jan 15 10:00 documents
-rw-r--r--    1 alice  staff 1234 Jan 14 09:00 readme.txt
226 Directory send OK

ftp> get readme.txt
227 Entering Passive Mode (192,168,1,100,195,150)
150 Opening data connection
226 Transfer complete

ftp> quit
221 Goodbye
```

## FTP Security Problems

```
✗ Passwords sent in plaintext
✗ Data transferred unencrypted
✗ No server authentication
✗ Complex firewall requirements

Anyone on the network can see:
  - Username and password
  - All file contents
  - All commands
```

## Secure Alternatives

### SFTP (SSH File Transfer Protocol)

```
Runs over SSH (port 22):
  ✓ Encrypted connection
  ✓ Strong authentication
  ✓ Single port (firewall-friendly)
  ✓ Widely supported

$ sftp user@server.example.com
sftp> put localfile.txt
sftp> get remotefile.txt
sftp> ls
sftp> exit
```

### SCP (Secure Copy)

```
Simple file copy over SSH:

# Copy local to remote
$ scp file.txt user@server:/path/

# Copy remote to local
$ scp user@server:/path/file.txt ./

# Copy directory recursively
$ scp -r localdir user@server:/path/
```

### FTPS (FTP over TLS)

```
FTP with TLS encryption:
  - Implicit FTPS: TLS from start (port 990)
  - Explicit FTPS: STARTTLS upgrade (port 21)

Still has FTP complexity (dual connections).
SFTP generally preferred.
```

## Recommendation

```
For new deployments:
  1. SFTP      Best overall (secure, firewall-friendly)
  2. SCP       Simple file copies
  3. rsync     Efficient synchronization
  4. HTTPS     API-based file transfer

Avoid:
  - Plain FTP (insecure)
  - TFTP (no authentication at all)
```
