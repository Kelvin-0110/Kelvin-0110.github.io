# 🔐 Access Control Basics

## What is Access Control?

Access control ensures that users can only access resources they are authorized to use.

---

## Types

### Vertical Privilege Escalation

A user gains higher privileges (e.g., normal user → admin).

### Horizontal Privilege Escalation

A user accesses another user's data.

---

## Common Vulnerabilities

* Missing authentication checks
* Role-based access control flaws
* IDOR (Insecure Direct Object Reference)

---

## Example

Changing request parameter:
`user_id=101 → user_id=102`

---

## Summary

Access control vulnerabilities can lead to serious issues like unauthorized access and full system compromise.
