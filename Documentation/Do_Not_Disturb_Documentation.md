# Do Not Disturb — TryHackMe Technical Walkthrough

> Professional penetration testing documentation for the **Do Not Disturb** TryHackMe room.

---

![Banner](../docs/assets/room-banner.png)

<div align="center">

## Hacker Holidays — Byte Lotus Hotel

### Web Exploitation • NoSQL Injection • SSTI • Node.js Inspector • Linux Privilege Escalation

</div>

---

## Document Information

| Field                     | Value                                |
| ------------------------- | ------------------------------------ |
| **Room**                  | Do Not Disturb                       |
| **Platform**              | TryHackMe                            |
| **Category**              | Web Security / Privilege Escalation  |
| **Difficulty**            | Medium                               |
| **Operating System**      | Linux                                |
| **Application Stack**     | Express.js • MongoDB • EJS • Node.js |
| **Author**                | Anurag Revankar                      |


---

# Executive Summary

## Overview

**Do Not Disturb** is a web application security challenge from the **Hacker Holidays** series on TryHackMe. The room demonstrates how multiple vulnerabilities inside a modern Node.js application can be chained together into a complete system compromise.

Rather than exploiting a single weakness, the assessment follows a realistic attacker workflow:

1. Discover hidden application functionality.
2. Bypass authentication using a NoSQL Injection vulnerability.
3. Abuse an authenticated staff feature vulnerable to Server-Side Template Injection.
4. Execute arbitrary JavaScript on the server.
5. Gain remote shell access.
6. Enumerate localhost-only services.
7. Abuse an exposed Node.js Inspector instance.
8. Escalate privileges by leveraging Linux filesystem permissions.

The room provides an excellent introduction to **Express.js security**, **MongoDB authentication flaws**, **EJS SSTI**, and **Linux privilege escalation**.

---

## Assessment Objectives

The objective of this engagement was to simulate an attacker attempting to compromise the Byte Lotus Hotel booking portal and obtain privileged access.

### Goals

* Perform reconnaissance against the web application.
* Identify hidden resources.
* Gain authenticated access.
* Validate template injection.
* Achieve command execution.
* Enumerate internal services.
* Escalate privileges.
* Retrieve protected artifacts.

> **Note:** All flags shown throughout this report have been intentionally redacted for educational integrity.

---

# Target Environment

## Target Application

![Homepage](../docs/assets/00-room-homepage.png)

The application represents a fictional hotel booking platform called **Byte Lotus Hotel**.

The platform contains separate interfaces for guests and hotel staff.

### Application Components

| Component              | Description                       |
| ---------------------- | --------------------------------- |
| Guest Booking Portal   | Public-facing reservation system. |
| Staff Portal           | Restricted employee dashboard.    |
| Confirmation Templates | Custom booking message editor.    |
| Session Management     | Express session cookies.          |
| Backend                | Node.js + Express.js.             |
| Database               | MongoDB authentication backend.   |

---

## Technology Stack Analysis

The challenge is intentionally designed around technologies commonly found in production web applications.

| Technology | Purpose                         |
| ---------- | ------------------------------- |
| Node.js    | Runtime environment.            |
| Express.js | Web framework.                  |
| MongoDB    | Authentication datastore.       |
| EJS        | HTML template rendering engine. |
| Linux      | Host operating system.          |

---

## Attack Surface Summary

The following attack surfaces were identified during reconnaissance.

| Surface             | Risk                         |
| ------------------- | ---------------------------- |
| Public HTTP Service | Initial entry point.         |
| Hidden Staff Portal | Authentication target.       |
| Template Editor     | SSTI attack surface.         |
| Session Cookies     | Authorization context.       |
| Local Debug Port    | Privilege escalation vector. |

---

# Attack Chain Overview

## Complete Kill Chain

```text
Internet
   │
   ▼
HTTP Enumeration
   │
Gobuster Discovery
   │
Hidden Staff Endpoint
   │
Burp Suite Interception
   │
NoSQL Injection
   │
Authenticated Session
   │
Staff Console
   │
Server-Side Template Injection
   │
Remote Code Execution
   │
Reverse Shell
   │
Local Enumeration
   │
Node Inspector
   │
Privilege Escalation
   │
Protected File Access
```

---

## Attack Methodology

The assessment follows a structured penetration testing methodology.

### Phase 1

Reconnaissance and Enumeration.

### Phase 2

Authentication Testing.

### Phase 3

Application Exploitation.

