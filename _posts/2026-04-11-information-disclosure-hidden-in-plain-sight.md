---
title: "Information Disclosure – Sensitive Data Exposure via Source Code, Headers & Public Files | Hidden in Plain Sight"
date: 2026-04-11 04:30:00 +0530
categories: [Web Security, Information Disclosure]
tags: [information-disclosure, sensitive-data-exposure, source-code, robots-txt, http-headers, security-txt, ctf, cybersplash2026]
platform: SecPlaygrount 
author: Shivansh Sharma
image:
  path: /assets/images/posts/information-disclosure.webp
  alt: Information Disclosure via Public Files and Headers
---

## Overview

This challenge demonstrates how sensitive information can be exposed through publicly accessible components of a web application, including source code comments, HTTP response headers, and standard files such as `robots.txt` and `security.txt`.

The objective was to identify and reconstruct a flag split into four parts and distributed across different locations within the application.

---

## Objective

- Identify all hidden flag fragments  
- Reconstruct the complete flag  

---

## Reconnaissance

Initial enumeration involved:

- Inspecting the homepage  
- Reviewing `robots.txt`  
- Accessing restricted endpoints  
- Performing basic directory exploration  

These steps revealed multiple sources of information disclosure.

---

## Exploitation

### Source Code Disclosure

Inspecting the homepage source code revealed a comment containing part of the hidden data:

```html
<!-- Development note: Part 1/4: [REDACTED] -->
```

This indicates sensitive information embedded directly within client-visible code.

### HTTP Header Disclosure

Accessing restricted endpoints returned a `403 Forbidden` response. However, inspecting the response headers exposed additional data:

```
curl -i http://<target>/admin
curl -i http://<target>/secret
```

Response:

```
X-Developer-Note: Part 2/4: [REDACTED]
```

Despite access restrictions, sensitive information was leaked through custom headers.

### robots.txt Disclosure

The `robots.txt` file contained another fragment:

```
http://<target>/robots.txt
```

```
User-agent: *
Disallow: /admin
Disallow: /secret
# Maintenance note - Part 3/4: [REDACTED]
```

This demonstrates how publicly accessible configuration files can expose internal information.

### security.txt Discovery

The final fragment was expected to be located in the `.well-known/security.txt` file:

```
http://<target>/.well-known/security.txt
```

However, the complete data could not be fully reconstructed as the final part was not successfully retrieved during this attempt.

---

## Proof of Findings

Multiple instances of sensitive data exposure were identified across:

- Source code comments
- HTTP response headers
- Public configuration files

These confirm the presence of widespread information disclosure within the application.

---

## Impact

This challenge highlights several common real-world risks:

- Exposure of sensitive data through source code comments
- Leakage of information via HTTP response headers
- Disclosure through publicly accessible files
- Inadequate separation between internal and external data

Such issues can lead to:

- Credential exposure
- Discovery of hidden endpoints
- Increased attack surface for further exploitation

---

## Mitigation

- Avoid embedding sensitive data in source code comments
- Remove unnecessary or sensitive HTTP headers
- Restrict access to configuration and metadata files
- Implement proper access control mechanisms on the server side
- Regularly audit publicly accessible resources

---

## Real-World Insight

Information disclosure is often underestimated but remains one of the most common weaknesses in web applications. Sensitive data is frequently exposed unintentionally through development artifacts, misconfigurations, or debugging remnants.

These issues align with risks identified by OWASP, particularly under sensitive data exposure and security misconfiguration.

---

## Conclusion

This challenge demonstrates how small oversights across multiple components of an application can collectively lead to significant information exposure. A structured reconnaissance process and attention to detail are essential when identifying such vulnerabilities.