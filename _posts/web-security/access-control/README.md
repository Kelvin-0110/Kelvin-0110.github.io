---

layout: default
title: 🔓 Access Control
------------------------

## 🧠 Overview

Access control vulnerabilities occur when an application does not properly enforce restrictions on what authenticated users are allowed to do.

These flaws can allow attackers to:

* Access unauthorized data
* Perform actions as other users
* Escalate privileges to admin level

---

## 📚 Topics Covered

* Vertical privilege escalation
* Horizontal privilege escalation
* IDOR (Insecure Direct Object Reference)
* Parameter-based access control flaws
* Security through obscurity flaws

---

## 🧪 Labs

* 🔥 [Unprotected Admin Functionality](./labs/unprotected-admin.md)
* 🔥 [Unprotected Admin Functionality with Unpredictable URL](./labs/unprotected-admin-url.md)
* 🔥 [User Role Controlled by Request Parameter](./labs/user-role-request-parameter.md)

---

## 🎯 Key Takeaways

* Never trust client-side data for authorization
* Always enforce access control on the server side
* Hidden endpoints are not secure
* Validate user permissions for every request

---

## 🔗 Navigation

* [← Back to Home](/)
* [View All Labs](./labs/)