### Phase 4

Remote Code Execution.

### Phase 5

Post Exploitation.

### Phase 6

Privilege Escalation.

Each phase is documented independently throughout this report.

---

# Phase 1 — Reconnaissance & Enumeration

Reconnaissance is the first stage of every penetration test.

The objective is to identify:

* Hidden endpoints.
* Administrative functionality.
* APIs.
* Login portals.
* Sensitive resources.

---

## Objective

Discover hidden directories exposed by the web server.

---

## Directory Enumeration

![Gobuster](../docs/assets/01-gobuster.png)

The application was scanned using a directory brute-force wordlist to identify unlinked resources.

### Purpose of Enumeration

Directory enumeration helps identify functionality that is not visible through normal navigation.

Examples include:

* Administrative interfaces.
* Backup folders.
* APIs.
* Authentication pages.
* Development endpoints.

---

### Enumeration Results

Two interesting resources were discovered.

| Endpoint  | Observation                              |
| --------- | ---------------------------------------- |
| `/staff`  | Restricted interface returning HTTP 403. |
| `/logout` | Existing authenticated functionality.    |

The `/staff` endpoint immediately became the primary target because:

* It exists.
* It is protected.
* It is likely intended for hotel employees.

---

### Security Observation

Returning **403 Forbidden** instead of **404 Not Found** reveals that the endpoint exists.

This allows attackers to distinguish between:

* Missing resources.
* Protected resources.

This information disclosure assists reconnaissance.

---

### Blue Team Note

A safer implementation would minimize unnecessary endpoint disclosure and restrict administrative routes behind proper authentication and monitoring.

---

# Phase 2 — HTTP Traffic Analysis

## Objective

Intercept and inspect authentication traffic before it reaches the application.

---

## Configuring Burp Suite

![FoxyProxy](../docs/assets/02-foxyproxy.png)

The browser was configured to proxy HTTP traffic through Burp Suite.

### Why Burp Suite?

Burp Suite provides visibility into:

* HTTP Requests.
* HTTP Responses.
* Cookies.
* Headers.
* Parameters.
* Authentication flows.

This enables security testing without modifying the browser itself.

---

## Request Interception Workflow

1. Browser sends request.
2. FoxyProxy redirects traffic.
3. Burp intercepts request.
4. Tester modifies request.
5. Request forwarded to server.

This workflow allows controlled manipulation of application input.

---

## Captured Authentication Request

![Burp Request](../docs/assets/03-auth-bypass.png)

The login request contains two user-controlled fields.

### Observations

* POST request.
* Form encoded body.
* Username parameter.
* Password parameter.

The request becomes the primary attack surface for authentication testing.

---

# Phase 3 — Authentication Bypass (NoSQL Injection)

Authentication is implemented using MongoDB.

Instead of validating user input, the application directly processes user-controlled values inside a database query.

---

## Vulnerability Overview

### Classification

**NoSQL Injection**

### Severity

**High**

### CWE Mapping

CWE-943 — Improper Neutralization of Special Elements in Data Query Logic.

---

## Root Cause

MongoDB supports query operators inside objects.

If user input is parsed into an object instead of treated as a string, operators can modify authentication behavior.

### Why This Happens

Instead of comparing strings:

```javascript
username === "attendant"
```

The backend evaluates a MongoDB query object.

This changes authentication logic.

---

## Authentication Logic Breakdown

### Intended Flow

```text
Browser
   │
Username
Password
   │
MongoDB Query
   │
Matching User
   │
Authenticated
```

### Vulnerable Flow

```text
Browser
   │
User Controlled Query Operator
   │
MongoDB Query Parser
   │
Modified Query
   │
Authentication Bypass
```

---

## Impact

The vulnerability allows an attacker to:

* Skip password validation.
* Authenticate as an existing user.
* Receive a valid session.
* Access staff-only functionality.

---

## Why This Is Dangerous

Authentication is the primary security boundary protecting privileged functionality.

Once bypassed, every authenticated feature becomes exposed.

This includes administrative dashboards and internal tools.

---

## Evidence

![Authentication Success](../docs/assets/04-session-cookie.png)

After forwarding the modified request, the server responds with an authenticated session cookie.

---

## Session Analysis

The application uses Express session management.

### Session Cookie

`connect.sid`

### Purpose

The cookie identifies an authenticated session maintained by the server.

### Security Discussion

Session cookies should only be issued after successful authentication.

