# 📚 Do Not Disturb — Technical Notes & Cheat Sheet

> Supporting notes for the **Do Not Disturb** TryHackMe Boot2Root room from **Hacker Holidays 2026 — Day 7**.

This document contains concise cybersecurity notes, exploitation concepts, commands, and privilege escalation references used while solving the room. It is designed as a **quick reference guide** rather than a walkthrough.

---

## Room Summary

| Category | Details |
|----------|---------|
| Platform | TryHackMe |
| Room | Do Not Disturb |
| Difficulty | Medium |
| Environment | Linux / Node.js / Express / MongoDB |
| Focus Areas | NoSQL Injection, SSTI, Node.js Inspector, Linux Privilege Escalation |

---

# Attack Path Cheat Sheet

```text
Web Enumeration
      │
      ▼
Hidden /staff Endpoint
      │
      ▼
NoSQL Injection Authentication Bypass
      │
      ▼
Authenticated Staff Console
      │
      ▼
EJS SSTI
      │
      ▼
Remote Code Execution
      │
      ▼
Reverse Shell
      │
      ▼
Internal Service Enumeration
      │
      ▼
Node.js Inspector (9229)
      │
      ▼
Service Account Enumeration
      │
      ▼
disk Group Abuse
      │
      ▼
Root Flag
```

---

# Skills Practiced

- Web directory enumeration
- Burp Suite request interception
- MongoDB NoSQL Injection
- Session authentication bypass
- Server-Side Template Injection (EJS)
- Remote Code Execution (Node.js)
- Reverse shell establishment
- Linux service enumeration
- Node.js Inspector abuse
- Privilege escalation using Linux permissions

---

# Tools Used

| Tool | Purpose |
|------|---------|
| Gobuster | Directory discovery |
| Burp Suite Community | HTTP interception |
| FoxyProxy | Browser proxy switching |
| Netcat | Reverse shell listener |
| Node Inspector | Runtime debugging |
| Linux CLI Utilities | Enumeration |

---

# Web Enumeration Notes

## Gobuster

Discover hidden directories.

```bash
gobuster dir \
-u http://TARGET_IP \
-w /usr/share/wordlists/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt
```

### Interesting Responses

| Status | Meaning |
|--------|---------|
| 200 | Accessible page |
| 301/302 | Redirect |
| 401 | Authentication Required |
| 403 | Resource exists but forbidden |
| 404 | Not Found |

### Enumeration Tips

- Use larger wordlists after initial discovery.
- Enumerate recursively if needed.
- Investigate forbidden endpoints.

---

# Burp Suite Notes

## Authentication Interception Workflow

1. Enable FoxyProxy.
2. Turn **Intercept On**.
3. Submit login request.
4. Modify request.
5. Forward request.
6. Capture session cookie.

### Useful Burp Tabs

| Tab | Purpose |
|-----|---------|
| Intercept | Modify live requests |
| HTTP History | Review requests |
| Repeater | Manual testing |
| Decoder | Encode / Decode payloads |

---

# NoSQL Injection Notes

## MongoDB Query Operators

| Operator | Description |
|----------|-------------|
| `$eq` | Equals |
| `$ne` | Not Equal |
| `$gt` | Greater Than |
| `$lt` | Less Than |
| `$regex` | Pattern Matching |
| `$exists` | Field Exists |

### Authentication Bypass Concept

When backend code directly accepts JSON query objects, MongoDB operators may become executable instead of treated as strings.

### Common Payload Patterns

```text
username[$ne]=...
password[$ne]=...
```

Other operators frequently tested include:

```text
$regex
$gt
$exists
```

> Always validate input server-side and enforce string types before querying MongoDB.

---

# Session Cookie Notes

## Express Session Cookie

Common cookie name:

```text
connect.sid
```

### Why It's Important

- Represents authenticated session.
- Used by Express Session middleware.
- Grants access without resending credentials.

### Security Best Practices

- HttpOnly cookies.
- Secure cookies.
- SameSite protection.
- Session expiration.

---

# Server-Side Template Injection (SSTI)

## What is SSTI?

A vulnerability where user-controlled template input is rendered on the server.

### EJS Syntax

```ejs
<%= expression %>
```

Outputs evaluated value.

```ejs
<% code %>
```

Executes JavaScript without printing output.

---

## SSTI Detection Checklist

- Arithmetic evaluation.
- String concatenation.
- Object access.
- JavaScript execution.
- Node.js module access.

### Indicators

| Payload Type | Expected Behaviour |
|--------------|-------------------|
| Arithmetic | Output calculated value |
| String | Output rendered string |
| Object | Output object value |
| Invalid syntax | Template error |

---

# Node.js Runtime Notes

## Child Process Module

Provides OS command execution.

Common functions:

```javascript
exec()
execSync()
spawn()
spawnSync()
execFile()
execFileSync()
```

### Differences

