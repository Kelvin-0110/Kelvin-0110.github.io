---

layout: default
title: 🔥 Lab: Unprotected Admin Functionality with Unpredictable URL
tags: [access-control, admin-panel, information-disclosure]
-----------------------------------------------------------

[← Back to Labs](./)

<span class="badge">Platform: PortSwigger</span>
<span class="badge">Category: Access Control</span>

---

## 🧠 Concept

This lab demonstrates that **security through obscurity is ineffective**. Even if sensitive endpoints use unpredictable URLs, they can still be exposed through client-side code.

---

## 🎯 Objective

Discover the hidden admin panel and delete the user `carlos`.

---

## 🔍 Reconnaissance

### Step 1: Inspect page source

* Viewed HTML source code of the application

### 🔍 Finding

Discovered a hidden admin endpoint:

```http
/admin-wfqh8r
```

### 📸 Evidence

![Hidden admin endpoint in source code](/assets/images/unprotected-admin-url-1.png)

👉 The admin panel location is exposed in client-side code.

---

## 💥 Exploitation

### Step 2: Access admin panel

Navigated to:

```http
/admin-wfqh8r
```

### 📸 Evidence

![Admin panel accessed via hidden URL](/assets/images/unprotected-admin-url-2.png)

### Step 3: Perform action

* Accessed admin interface
* Located user management
* Deleted user `carlos`

---

## 📸 Proof

### Lab completion confirmation

![Lab solved confirmation](/assets/images/unprotected-admin-url-3.png)

* Admin panel accessible without proper protection
* User `carlos` successfully deleted
* Application confirms successful exploitation

---

## 🛡️ Impact

* Sensitive admin endpoints exposed via client-side code
* Attackers can discover hidden functionality
* Leads to unauthorized administrative actions

---

## 🛠️ Mitigation

* Do not rely on hidden or unpredictable URLs for security
* Enforce authentication and authorization checks on all endpoints
* Avoid exposing sensitive routes in client-side code

---

## 🌍 Real-World Insight

Attackers frequently analyze:

* HTML source code
* JavaScript files
* API responses

to discover hidden endpoints. Relying on obscurity instead of proper access control is a common real-world vulnerability.

---

[← Back to Labs](./)