In this scenario, flawed authentication logic results in an authenticated session for an unauthorized user.

---

### Blue Team Detection

Potential indicators include:

* Authentication without valid credentials.
* MongoDB operators appearing inside POST parameters.
* Unusual login success events.

---

# Phase 4 — Staff Console Discovery

## Objective

Explore newly accessible functionality.

---

## Staff Dashboard

![Cabana Desk](../docs/assets/05-staff-console.png)

The authenticated session exposes an internal employee dashboard called **Cabana Desk**.

### Purpose of Dashboard

Staff members customize guest booking confirmation messages before reservations are sent.

### Available Functionality

* Template editing.
* Preview rendering.
* Guest personalization.
* Booking confirmation generation.

This template preview becomes the next attack surface.

---

## Initial Security Assessment

The dashboard contains an editable confirmation template.

A note indicates that personalization is performed using **EJS** syntax.

This immediately suggests possible server-side template rendering.

---

## Threat Model

Any application allowing users to edit templates should be treated as high risk.

Questions to investigate include:

* Are expressions evaluated?
* Is user input escaped?
* Can JavaScript execute?
* Can application objects be accessed?

The next phase answers these questions.

---

# Phase 5 — Server-Side Template Injection Discovery

Server-Side Template Injection occurs when user input is processed as executable template code instead of plain text.

---

## Objective

Determine whether EJS expressions are evaluated by the server.

---

## Validation Test

![SSTI Test](../docs/assets/06-ssti-test.png)

A harmless arithmetic expression was inserted into the confirmation template.

### Result

The preview displays the computed value rather than the literal expression.

### Conclusion

The application evaluates template expressions on the server.

This confirms a **Server-Side Template Injection (SSTI)** vulnerability.

---

## Why This Test Matters

A safe validation payload demonstrates code execution without affecting the server.

It establishes:

* Template execution.
* Rendering context.
* Evaluation behavior.

This is a common first step during SSTI testing.

---

## Understanding EJS

Embedded JavaScript templates render HTML dynamically.

Example behavior:

* Variables inserted.
* Expressions evaluated.
* JavaScript executed during rendering.

This flexibility becomes dangerous when users control template content.

---

## Security Impact

Once SSTI is confirmed, attackers may attempt to access:

* Environment variables.
* Node.js globals.
* File system APIs.
* Operating system commands.

This significantly expands the attack surface.

---

---

# Phase 6 — Remote Code Execution via EJS SSTI

Once Server-Side Template Injection was confirmed, the next objective was to determine whether the template engine could access the underlying **Node.js runtime** and execute operating system commands.

This phase transitions from **template injection** to **remote code execution**, significantly increasing the attacker's capabilities.

---

## Objective

Validate whether the SSTI vulnerability provides access to Node.js APIs capable of interacting with the operating system.

---

## Understanding the Execution Context

![Command Execution](../docs/assets/07-command-execution.png)

Unlike client-side template engines, **EJS renders templates on the server** before sending HTML to the browser.

During rendering, the template has access to the JavaScript execution environment of the application.

### Why This Matters

A vulnerable EJS template may expose access to:

* JavaScript expressions.
* Global objects.
* Environment variables.
* Node.js modules.
* Operating system interfaces.

This effectively turns template rendering into a code execution primitive.

---

## Technical Explanation

### Server-Side Rendering Flow

```text
Browser
   │
Template Input
   │
Express.js
   │
EJS Rendering Engine
   │
JavaScript Evaluation
   │
Generated HTML Response
```

The template engine evaluates JavaScript **before** generating the HTML response.

If user input is inserted into executable template blocks, arbitrary JavaScript executes inside the application's process.

---

## Evidence of Code Execution

The template was modified with a controlled JavaScript expression to verify access to runtime functionality.

The response confirmed that the application executed JavaScript during rendering rather than treating the input as plain text.

### Observation

* Expression evaluated successfully.
* Server generated output dynamically.
* JavaScript execution confirmed.

---

## Security Impact

This vulnerability expands attacker capabilities from authenticated user access to:

* Reading server-side files.
* Inspecting environment variables.
* Executing system commands.
* Launching additional processes.
* Accessing network interfaces.

### Severity Assessment

| Attribute               | Assessment         |
| ----------------------- | ------------------ |
| CVSS Impact             | Critical           |
| Authentication Required | Yes (after bypass) |
| Code Execution          | Yes                |
| Confidentiality         | High               |
| Integrity               | High               |
| Availability            | High               |

---

