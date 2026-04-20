---
title: "Broken Access Control – Privilege Escalation via Client-Controlled Cookie | Privilege Escalation via Client-Controlled Cookie"
date: 2026-03-28 06:00:00 +0530
categories: [Web Security, Access Control]
tags: [broken-access-control, privilege-escalation, cookie-manipulation, authorization-bypass, portswigger]
platform: PortSwigger
author: Shivansh Sharma
image:
  path: /assets/images/posts/broken-access-control.webp
  alt: Admin Panel Access via Cookie Manipulation
---

## 🧠 Overview

This lab demonstrates a **privilege escalation vulnerability** where user roles are controlled via client-side input.

By modifying a cookie value, an attacker can escalate privileges and gain unauthorized administrative access.

---

## 🎯 Objective

- Access the admin panel  
- Delete the user `carlos`  

---

## 🔍 Reconnaissance

### Step 1: Login as Normal User

Logged in using the provided credentials:

```
wiener:peter
```

### 📸 Evidence

![User account dashboard](/assets/images/favicons/user-role-request-parameter-1.png)

This confirms access as a regular user.

---

### Step 2: Inspect Cookies

Browser cookies were analyzed using developer tools or a proxy.

### 🔎 Observation

A role-related cookie was identified:

```http
admin=false
```

### 📸 Evidence

![Cookie showing admin=false](/assets/images/favicons/user-role-request-parameter-2.png)

This indicates that role-based access control is handled on the client side.

---

## 💥 Exploitation

### Step 3: Modify Cookie

The cookie value was modified from:

```http
admin=false
```

to:

```http
admin=true
```

### 📸 Evidence

![Cookie modified to admin=true](/assets/images/favicons/user-role-request-parameter-3.png)

---

### Step 4: Access Admin Panel

Navigated to:

```http
/admin
```

### 📸 Evidence

![Admin panel access](/assets/images/favicons/user-role-request-parameter-4.png)

Administrative access was granted without proper authorization.

---

### Step 5: Perform Administrative Action

- Accessed admin interface  
- Navigated to user management  
- Deleted user `carlos`  

---

## Proof of Exploitation

### ✅ Result

- Privilege escalation achieved via cookie manipulation  
- Unauthorized administrative access obtained  
- User `carlos` successfully deleted  

---

## 🛡️ Impact

- Privilege escalation through client-side manipulation  
- Unauthorized access to administrative functionality  
- Potential data modification or system compromise  

---

## 🛠️ Mitigation

- Never trust client-side input for authorization decisions  
- Enforce role validation on the server side  
- Implement secure session management  
- Validate all privilege-related data server-side  

---

## 🌍 Real-World Insight

This vulnerability is common in poorly designed applications where role-based access control is handled on the client side.

Attackers often manipulate:

- Cookies  
- JWTs  
- Request parameters  

to escalate privileges and gain unauthorized access.

---
