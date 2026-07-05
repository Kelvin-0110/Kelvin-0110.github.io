---
title: "Race Condition Leads to Like Counter Manipulation | CatTrap"
date: 2026-07-05 12:20:00 +0530
categories: [A04 - Insecure Design, Race Condition]
tags: [race-condition, business-logic, toctou, like-counter, burp-suite, webverse, webverse-pro]
platform: WebVerse Pro
author: Shivansh Sharma
image:
  path: /assets/images/posts/cattrap-race-condition.webp
  alt: "CatTrap race condition like counter manipulation"
---

## Lab Link

[CatTrap](https://dashboard.webverselabs-pro.com/mystery-challenges/cattrap)

---

## Overview

**CatTrap** was a WebVerse Pro challenge built around a small cat gallery where visitors could like photos.  
The application claimed to enforce a simple rule: each visitor could like each photo only once.

At first, the endpoint appeared to behave correctly. Sending the same like request twice normally returned an idempotent response and did not repeatedly increase the counter. However, when the same request was duplicated in Burp Suite and sent in parallel as a group, the backend failed to handle the like operation atomically.

This allowed multiple concurrent requests to pass the like-check at the same time, creating an inconsistent like ledger and eventually leaking the flag through an internal error.

---

## Objective

The objective was to identify the flaw in the like system and trigger the lab solve condition by abusing a race condition in the like endpoint.

---

## Vulnerability Identification

The home page displayed multiple cat cards, each with a like button.

The application created a visitor identifier using the `liker` cookie:

```http
Set-Cookie: liker=<visitor_id>; HttpOnly; Path=/; SameSite=Lax
```

When a heart button was clicked, the browser sent a request to the like endpoint:

```http
POST /like/1
```

A normal successful request returned a JSON response similar to this:

```http
HTTP/2 200 OK
Content-Type: application/json

{"liked":true,"likes":129}
```

Repeating the same request normally did not continuously increase the counter. This showed that the application attempted to enforce one like per visitor.

The important observation was that the protection worked during normal sequential testing, but the endpoint still needed to be tested under concurrent requests.

---

## Recon and Approach

The relevant endpoints were:

```text
POST /like/<photo_id>
POST /unlike/<photo_id>
```

The like request used the current visitor cookie:

```http
POST /like/6 HTTP/2
Host: redacted
Cookie: cf_clearance=<redacted>; liker=<same_valid_liker_cookie>
X-Requested-With: fetch
Origin: https://redacted
Referer: https://redacted/
Content-Length: 0
```

Testing showed that normal duplicate likes were handled safely.  
The next step was to check whether the backend processed the like check and the counter update as one atomic operation.

The suspected vulnerable logic was:

```text
1. Check whether this liker has already liked the photo
2. Increment the photo like counter
3. Record the liker in the like ledger
```

If these steps are not protected by a transaction, lock, or unique database constraint, multiple parallel requests can pass the first check before the first request finishes recording the like.

---

## Exploitation

The exploit was performed using **Burp Suite Repeater** with duplicate requests sent in parallel.

The captured request was:

```http
POST /like/6 HTTP/2
Host: redacted
Cookie: cf_clearance=<redacted>; liker=<same_valid_liker_cookie>
User-Agent: Mozilla/5.0
Accept: */*
Referer: https://redacted/
X-Requested-With: fetch
Origin: https://redacted
Content-Length: 0
```

The same request was duplicated multiple times in Burp Repeater and then sent together using Burp's parallel group send feature.

Steps followed:

```text
1. Capture a normal POST /like/6 request in Burp.
2. Send the request to Repeater.
3. Duplicate the Repeater tab multiple times.
4. Keep the exact same Cookie header in every duplicate request.
5. Select the duplicated Repeater tabs.
6. Use "Send group in parallel".
7. Watch for inconsistent like counts or a 500 error response.
```

The key detail is that every request must use the **same** visitor identity:

```http
Cookie: cf_clearance=<redacted>; liker=<same_valid_liker_cookie>
```

Using different or random `liker` values would create separate visitors and would not demonstrate the intended race condition.  
The bug was triggered by racing the same visitor's like request against itself.

---

## Proof and Flag

After the duplicated Burp Repeater requests were sent in parallel, the application entered an inconsistent state.

The like counter and internal like ledger no longer matched.  
This caused the backend to throw an internal `LikeLedgerError`.

The error response leaked an internal reference containing the flag:

```text
WEBVERSE{c610f5c0999bded694a66776342d84b0}
```

---

## Root Cause

The root cause was a **time-of-check to time-of-use race condition** in the like workflow.

The application checked whether a visitor had already liked a photo and then updated the like state as separate operations. Because these operations were not atomic, parallel requests could pass the validation check before the backend recorded the first successful like.

In short:

```text
The like-existence check and the like-counter update were not protected by an atomic transaction, unique constraint, or lock.
```

---

## Impact

An attacker could manipulate the like count despite the one-like-per-visitor rule.

Potential impact includes:

- Artificially inflating or deflating popularity metrics
- Breaking trust in ranking or voting systems
- Triggering inconsistent backend state
- Causing internal server errors
- Leaking sensitive debug information through error responses

In this challenge, the inconsistent state triggered a backend error that exposed the flag.

---

## Mitigation

The application should enforce the like operation atomically on the server side.

Recommended fixes:

- Store likes in a table with a unique constraint on `(photo_id, liker_id)`
- Perform the insert and counter update inside a database transaction
- Use an atomic upsert operation such as `INSERT ... ON CONFLICT DO NOTHING`
- Derive like counts from the ledger instead of maintaining an unsafe separate counter
- Avoid exposing internal exception details to users
- Return generic error messages for unexpected backend failures

A safer pattern would be:

```text
1. Begin transaction
2. Insert (photo_id, liker_id) into likes table with a unique constraint
3. If insert succeeds, increment count
4. Commit transaction
5. If duplicate key occurs, return the existing liked state
```

---

## Lessons Learned

Race conditions often hide behind features that look secure during normal testing.  
A single repeated request may behave correctly, but concurrent requests can expose flaws in how the backend handles shared state.

For voting, liking, inventory, payment, coupon, and booking systems, always test whether the business rule still holds under parallel requests.

The key takeaway from CatTrap:

```text
A business rule is only reliable if the state transition enforcing it is atomic.
```