## Why Node.js SSTI is Dangerous

Node.js applications expose powerful runtime modules.

Examples include:

| Capability            | Node.js Module  |
| --------------------- | --------------- |
| File Access           | `fs`            |
| Child Processes       | `child_process` |
| Environment Variables | `process.env`   |
| Network Access        | `net`           |
| Operating System      | `os`            |

If template execution reaches these modules, the attacker gains extensive control over the application process.

---

## Blue Team Perspective

Indicators of SSTI include:

* Template syntax appearing in user input.
* Unexpected rendering errors.
* JavaScript exceptions during rendering.
* Dynamic evaluation logs.

### Defensive Recommendations

* Escape user-controlled template content.
* Disable template editing for untrusted users.
* Render templates from trusted server-side files only.
* Apply strict input validation.

---

# Phase 7 — Initial Shell Access

Server-side command execution enables post-exploitation activities.

The next objective is to obtain an **interactive shell** on the target machine.

---

## Objective

Establish an interactive command-line session as the compromised application user.

---

## Reverse Shell Listener

![Netcat Listener](../docs/assets/09-netcat-listener.png)

A listener is configured on the attack machine before triggering the reverse shell from the application.

### Why a Reverse Shell?

Executing individual commands through SSTI is possible but inefficient.

A reverse shell provides:

* Interactive Bash session.
* Command history.
* Process interaction.
* Easier enumeration.
* Privilege escalation workflow.

---

## Reverse Shell Connection

![Shell Connected](../docs/assets/10-shell-connected.png)

The target successfully connects back to the listener.

### Initial Access Achieved

The shell belongs to an unprivileged service account responsible for running the hotel booking application.

### Initial Privilege Level

| Property          | Value                    |
| ----------------- | ------------------------ |
| User Context      | Application Service User |
| Shell Type        | Interactive Bash         |
| Working Directory | Application Directory    |
| Privilege Level   | Non-root                 |

---

## Post Exploitation Begins

After obtaining shell access, the engagement shifts from application exploitation to **host enumeration**.

Typical enumeration targets include:

* Current user.
* Groups.
* Processes.
* Network interfaces.
* Listening services.
* Sensitive files.

---

## Security Observation

Remote code execution does **not** necessarily imply administrative privileges.

Many production applications execute as dedicated service accounts with restricted permissions.

Privilege escalation therefore becomes a separate phase.

---

# Phase 8 — Local Enumeration

Enumeration is the process of identifying additional attack vectors after gaining initial access.

---

## Objective

Discover locally exposed services and privilege escalation opportunities.

---

## Enumerating Listening Services

![Listening Services](../docs/assets/11-ss-tlnu.png)

Listening sockets reveal services that are inaccessible externally but reachable locally.

### Why Enumerate Local Ports?

Services bound to `127.0.0.1` cannot be reached from the internet.

However, once local shell access exists, these services become available.

Examples include:

* Databases.
* Redis.
* Docker.
* Jenkins.
* Node Inspector.
* Internal APIs.

---

## Key Discovery

A TCP service is listening on:

`127.0.0.1:9229`

### Why This Port Stands Out

Port **9229** is the default debugging interface for Node.js applications.

### Risk Assessment

| Property            | Value           |
| ------------------- | --------------- |
| Accessible Remotely | No              |
| Accessible Locally  | Yes             |
| Authentication      | None observed   |
| Potential Impact    | Runtime control |

---

## Localhost Attack Surface

```text
Internet
   │
Port 80
   │
Web Application
   │
──────────────
Localhost Only
   │
127.0.0.1:9229
```

This demonstrates an important penetration testing principle:

> Local access often exposes an entirely different attack surface than external reconnaissance.

---

## Blue Team Insight

Developers frequently expose debugging services assuming localhost is trusted.

This assumption becomes invalid after initial compromise.

### Recommended Mitigations

* Disable debugging interfaces.
* Restrict localhost services.
* Monitor debugger connections.
* Use firewall rules.

---

# Phase 9 — Investigating Node.js Inspector

The discovered debugging service becomes the primary privilege escalation target.

---

## Objective

Investigate the Node.js Inspector runtime.

---

## Connecting to the Debugger

![Node Inspector REPL](../docs/assets/12-node-inspector.png)

A debugger connection establishes an interactive JavaScript REPL attached to the running Node.js process.

### What is the REPL?

REPL stands for:

* Read
* Evaluate
* Print
* Loop

It allows JavaScript expressions to execute inside the application's process.

