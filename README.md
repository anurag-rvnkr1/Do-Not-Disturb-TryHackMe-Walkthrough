<div align="center">

# 🏝️ Do Not Disturb — TryHackMe Walkthrough

<img src="docs/assets/room-banner.png" alt="Do Not Disturb Banner" width="100%">

### Web Exploitation • NoSQL Injection • EJS SSTI • Node.js Inspector • Linux Privilege Escalation

[![TryHackMe](https://img.shields.io/badge/TryHackMe-Room-red?style=for-the-badge&logo=tryhackme)](https://tryhackme.com/)
[![Difficulty](https://img.shields.io/badge/Difficulty-Easy--Medium-orange?style=for-the-badge)]()
[![Category](https://img.shields.io/badge/Category-Web%20Security-blue?style=for-the-badge)]()
[![Platform](https://img.shields.io/badge/Platform-Linux-black?style=for-the-badge&logo=linux)]()
[![NodeJS](https://img.shields.io/badge/Node.js-Express-339933?style=for-the-badge&logo=node.js)]()
[![License](https://img.shields.io/badge/License-MIT-success?style=for-the-badge)](LICENSE)

*A complete cybersecurity portfolio walkthrough documenting the exploitation of a vulnerable hotel booking application through NoSQL Injection, Server-Side Template Injection (SSTI), Remote Code Execution, Node.js Inspector abuse, and Linux privilege escalation.*

</div>

---

## 📖 About This Repository

This repository contains a **professional penetration testing walkthrough** for the **Do Not Disturb** room from **TryHackMe's Hacker Holidays series**.

Unlike a simple write-up, this project is written as a **technical case study** that demonstrates the complete attacker workflow—from reconnaissance to privilege escalation—while also explaining **why each vulnerability exists**, **how it is exploited**, and **how defenders can mitigate it**.

The documentation has been rewritten from scratch for portfolio purposes and follows responsible disclosure practices.

> **All TryHackMe flags have been intentionally redacted** to avoid plagiarism and preserve the learning experience.

---

# 🎯 Room Overview

<table>
<tr>
<td width="220"><strong>Room Name</strong></td>
<td>Do Not Disturb</td>
</tr>

<tr>
<td><strong>Series</strong></td>
<td>Hacker Holidays</td>
</tr>

<tr>
<td><strong>Platform</strong></td>
<td>TryHackMe</td>
</tr>

<tr>
<td><strong>Difficulty</strong></td>
<td>Easy / Medium</td>
</tr>

<tr>
<td><strong>Operating System</strong></td>
<td>Linux</td>
</tr>

<tr>
<td><strong>Application Stack</strong></td>
<td>Express.js • MongoDB • EJS Templates • Node.js</td>
</tr>

<tr>
<td><strong>Focus Areas</strong></td>
<td>
Reconnaissance • Authentication Bypass • SSTI • RCE • Local Enumeration • Privilege Escalation
</td>
</tr>
</table>

---

# 🌴 Scenario

The Byte Lotus Hotel has recently launched a new digital booking portal for guests and staff.

During a security assessment, multiple vulnerabilities were discovered inside the application that can be chained together into a complete compromise.

As an attacker, the objective is to:

- Enumerate hidden application endpoints.
- Bypass authentication.
- Gain access to the internal staff portal.
- Abuse an EJS template rendering feature.
- Achieve remote code execution.
- Enumerate internal services.
- Escalate privileges through a misconfigured debugging service.
- Access privileged resources on the underlying Linux host.

This room demonstrates how seemingly unrelated weaknesses can become a complete attack chain when combined together.

---

# 🧠 Skills Demonstrated

<table>
<tr>
<td width="50%">✅ Web Application Reconnaissance</td>
<td>Gobuster directory enumeration</td>
</tr>

<tr>
<td>✅ HTTP Request Manipulation</td>
<td>Burp Suite Proxy interception</td>
</tr>

<tr>
<td>✅ NoSQL Injection</td>
<td>MongoDB authentication bypass</td>
</tr>

<tr>
<td>✅ Session Hijacking Concepts</td>
<td>Authenticated cookie analysis</td>
</tr>

<tr>
<td>✅ Server-Side Template Injection</td>
<td>EJS template evaluation</td>
</tr>

<tr>
<td>✅ Remote Code Execution</td>
<td>Node.js child_process abuse</td>
</tr>

<tr>
<td>✅ Linux Enumeration</td>
<td>Listening services and local sockets</td>
</tr>

<tr>
<td>✅ Debug Service Enumeration</td>
<td>Node.js Inspector</td>
</tr>

<tr>
<td>✅ Privilege Escalation</td>
<td>debugfs & disk group abuse</td>
</tr>

<tr>
<td>✅ Security Mitigation Analysis</td>
<td>Blue Team recommendations</td>
</tr>

</table>

---

# 🧰 Tools Used

| Tool | Purpose |
|------|---------|
| **Gobuster** | Directory brute-force enumeration |
| **Burp Suite Community** | Intercepting and modifying HTTP requests |
| **FoxyProxy** | Browser proxy management |
| **Firefox** | Web application testing |
| **Netcat** | Reverse shell listener |
| **Node Inspector** | Interacting with exposed debugger |
| **Linux CLI** | Enumeration and privilege escalation |

---

# 📂 Repository Structure

```text
Do-Not-Disturb-TryHackMe-Walkthrough
│
├── README.md
├── LICENSE
├── SECURITY.md
├── CONTRIBUTING.md
├── CODE_OF_CONDUCT.md
│
├── Documentation/
│   ├── Do_Not_Disturb_Documentation.md
│   ├── Do_Not_Disturb_Documentation.docx
│   └── methodology.md
│
├── docs/
│   ├── index.md
│   ├── assets/
│   └── styles/
│
├── Screenshots/
│
└── resources/
    ├── notes.md
    ├── attack-path.md
    ├── iocs.md
    └── references.md
```

---

# 📚 Table of Contents

- [Executive Summary](#-executive-summary)
- [Attack Surface Overview](#-attack-surface-overview)
- [Attack Chain](#-attack-chain)
- [Walkthrough](#-walkthrough)
- [Vulnerability Analysis](#-vulnerability-analysis)
- [Privilege Escalation](#-privilege-escalation)
- [Security Recommendations](#-security-recommendations)
- [MITRE ATT&CK Mapping](#-mitre-attck-mapping)
- [Learning Outcomes](#-learning-outcomes)
- [References](#-references)

---

# 📝 Executive Summary

The **Do Not Disturb** room focuses on identifying and chaining together multiple vulnerabilities inside a Node.js web application.

Rather than relying on a single exploit, the compromise is achieved through a realistic sequence of attacker actions:

1. Discover hidden endpoints.
2. Abuse insecure authentication logic.
3. Gain an authenticated session.
4. Identify unsafe server-side template rendering.
5. Execute JavaScript on the server.
6. Obtain command execution.
7. Enumerate local services.
8. Abuse a misconfigured debugging interface.
9. Escalate privileges by reading the underlying filesystem directly.

This mirrors real-world attack paths where **multiple medium-severity issues combine into a critical compromise**.

---

# 🌐 Attack Surface Overview

## Initial Target

<img src="docs/assets/00-room-homepage.png" width="100%" alt="Room Homepage">

The target is a hotel booking application named **Byte Lotus**.

The application exposes:

- Guest booking portal.
- Staff authentication portal.
- Dynamic booking confirmation templates.
- Express.js backend.
- MongoDB authentication.
- Node.js runtime.

### Observations

- HTTP service running on port 80.
- Dynamic content rendered through EJS.
- Session-based authentication.
- Hidden staff functionality.

---

# 🔗 Complete Attack Chain

```text
                INTERNET
                    │
                    ▼
        Gobuster Enumeration
                    │
        Hidden /staff Endpoint
                    │
                    ▼
      Burp Suite Request Interception
                    │
                    ▼
    NoSQL Injection Authentication Bypass
                    │
        Authenticated Session Cookie
                    │
                    ▼
          Staff Console Access
                    │
                    ▼
        Server-Side Template Injection
                    │
                    ▼
       Remote Code Execution (Node.js)
                    │
                    ▼
         Reverse Shell (poolside user)
                    │
                    ▼
         Local Service Enumeration
                    │
                    ▼
        Node Inspector on 127.0.0.1
                    │
                    ▼
        pipelinesvc Debug Context
                    │
                    ▼
    debugfs Reads Protected Filesystem
                    │
                    ▼
      Privileged File Access (Redacted)
```

---

# 🧩 Vulnerabilities Chained

<table>
<tr>
<th>Stage</th>
<th>Vulnerability</th>
<th>Impact</th>
</tr>

<tr>
<td>Recon</td>
<td>Hidden Staff Endpoint</td>
<td>Discovery of privileged functionality</td>
</tr>

<tr>
<td>Initial Access</td>
<td>NoSQL Injection</td>
<td>Authentication bypass</td>
</tr>

<tr>
<td>Execution</td>
<td>EJS SSTI</td>
<td>JavaScript execution on server</td>
</tr>

<tr>
<td>Execution</td>
<td>Node child_process Abuse</td>
<td>Operating system command execution</td>
</tr>

<tr>
<td>Persistence</td>
<td>Session Cookie</td>
<td>Authenticated portal access</td>
</tr>

<tr>
<td>Discovery</td>
<td>Exposed Node Inspector</td>
<td>Access to internal runtime</td>
</tr>

<tr>
<td>Privilege Escalation</td>
<td>disk Group + debugfs</td>
<td>Read protected filesystem</td>
</tr>

</table>

---

# 🚩 Walkthrough

## Phase 1 — Reconnaissance

### Goal

Identify hidden application functionality.

---

### Directory Enumeration

<img src="docs/assets/01-gobuster.png" width="100%" alt="Gobuster Enumeration">

The assessment begins with directory brute-forcing against the web server.

### Why This Matters

Hidden directories often expose:

- Administrative portals.
- APIs.
- Development environments.
- Backup pages.
- Authentication endpoints.

### Findings

| Endpoint | Observation |
|----------|-------------|
| `/staff` | Protected staff interface. |
| `/logout` | Existing authenticated session functionality. |

The `/staff` endpoint becomes the primary attack target because it is protected with authorization.

---

## Phase 2 — Traffic Interception

<img src="docs/assets/02-foxyproxy.png" width="100%" alt="FoxyProxy">

Traffic is routed through **Burp Suite** using FoxyProxy.

### Burp Suite Workflow

1. Configure browser proxy.
2. Enable request interception.
3. Capture authentication request.
4. Modify parameters.
5. Forward manipulated request.

This allows testing the application's authentication logic without modifying the client.

---

## Phase 3 — Authentication Bypass

<img src="docs/assets/03-auth-bypass.png" width="100%" alt="Authentication Bypass">

The login request is intercepted before reaching the server.

### Vulnerability

**NoSQL Injection**

The backend fails to validate user-controlled query operators.

### Technical Explanation

MongoDB accepts operators inside JSON-like objects.

When user input is parsed directly into a database query, operators can alter authentication logic.

Instead of supplying expected credentials, specially crafted parameters manipulate the query conditions and bypass normal authentication checks.

### Security Impact

- Authentication bypass.
- Unauthorized session creation.
- Access to restricted application features.

---

## Phase 4 — Authenticated Session

<img src="docs/assets/04-session-cookie.png" width="100%" alt="Session Cookie">

After forwarding the manipulated request, the application issues an authenticated session cookie.

### Why It Matters

The `connect.sid` cookie represents:

- Logged-in user state.
- Authorization context.
- Session persistence.

Once issued, the browser gains access to staff-only resources.

### Security Observation

Improper authentication validation compromises the application's authorization model.

---

## Phase 5 — Staff Console Access

<img src="docs/assets/05-staff-console.png" width="100%" alt="Staff Console">

The authenticated session unlocks the **Cabana Desk**.

### Staff Feature

Employees can customize booking confirmation messages using **Embedded JavaScript (EJS)** templates.

This feature becomes the next attack surface.

---

## Phase 6 — Server-Side Template Injection Discovery

<img src="docs/assets/06-ssti-test.png" width="100%" alt="SSTI Test">

A harmless arithmetic expression is rendered.

### Result

The server evaluates the expression instead of displaying it as plain text.

### Vulnerability Confirmed

**Server-Side Template Injection**

### Why This Is Dangerous

If untrusted users control template expressions:

- JavaScript executes on the server.
- Sensitive objects become accessible.
- Operating system commands may become reachable through Node.js APIs.

---

## Phase 7 — Remote Code Execution

<img src="docs/assets/07-command-execution.png" width="100%" alt="Command Execution">

The SSTI vulnerability is leveraged to interact with the underlying Node.js runtime.

### Objective

Confirm operating system command execution.

### Observation

The server executes JavaScript inside the application process.

### Impact

- Command execution.
- Environment inspection.
- File access.
- Process interaction.

> **Exploit payload intentionally omitted for responsible disclosure.**

---

## Phase 8 — User Flag Retrieval (Redacted)

<img src="docs/assets/08-user-flag.png" width="100%" alt="User Flag">

The application can now access protected resources available to the compromised account.

### Portfolio Version

```text
THM{******************************}
```

The flag has been intentionally hidden.

---
