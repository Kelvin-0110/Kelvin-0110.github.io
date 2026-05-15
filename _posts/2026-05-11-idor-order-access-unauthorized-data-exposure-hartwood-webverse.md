---
title: "IDOR in Order Access – Unauthorized Order Data Exposure | Hartwood "
date: 2026-05-11 01:00:00 +0530
categories: [Web Security, IDOR]
tags: [idor, access-control, authorization, burp-suite, api-security, webverse, brute-force, webversepro]
platform: Webverse
author: Shivansh Sharma
image:
  path: /assets/images/posts/hartwood-idor.webp
  alt: IDOR in order endpoint leading to sensitive order exposure
---

## Overview

This lab is based on a fictional e-commerce platform, Hartwood & Co., which sells pet-related products. The application was rushed into production, and a test order was accidentally left in the live database.

The issue revolves around weak authorization checks on order resources, allowing direct access to other users' order details by manipulating the `orderid` parameter.

## Lab Link

- Lab: [DocketHive – WebVerse](https://dashboard.webverselabs-pro.com/mystery-challenges/hartwood-and-co)

---

## Objective

Identify insecure direct object reference (IDOR) in the order management flow and extract sensitive order information by abusing predictable identifiers.

---

## Reconnaissance

While browsing the application, a product page was discovered with a predictable ID-based parameter:

```
https://28a0c3e1-4065-hartwood-and-co-730d1.events.webverselabs-pro.com/product.php?id=HW-001
```

This confirmed the application uses sequential or predictable identifiers for product objects, which often extends to other components like orders.

---

## Exploitation

After adding a product to the cart and placing an order, the following request was observed in Burp Suite:

```
GET /order.php?orderid=18 HTTP/1.1
```

This endpoint directly exposes order details based on the `orderid` parameter without enforcing ownership validation.

### Testing IDOR

By modifying the order ID manually:

```
GET /order.php?orderid=19 HTTP/1.1
```

The application returned another user's order details successfully, confirming IDOR.

---

## Automated Enumeration

To scale the attack, a numeric wordlist was generated (1–200) and used in Burp Suite Intruder:

```
GET /order.php?orderid=<payload_position> HTTP/1.1
```

Attack configuration:
- Attack type: Sniper
- Payload: Sequential numeric list (1–200)
- Target: `/order.php?orderid=`

This allowed systematic enumeration of all accessible order objects.

---

## Proof of Exploitation

During enumeration, sensitive information was discovered in order ID 120:

```
GET /order.php?orderid=120 HTTP/1.1
```

The response contained internal data inside the **Promo & internal notes** section.

```
WEBVERSE{....}
```

This flag confirms successful exploitation of IDOR leading to unauthorized data exposure.

---

## Impact

This vulnerability allows:
- Unauthorized access to customer orders
- Exposure of internal notes and promotional data
- Potential leakage of business-sensitive or financial information
- Large-scale enumeration of user data using brute force

In real systems, this could escalate into privacy violations or compliance issues.

---

## Mitigation

To prevent this issue:
- Enforce strict access control checks on every object request
- Verify ownership before returning order data
- Replace predictable IDs with non-enumerable references (UUIDs alone are not enough)
- Implement rate limiting to prevent brute-force enumeration
- Log and monitor repeated object access patterns

---

## Real-World Insight

IDOR is one of the most common and impactful access control flaws in modern applications. Even when systems use UUIDs or "secure IDs", poor authorization logic can still expose sensitive data.

The key takeaway is simple:  
Never trust object references coming from the client side without verifying access rights on the server.