---

## Why This is Powerful

Instead of interacting through SSTI, the attacker now interacts directly with the application's runtime.

Capabilities include:

* Evaluate JavaScript.
* Inspect objects.
* Call functions.
* Access built-in modules.
* Inspect memory.

---

## Runtime Context Verification

![UID and GID](../docs/assets/13-uid-gid.png)

The debugger exposes process identity information.

### Findings

* Different user context.
* Different group context.
* Separate runtime permissions.

This confirms the debugger is attached to another application process.

---

## Security Analysis

Applications commonly separate responsibilities between service accounts.

Examples include:

| Service       | Purpose             |
| ------------- | ------------------- |
| Web User      | HTTP requests       |
| Worker User   | Background jobs     |
| Pipeline User | Automation tasks    |
| Database User | Internal processing |

Attaching to a privileged runtime inherits those permissions.

---

# Phase 10 — Runtime Process Analysis

The debugger provides visibility into the process owner.

---

## Service Account Investigation

![pipelinesvc Context](../docs/assets/14-pipelinesvc.png)

The Node.js process executes under a dedicated service account.

### Why This Matters

Different accounts often have:

* Different filesystem permissions.
* Different Linux groups.
* Access to internal resources.

---

## Linux Groups

One additional group membership becomes especially important.

| Group  | Security Impact          |
| ------ | ------------------------ |
| `disk` | Raw block device access. |

### Why the disk Group is Sensitive

Members of the `disk` group can often interact directly with storage devices.

Examples include:

* NVMe devices.
* SATA devices.
* Virtual disks.
* Filesystem partitions.

This creates a privilege escalation opportunity.

---

## Principle of Least Privilege

This service account violates least privilege because the web application's runtime should not require direct block device access.

### Blue Team Recommendation

* Remove unnecessary Linux groups.
* Separate application permissions.
* Audit privileged service accounts.

---

# Phase 11 — Enumerating Block Devices

Understanding accessible storage devices is essential before attempting filesystem analysis.

---

## Filesystem Enumeration

![Disk Enumeration](../docs/assets/15-debugfs-discovery.png)

The system exposes multiple block devices.

### Why Enumerate Storage?

Attackers look for:

* Mounted partitions.
* Root filesystem.
* Backup partitions.
* Additional volumes.

### Observations

* Primary filesystem partition identified.
* Raw device accessible.
* Filesystem debugging becomes possible.

---

## Understanding Block Devices

Linux exposes disks through `/dev`.

Examples include:

| Device           | Description  |
| ---------------- | ------------ |
| `/dev/sda`       | SATA Disk    |
| `/dev/vda`       | Virtual Disk |
| `/dev/nvme0n1`   | NVMe SSD     |
| `/dev/nvme0n1p1` | Partition    |

Accessing these devices bypasses ordinary file APIs.

---

## Why This Matters

Normal file access follows:

```text
Application
   │
Kernel Permission Checks
   │
Filesystem
```

Raw device access interacts with the filesystem below normal permission enforcement.

---

# Phase 12 — Privilege Escalation via debugfs

The final stage abuses Linux filesystem debugging functionality.

---

## Objective

Access protected filesystem content through raw disk permissions.

---

## Understanding debugfs

`debugfs` is a debugging utility for ext-family Linux filesystems.

Capabilities include:

* Read files.
* Browse directories.
* Inspect metadata.
* Read inode contents.

### Why It Is Dangerous

When executed against a block device, `debugfs` bypasses ordinary filesystem permission checks.

---

## Privilege Escalation Concept

Instead of opening protected files through the kernel:

```text
User
 │
Permission Check
 │
Denied
```

The filesystem is read directly:

```text
User
 │
Raw Block Device
 │
Filesystem Structures
 │
Protected File Contents
```

### Security Consequence

The attacker reads sensitive files without possessing root privileges.

---

## Evidence

![Root Flag Redacted](../docs/assets/16-root-flag.png)

### Portfolio Version

```text
THM{*******************************}
```

The root flag has been intentionally hidden.

---

## Root Cause Analysis

This privilege escalation relies on **two independent misconfigurations**:

1. Exposed debugging interface.
2. Overprivileged service account.

Either issue alone is significant.

Combined, they allow privileged filesystem access.

---

## Lessons Learned

* Debug services should never remain enabled in production.
* Service accounts require minimal permissions.
* Localhost services must be monitored.
* Filesystem debugging tools require strict access controls.

