---
title: "Path Traversal Leads to Arbitrary File Read | Ohmly"
date: 2026-07-11 06:10:00 +0530
categories: [A01 - Broken Access Control, Path Traversal]
tags: [path-traversal, local-file-inclusion, arbitrary-file-read, php, webverse, webverse-pro]
platform: WebVerse Pro
author: Shivansh Sharma
image:
  path: /assets/images/posts/ohmly-path-traversal.webp
  alt: "Ohmly path traversal vulnerability"
---

## Lab Link

[Ohmly](https://dashboard.webverselabs-pro.com/mystery-challenges/ohmly)

---

## Overview

**Ohmly** was a WebVerse Pro mystery challenge built around a small electronics component catalog. Each product page included a downloadable PDF datasheet, and the application used a user-controlled query parameter to select which document should be returned.

The download endpoint accepted filenames through the `doc` parameter but failed to restrict the resolved file path to the intended datasheet directory. By supplying directory traversal sequences such as `../`, it was possible to escape the document folder and read arbitrary files from the server.

After confirming the issue by retrieving `/etc/passwd`, the same vulnerability was used to access the flag file.

---

## Objective

The objective was to identify the vulnerability in the datasheet download feature and use it to retrieve the challenge flag.

---

## Vulnerability Identification

The application exposed datasheet links in the following format:

```http
GET /datasheet?doc=ne555.pdf HTTP/1.1
Host: <redacted>
```

The `doc` parameter appeared to be passed directly into a server-side file-reading operation.

A secure implementation should map a fixed product identifier to an approved file or validate the final resolved path before serving it. Ohmly instead trusted the supplied filename, creating a path traversal vulnerability.

---

## Recon and Approach

The initial page presented a small catalog of electronic components. Product pages contained links to PDF datasheets, which pointed to the `/datasheet` endpoint.

A legitimate request looked similar to:

```http
GET /datasheet?doc=ne555.pdf HTTP/1.1
Host: <redacted>
```

Because the filename was supplied directly through a query parameter, the endpoint was tested with traversal sequences.

An initial request targeting a guessed relative flag path did not immediately return the flag:

```http
GET /datasheet?doc=../flag.txt HTTP/1.1
Host: <redacted>
```

This did not prove the endpoint was secure. The flag could simply have been located outside the reached directory depth.

To verify whether traversal was possible, a known Linux file was requested.

---

## Exploitation

### Confirming Path Traversal

The following request attempted to escape the datasheet directory and read `/etc/passwd`:

```http
GET /datasheet?doc=../../../../../../etc/passwd HTTP/1.1
Host: <redacted>
```

The server returned the contents of `/etc/passwd`, confirming that:

- traversal sequences were not removed or rejected;
- the application did not enforce a fixed base directory;
- arbitrary readable files could be accessed through the endpoint.

The response also disclosed local system account information, including an application-related user.

### Reading the Flag File

After confirming arbitrary file read, the traversal depth was reused to target the root-level flag file:

```http
GET /datasheet?doc=../../../../../../flag.txt HTTP/1.1
Host: <redacted>
```

The server returned the challenge flag.

---

## Proof and Flag

The `/datasheet` endpoint successfully returned the contents of a file outside the intended datasheet directory.

```text
WEBVERSE{REDACTED}
```

The flag was retrieved through:

```text
/datasheet?doc=../../../../../../flag.txt
```

---

## Root Cause

The root cause was the use of untrusted user input as part of a filesystem path without canonicalization or boundary validation.

The vulnerable logic was likely equivalent to:

```php
<?php
$doc = $_GET['doc'];
readfile('/var/www/html/datasheets/' . $doc);
```

When the supplied value contained `../`, the operating system resolved the resulting path outside the intended directory.

For example:

```text
/var/www/html/datasheets/../../../../../../flag.txt
```

could resolve to:

```text
/flag.txt
```

The application trusted the requested path instead of confirming that the final canonical path remained inside the approved datasheet directory.

---

## Impact

An attacker could use the vulnerability to read any file accessible to the web server process.

Potentially exposed data could include:

- application source code;
- environment files;
- database credentials;
- API keys and secrets;
- system account information;
- private configuration files;
- challenge flags or other sensitive server-side files.

If exposed credentials were reusable elsewhere, the arbitrary file read could become the first stage of a broader compromise.

---

## Mitigation

### Use an Allowlist

The safest design is to avoid accepting filenames from the user. The application should accept a product identifier and map it to a predefined file.

```php
<?php
$documents = [
    'ne555' => 'ne555.pdf',
    'lm358' => 'lm358.pdf'
];

$id = $_GET['id'] ?? '';

if (!isset($documents[$id])) {
    http_response_code(404);
    exit('Document not found');
}

readfile('/var/www/html/datasheets/' . $documents[$id]);
```

### Validate the Canonical Path

When dynamic filenames are unavoidable, resolve the path with `realpath()` and verify that it remains inside the approved directory.

```php
<?php
$baseDir = realpath('/var/www/html/datasheets');
$requested = $_GET['doc'] ?? '';
$resolved = realpath($baseDir . DIRECTORY_SEPARATOR . $requested);

if (
    $resolved === false ||
    !str_starts_with($resolved, $baseDir . DIRECTORY_SEPARATOR)
) {
    http_response_code(400);
    exit('Invalid document path');
}

readfile($resolved);
```

### Additional Controls

The application should also:

- reject absolute paths and traversal sequences;
- avoid exposing filesystem errors;
- run the web server with minimal filesystem permissions;
- store secrets outside directories readable by the web process;
- log repeated traversal attempts;
- return files through internal identifiers rather than raw paths.

---

## Lessons

- File download parameters should always be treated as potential filesystem attack surfaces.
- A failed request for `../flag.txt` does not disprove path traversal; the target may require a different traversal depth.
- Known files such as `/etc/passwd` are useful for confirming arbitrary file read in controlled lab environments.
- Canonical path validation is more reliable than simply filtering the literal string `../`.
- The web server should never have unnecessary access to sensitive files.