| Function | Behaviour |
|----------|-----------|
| execSync | Executes shell synchronously |
| spawn | Starts process asynchronously |
| execFileSync | Executes binary without shell |

---

# Reverse Shell Notes

## Reverse Shell Workflow

1. Start listener.
2. Trigger payload.
3. Receive shell.

Listener example:

```bash
nc -lvnp PORT
```

### Reverse Shell Checklist

- Listener active.
- Correct IP.
- Correct port.
- Firewall reachable.

---

# Linux Enumeration Notes

## Useful Enumeration Commands

### Network

```bash
ss -tlnu
```

Shows listening TCP and UDP sockets.

### Processes

```bash
ps aux
```

### Identity

```bash
id
whoami
groups
```

### Kernel

```bash
uname -a
```

### OS

```bash
cat /etc/os-release
```

---

## Localhost Enumeration

Always inspect services bound to:

```text
127.0.0.1
localhost
```

Common internal services:

| Port | Service |
|------|---------|
| 3306 | MySQL |
| 5432 | PostgreSQL |
| 6379 | Redis |
| 9229 | Node Inspector |
| 27017 | MongoDB |

---

# Node.js Inspector Notes

## Default Port

```text
9229
```

### Purpose

Remote debugging interface for Node.js applications.

### Risks

- Execute JavaScript.
- Inspect memory.
- Read variables.
- Execute privileged runtime code.

### Detection

```bash
ss -tlnu
```

---

# JavaScript REPL Notes

Useful runtime information.

### Process Information

```javascript
process.getuid()
process.getgid()
process.cwd()
process.env
```

### Runtime Inspection

```javascript
process.version
process.pid
process.platform
```

### Built-in Modules

```javascript
process.getBuiltinModule(...)
```

---

# Linux Groups Cheat Sheet

| Group | Privilege |
|-------|-----------|
| sudo | Administrative access |
| docker | Docker daemon access |
| lxd | Container escape possibilities |
| adm | Logs |
| disk | Raw disk access |

---

## Why the disk Group is Dangerous

Members may access raw block devices.

Examples:

```text
/dev/sda
/dev/sdb
/dev/nvme0n1
/dev/nvme0n1p1
```

Potential abuse:

- Read protected files.
- Extract hashes.
- Inspect filesystem offline.

---

# debugfs Notes

## Purpose

Filesystem debugger for ext filesystems.

### Capabilities

- Browse filesystem.
- Read files.
- Recover deleted files.
- Inspect inodes.

### Example Operations

```bash
ls
stat
cat
dump
```

### Security Impact

If user has raw disk access, filesystem permissions can be bypassed.

---

# Privilege Escalation Indicators

## Things Worth Checking

- SUID binaries.
- Writable cron jobs.
- Docker group.
- LXD group.
- Capabilities.
- Kernel exploits.
- Debug services.
- Disk group membership.

---

# Vulnerability Summary

| Vulnerability | Risk |
|--------------|------|
| NoSQL Injection | Authentication Bypass |
| SSTI | Remote Code Execution |
| Exposed Node Inspector | Runtime Compromise |
| disk Group Membership | Privilege Escalation |

---

# Defensive Mitigations

## Web Application

- Validate input types.
- Disable MongoDB operator injection.
- Sanitize template input.
- Restrict template functionality.

## Infrastructure

- Disable Node Inspector in production.
- Bind debugger only during development.
- Remove unnecessary Linux group memberships.
- Audit service account privileges.

---

# MITRE ATT&CK Mapping

| Tactic | Technique |
|--------|-----------|
| Reconnaissance | Active Scanning |
| Initial Access | Exploit Public-Facing Application |
| Execution | Command and Scripting Interpreter |
| Discovery | System Information Discovery |
| Privilege Escalation | Abuse Elevation Control Mechanism |
| Collection | Data from Local System |

---

# Commands Used During the Room

## Enumeration

```bash
gobuster dir ...
```

```bash
ss -tlnu
```

## Reverse Shell Listener

```bash
nc -lvnp 4444
```

## Runtime Enumeration

```javascript
process.getuid()
process.getgid()
```

---

# Key Takeaways

- **403 endpoints are valuable reconnaissance findings.**
- **NoSQL Injection** occurs when backend query operators are accepted as user input.
- **SSTI** allows execution inside the server-side rendering engine.
- **Node.js Inspector** should never be exposed in production environments.
- **Least Privilege** is critical—membership in privileged Linux groups like `disk` can lead to full system compromise.

---

## Related Topics for Further Practice

- OWASP NoSQL Injection
- OWASP Server-Side Template Injection
- Express Session Security
- Node.js Security Best Practices
- Linux Privilege Escalation
- TryHackMe Linux PrivEsc
- TryHackMe Node.js Security Rooms

---

**Repository:** *Do Not Disturb TryHackMe Walkthrough*

*Quick technical reference maintained alongside the complete walkthrough documentation.*