---

---

# Deep Technical Analysis

This section explains the vulnerabilities exploited during the engagement from both an offensive security and defensive security perspective. Rather than focusing on exploit payloads, it explains **why the vulnerabilities exist**, **how they work internally**, and **how they should be mitigated**.

---

# Vulnerability 1 — NoSQL Injection Authentication Bypass

## Overview

NoSQL Injection occurs when user-controlled input is interpreted as part of a database query instead of as plain application data.

Unlike SQL Injection, which manipulates SQL statements, NoSQL Injection targets query objects used by databases such as **MongoDB**.

### Severity

**High**

### ATT&CK Category

Initial Access

---

## Understanding MongoDB Queries

MongoDB queries are represented as JavaScript-like objects.

For example:

```javascript
{
    username: "employee",
    password: "password123"
}
```

The application should compare literal values.

Instead, the backend accepts structured query operators from user input.

---

## Root Cause

The login endpoint trusts user input without validating its structure.

### Authentication Flow

```text
Client
   │
Username + Password
   │
Application
   │
MongoDB Query Object
   │
Database Match
```

When input is parsed as an object, MongoDB operators alter query behavior.

---

## Security Impact

Improper input validation allows attackers to:

* Bypass login.
* Authenticate as existing users.
* Receive valid sessions.
* Access privileged application functionality.

---

## Why This Happens in Node.js

Applications built with Express and MongoDB often deserialize incoming request bodies automatically.

Without validation libraries or sanitization middleware, operators may reach the database unchanged.

---

## Detection Opportunities

Blue Team defenders should monitor for:

* Unexpected MongoDB operators.
* Authentication success without valid credentials.
* Malformed JSON objects.
* Unusual request bodies.

### Detection Sources

| Source              | Detection Opportunity                 |
| ------------------- | ------------------------------------- |
| Application Logs    | Unexpected authentication parameters. |
| WAF                 | MongoDB operator signatures.          |
| IDS/IPS             | Injection patterns.                   |
| Authentication Logs | Suspicious login behavior.            |

---

## Defensive Recommendations

* Validate request schemas.
* Reject unexpected objects.
* Treat credentials strictly as strings.
* Sanitize MongoDB operators.
* Implement authentication logging.

---

# Vulnerability 2 — Server-Side Template Injection (EJS)

## Overview

Server-Side Template Injection (SSTI) occurs when user-controlled input is evaluated by the template engine instead of being rendered as plain text.

### Severity

**Critical**

### Technology

Embedded JavaScript Templates (EJS)

---

## How EJS Works

EJS allows developers to embed JavaScript directly into HTML templates.

### Rendering Pipeline

```text
Template
   │
JavaScript Evaluation
   │
HTML Generation
   │
Browser Response
```

The template engine executes JavaScript during rendering.

---

## Why This Is Dangerous

User-controlled templates may access:

* Variables.
* Objects.
* Environment variables.
* Runtime APIs.
* Built-in Node modules.

---

## Impact Assessment

SSTI frequently results in:

* Information Disclosure.
* Remote Code Execution.
* File Access.
* Command Execution.
* Privilege Escalation.

---

## Security Observations

The application allows authenticated staff to customize booking confirmation templates.

This feature introduces executable server-side content without sanitization.

---

## Detection Opportunities

Indicators include:

* Template syntax inside user input.
* Rendering exceptions.
* Unexpected JavaScript output.
* Runtime evaluation logs.

---

## Mitigation Strategies

* Never evaluate user-controlled templates.
* Escape template syntax.
* Disable dynamic template editing.
* Restrict rendering context.
* Use sandboxed template environments.

---

# Vulnerability 3 — Remote Code Execution

## Overview

The confirmed SSTI vulnerability allows JavaScript execution inside the application's runtime.

### Why RCE Matters

Remote Code Execution moves the attacker beyond the web application layer.

Capabilities expand to:

* Process interaction.
* File operations.
* Network communication.
* Local system enumeration.

---

## Node.js Runtime Exposure

Node.js provides APIs capable of interacting with the operating system.

Examples include:

| Module          | Capability                    |
| --------------- | ----------------------------- |
| `fs`            | Filesystem access.            |
| `os`            | Operating system information. |
| `process`       | Runtime context.              |
| `child_process` | Launch external programs.     |

The room demonstrates why exposing these APIs through template injection is dangerous.

---

## Defensive Recommendations

