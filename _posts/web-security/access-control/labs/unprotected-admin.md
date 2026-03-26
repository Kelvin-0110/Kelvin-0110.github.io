---
layout: default
title: Unprotected Admin Functionality
tags: [access-control, auth-bypass, admin-panel]
---

[← Back to Labs](./)

**Platform:** PortSwigger  
**Category:** Access Control

---

## 🧠 Concept

This lab demonstrates **broken access control**, where sensitive functionality is accessible without proper authentication.

---

## 🎯 Objective

Access the admin panel and delete the user `carlos`.

---

## 🔍 Reconnaissance

### Step 1: Check robots.txt

Accessing:

```
/robots.txt
```

### 🔍 Finding

The file reveals a hidden admin endpoint:

```
/administrator-panel
```

### 📸 Evidence

![robots.txt output showing admin path](/assets/images/unprotected-admin-1.png)

👉 This indicates a potentially accessible admin interface.

---

## 💥 Exploitation

### Step 2: Access admin panel

Navigating to:

```
/administrator-panel
```

### 📸 Evidence

![Admin panel accessed without authentication](/assets/images/unprotected-admin-2.png)

### Step 3: Perform action

* Accessed admin interface
* Located user management section
* Deleted user `carlos`

## 📸 Proof

### Lab completion confirmation

![Lab solved confirmation](/assets/images/unprotected-admin-3.png)

* Admin panel accessible without login
* User `carlos` successfully deleted
* Application confirms successful exploitation (lab solved)


## 🛡️ Impact

* Unauthorized access to admin functionality
* Ability to delete users or manipulate data
* Potential full system compromise

---

## 🛠️ Mitigation

* Enforce authentication for all admin endpoints
* Implement role-based access control (RBAC)
* Avoid exposing sensitive paths in `/robots.txt`

---

## 🌍 Real-World Insight

Developers sometimes rely on hidden URLs instead of proper access control. Attackers can easily discover such endpoints using files like `/robots.txt`, JavaScript analysis, or directory brute-forcing tools.

---
