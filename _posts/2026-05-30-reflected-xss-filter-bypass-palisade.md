---
title: "Cross-Site Scripting (XSS) – Inadequate Input Filter Bypass | Palisade"
date: 2026-05-30 00:00:00 +0530
categories: [A05 - Injection, Cross-Site Scripting (XSS)]
tags: [webversepro, xss, reflected-xss, filter-bypass, injection, input-validation]
author: Shivansh Sharma
image:
path: /assets/images/posts/palisade-reflected-xss.webp
alt: Cross-Site Scripting (XSS) – Inadequate Input Filter Bypass | Palisade
---------------------------------------------------------------------------

## Lab Link

Lab: [Palisade](https://dashboard.webverselabs-pro.com/challenges/palisade)

---

## Overview

Palisade is a mountaineering equipment rental platform that allows users to configure equipment sizing preferences before renting gear.

The application includes a note field that reflects user-controlled content back into the page. A weak filtering mechanism attempts to prevent Cross-Site Scripting (XSS) attacks by removing specific script-related strings.

Unfortunately, the protection is incomplete and relies on simple string replacement rather than proper output encoding.

As a result, attacker-controlled HTML and JavaScript are rendered directly into the page, allowing arbitrary script execution.

---

## Objective

Identify the reflected XSS vulnerability in the preferences page and execute JavaScript to obtain the flag.

---

## Vulnerability Identification

This challenge is primarily a Cross-Site Scripting vulnerability.

### Classification Hierarchy

A05 - Injection
└── Client-Side Injection
└── Cross-Site Scripting (XSS)
└── Reflected XSS

---

## Reconnaissance

Navigate to:

```text
https://06b0df68-4065-palisade-12879.challenges.webverselabs-pro.com/preferences
```

The page contains a sizing preference form that submits user-supplied information.

A captured request appears as:

```http
POST /preferences HTTP/2

boot=36&torso=M&height=&note=testing
```

The `note` parameter is reflected back into the response.

Whenever user input is reflected into HTML output, it should be tested for XSS.

---

## Exploitation

### Step 1 - Identify Reflection

Submit a simple test value:

```text
testing
```

After submission, inspect the resulting page source.

The value appears inside the response:

```html
<div class="ps-nav-right">
    <span class="ps-badge">
        Sized for you:
        <b>boot 36, torso M · testing</b>
    </span>
</div>
```

This confirms that the `note` parameter is reflected into the HTML document.

---

### Step 2 - Analyze the Context

The user-controlled value is inserted directly inside a rendered HTML element:

```html
<b>boot 36, torso M · USER_INPUT</b>
```

Because the reflection occurs within an HTML context, injected tags may be interpreted by the browser.

---

### Step 3 - Test Script Execution

Submit the following payload in the note field:

```html
<script>alert(1)</script>
```

The payload executes successfully.

The browser interprets the injected content as executable JavaScript rather than plain text.

This confirms the presence of a reflected XSS vulnerability.

---

### Step 4 - Observe Flag Retrieval

Once the script executes, the application detects successful XSS execution and displays the flag.

```text
WEBVERSE{.....}
```

The challenge environment automatically reveals the flag after JavaScript execution is confirmed.

---

## Understanding the Root Cause

The challenge description hints that the developer attempted to block XSS using a simple string replacement:

```javascript
input.replace("<script", "")
```

This approach is fundamentally flawed because:

* It relies on blacklisting.
* It does not understand HTML parsing contexts.
* It can often be bypassed through alternative syntax.
* It fails to provide output encoding.

Proper XSS prevention requires context-aware encoding rather than string removal.

---

## Proof of Exploitation

### Vulnerable Request

```http
POST /preferences

boot=36&torso=M&height=&note=testing
```

### Reflected Output

```html
<b>boot 36, torso M · testing</b>
```

### XSS Payload

```html
<script>alert(1)</script>
```

### Result

```text
JavaScript executed successfully
```

### Flag

```text
WEBVERSE{.....}
```

---

## Impact

An attacker can execute arbitrary JavaScript within another user's browser.

Potential impact includes:

* Session theft
* Account takeover
* Credential harvesting
* DOM manipulation
* CSRF bypass
* Data exfiltration
* Malicious redirects

Because the payload executes in the victim's browser context, it inherits the victim's privileges and session state.

---

## Mitigation

### Use Output Encoding

Encode user-controlled data before rendering.

Example:

```html
&lt;script&gt;alert(1)&lt;/script&gt;
```

instead of:

```html
<script>alert(1)</script>
```

### Avoid Blacklist Filters

Do not rely on:

```javascript
replace("<script", "")
```

or similar string replacements.

Blacklist-based filtering is unreliable and easy to bypass.

### Use Safe Rendering Functions

Avoid:

```javascript
element.innerHTML = userInput;
```

Prefer:

```javascript
element.textContent = userInput;
```

### Implement Content Security Policy

A strong CSP can reduce the impact of XSS.

### Validate and Sanitize User Input

Apply appropriate sanitization where HTML is not required.

---

## Real-World Insight

Many XSS vulnerabilities originate from developers attempting to block attacks using ad-hoc string replacement filters.

Attackers routinely bypass such filters using:

```text
Case manipulation
Alternative HTML tags
Event handlers
Encoding tricks
Browser parsing quirks
```

Modern XSS defenses focus on safe output encoding rather than attempting to identify and remove dangerous input patterns.

The Palisade challenge demonstrates an important lesson:

**Filtering specific payloads is not the same as securing the output context.**