* Restrict runtime APIs.
* Avoid executing templates containing JavaScript.
* Validate rendered content.
* Disable template evaluation for untrusted users.

---

# Vulnerability 4 — Exposed Node.js Inspector

## Overview

The Node.js Inspector is intended exclusively for debugging during development.

### Default Port

`9229`

---

## What Inspector Provides

* Runtime inspection.
* Variable inspection.
* Memory analysis.
* Interactive REPL.
* JavaScript execution.

### Why It Is Sensitive

Anyone able to connect gains interaction with the application's execution context.

---

## Production Misconfiguration

Leaving Inspector enabled in production exposes powerful debugging capabilities.

### Risk

* Runtime modification.
* Credential exposure.
* Environment disclosure.
* Process inspection.
* Privilege escalation opportunities.

---

## Detection Opportunities

Blue Teams should monitor:

* Connections to localhost debugger ports.
* Node debugging processes.
* Unexpected REPL activity.
* Debugger startup arguments.

---

## Hardening Recommendations

* Disable Inspector in production.
* Bind debugger only when necessary.
* Restrict localhost debugging.
* Authenticate debugger access.

---

# Vulnerability 5 — Linux Privilege Escalation

## Overview

The compromised Node.js process executes under a service account belonging to the `disk` group.

### Why This Matters

The `disk` group has access to raw storage devices.

---

## Linux Group Permissions

| Group  | Typical Capability          |
| ------ | --------------------------- |
| root   | Full administrative access. |
| sudo   | Administrative commands.    |
| docker | Docker daemon access.       |
| disk   | Raw block device access.    |

Membership in privileged groups should be minimized.

---

## Filesystem Debugging

Filesystem debugging utilities operate below ordinary file permission enforcement.

### Security Risk

An attacker may read filesystem contents directly through block devices.

---

## Principle Violated

**Least Privilege**

Application service accounts should only possess permissions required for application functionality.

---

## Defensive Recommendations

* Remove unnecessary group memberships.
* Audit privileged service accounts.
* Restrict filesystem debugging utilities.
* Monitor execution of debugging binaries.

---

# Indicators of Compromise (IoCs)

This section summarizes observable attacker behavior.

## Network Indicators

| IOC                            | Description                            |
| ------------------------------ | -------------------------------------- |
| Directory enumeration requests | Multiple requests to hidden endpoints. |
| Authentication manipulation    | Unusual POST bodies.                   |
| Reverse shell connection       | Unexpected outbound TCP connection.    |
| Local debugger connection      | Access to port 9229.                   |

---

## Application Indicators

| IOC                           | Detection Opportunity  |
| ----------------------------- | ---------------------- |
| Template syntax in requests   | SSTI attempts.         |
| Unexpected template rendering | Template abuse.        |
| Authentication anomalies      | Login bypass behavior. |
| Session creation anomalies    | Unauthorized sessions. |

---

## Host Indicators

| IOC                        | Detection Source           |
| -------------------------- | -------------------------- |
| Node Inspector connection  | Host monitoring.           |
| Execution of debugfs       | Auditd / Sysmon for Linux. |
| Reverse shell process      | Process monitoring.        |
| Unexpected child processes | EDR telemetry.             |

---

# MITRE ATT&CK Mapping

| ATT&CK Tactic               | Technique                                 |
| --------------------------- | ----------------------------------------- |
| Initial Access              | T1190 — Exploit Public-Facing Application |
| Execution                   | T1059 — Command and Scripting Interpreter |
| Discovery                   | T1046 — Network Service Discovery         |
| Discovery                   | T1082 — System Information Discovery      |
| Credential / Session Access | Session Abuse                             |
| Privilege Escalation        | Abuse of Misconfigured Services           |
| Collection                  | T1005 — Data from Local System            |

---

## ATT&CK Kill Chain Visualization

```text
Initial Access
      │
      ▼
NoSQL Injection
      │
      ▼
Execution
      │
      ▼
SSTI → JavaScript Execution
      │
      ▼
Remote Shell
      │
      ▼
Discovery
      │
      ▼
Node Inspector
      │
      ▼
Privilege Escalation
      │
      ▼
Protected Filesystem Access
```

---

# OWASP Top 10 Mapping

| OWASP Category | Application in Room                        |
| -------------- | ------------------------------------------ |
| A01            | Broken Access Control                      |
| A03            | Injection                                  |
| A05            | Security Misconfiguration                  |
| A07            | Identification and Authentication Failures |
| A09            | Security Logging and Monitoring Failures   |

