---

layout: default
title: 🔥 Lab: User Role Controlled by Request Parameter
tags: [access-control, privilege-escalation, cookies]
-----------------------------------------------------

[← Back to Labs](./)

<span class="badge">Platform: PortSwigger</span>
 <span class="badge">Category: Access Control</span>

---

## 🧠 Concept

This lab demonstrates **privilege escalation via client-controlled parameters**, where user roles are determined by a modifiable cookie.

---

## 🎯 Objective

Access the admin panel and delete the user `carlos`.

---

## 🔍 Reconnaissance

### Step 1: Login as normal user

* Logged in using provided credentials:

  ```
  wiener:peter
  ```

### 📸 Evidence

![User account dashboard](/assets/images/user-role-1.png)

👉 Verified access as a regular user.

---

### Step 2: Inspect cookies

* Checked browser cookies using developer tools / proxy

### 🔍 Finding

Discovered a cookie parameter:

```http
admin=false
```

### 📸 Evidence

![Cookie showing admin=false](/assets/images/user-role-2.png)

👉 Indicates role is controlled client-side.

---

## 💥 Exploitation

### Step 3: Modify cookie

Changed:

```http
admin=false
```

to:

```http
admin=true
```

### 📸 Evidence

![Cookie modified to admin=true](/assets/images/user-role-3.png)

---

### Step 4: Access admin panel

* Navigated to `/admin`
* Gained administrative access

### 📸 Evidence

![Admin panel access](/assets/images/user-role-4.png)

---

### Step 5: Perform action

* Located user management
* Deleted user `carlos`

---

## 📸 Proof

### Lab completion confirmation

* Privilege escalation achieved via cookie manipulation
* Unauthorized admin access obtained
* User `carlos` successfully deleted

---

## 🛡️ Impact

* Attackers can escalate privileges by modifying client-side data
* Full administrative control without authentication
* Leads to data manipulation or system compromise

---

## 🛠️ Mitigation

* Never trust client-side input for authorization decisions
* Enforce role checks on the server side
* Use secure session management
* Validate all privilege-related data server-side

---

## 🌍 Real-World Insight

This vulnerability is common in poorly designed applications where role-based access control is handled on the client side. Attackers frequently manipulate cookies, JWTs, or request parameters to escalate privileges.

---

[← Back to Labs](./)