---

## Why Multiple OWASP Categories Apply

This room intentionally chains together several independent weaknesses.

* Authentication failure enables access.
* Injection enables execution.
* Misconfiguration enables escalation.
* Monitoring failures allow attacker activity to go unnoticed.

---

# Blue Team Detection & Monitoring

## Detection Strategy

A layered defense should monitor:

### Web Layer

* Directory brute forcing.
* Injection attempts.
* Template syntax.
* Authentication anomalies.

### Application Layer

* Template rendering failures.
* Unexpected JavaScript execution.
* Session generation events.

### Host Layer

* Reverse shells.
* Debugger connections.
* Child process execution.
* Privileged utilities.

---

## Logging Recommendations

| Component  | Recommendation                 |
| ---------- | ------------------------------ |
| Express.js | Structured request logging.    |
| MongoDB    | Authentication audit logging.  |
| Linux      | Audit privileged binaries.     |
| Firewall   | Alert on debugger connections. |
| EDR        | Monitor process spawning.      |

---

# Security Hardening Recommendations

## Authentication Security

* Validate request schemas.
* Reject MongoDB operators.
* Use strict typing.
* Enable MFA where appropriate.

---

## Template Security

* Escape template expressions.
* Disable user-controlled templates.
* Sandbox rendering engines.
* Restrict JavaScript execution.

---

## Node.js Security

* Disable Inspector in production.
* Remove debugging arguments.
* Protect environment variables.
* Restrict runtime APIs.

---

## Linux Hardening

* Remove `disk` group membership.
* Restrict filesystem utilities.
* Monitor raw device access.
* Audit privileged service accounts.

---

## Session Security

* Regenerate session identifiers after login.
* Enable Secure cookies.
* Enable HttpOnly.
* Enable SameSite protection.
* Set appropriate expiration times.

---

# Lessons Learned

This room demonstrates several important penetration testing concepts.

## Offensive Security Lessons

* Hidden endpoints frequently expose privileged functionality.
* Authentication logic should always be tested for injection.
* Template engines are high-risk attack surfaces.
* Localhost services become attack vectors after initial compromise.
* Linux group memberships require enumeration during privilege escalation.

---

## Defensive Security Lessons

* Debug interfaces must never remain enabled in production.
* Least privilege should apply to application service accounts.
* User-controlled template rendering should be avoided.
* Application logging should capture abnormal authentication behavior.
* Internal services require monitoring even when bound to localhost.

---

# Skills Gained

## Web Security

* HTTP interception.
* Burp Suite workflow.
* Authentication testing.
* Session analysis.
* Template injection discovery.

## Node.js Security

* Express authentication flow.
* EJS rendering internals.
* Runtime process inspection.
* Debugger analysis.

## Linux Privilege Escalation

* Local service enumeration.
* Group permission analysis.
* Filesystem debugging concepts.
* Service account enumeration.

## Defensive Security

* MITRE ATT&CK mapping.
* OWASP categorization.
* Detection engineering.
* Security hardening.

---

# References

The concepts discussed throughout this documentation are based on publicly documented security guidance.

## Frameworks

* MITRE ATT&CK Enterprise Framework.
* OWASP Top 10.
* OWASP Web Security Testing Guide.

## Technologies

* Node.js Documentation.
* Express.js Documentation.
* MongoDB Security Documentation.
* EJS Documentation.

## Linux Security

* Linux File Permissions.
* debugfs Manual Pages.
* Linux Audit Framework Documentation.

---

# Responsible Disclosure

This repository is intended **solely for educational and defensive cybersecurity purposes**.

### Scope

* All testing occurred inside the authorized TryHackMe environment.
* No production systems were targeted.
* Sensitive challenge flags have been removed.
* Exploit payloads have been intentionally redacted.

---

# Conclusion

The **Do Not Disturb** room demonstrates how multiple medium-severity vulnerabilities can combine into a complete system compromise when secure development practices are absent.

The assessment highlights the importance of:

* Strong authentication validation.
* Secure template rendering.
* Proper session management.
* Production hardening.
* Principle of least privilege.
* Monitoring localhost-only services.

Understanding this attack chain provides valuable insight into both offensive penetration testing techniques and defensive security engineering practices.

---

<div align="center">

## ⭐ End of Technical Walkthrough

**Professional TryHackMe Documentation for GitHub Portfolio**

Cybersecurity • Penetration Testing • Web Security • Linux Privilege Escalation

</div>
