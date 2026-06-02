---
title: "Exposed Git Repository Information Disclosure | Loop & Roam Records"
date: 2026-05-27 18:30:00 +0530
categories: [A02 - Security Misconfiguration, Information Disclosure]
tags: [webverselabs-pro, git, exposed-git, information-disclosure, source-code-disclosure, security-misconfiguration, reconnaissance]
platform: WebVerse Labs
author: Shivansh Sharma
image:
  path: /assets/images/posts/loop-and-roam-records.webp
  alt: Loop and Roam Records Exposed Git Repository
---

## Lab Link

Lab: [Herbalist Remedies](https://dashboard.webverselabs-pro.com/challenges/b-side)

---

## Overview

**Loop & Roam Records** is an independent record label website that unintentionally exposes its entire Git repository to the public through the web server.

Although a sensitive `.env` file was removed shortly after being committed, Git permanently preserves historical commits unless the repository history is rewritten. Because the `.git` directory was deployed to the public web root, attackers can reconstruct the repository and inspect previous commits for secrets.

This challenge demonstrates how source code exposure and leaked Git history can lead to disclosure of sensitive information long after a file has been deleted.

---

## Objective

Recover the exposed Git repository, inspect its commit history, and locate the flag hidden within historical commits.

---

## Scenario

> Loop & Roam was founded in a Detroit garage in 2011 by a bassist named Jovan who got tired of touring and wanted to put out his friends' records. The site is maintained by Aphra, a contract designer who was learning git when she built it. One night in February 2024 she committed a `.env` with prod credentials, noticed an hour later, ran `git rm .env`, and pushed. The deploy script `rsync`s the working tree — including the `.git/` directory — to the public web root.

---

## Reconnaissance

While enumerating the web application, a publicly accessible Git repository can be discovered:

```text
https://fa5c1182-4065-b-side-fab3a.challenges.webverselabs-pro.com/.git/
```

Accessing this endpoint reveals that the repository metadata is exposed to unauthenticated users.

An exposed `.git` directory is a serious security issue because it often allows attackers to reconstruct the entire source code repository.

---

## Vulnerability Analysis

A Git repository contains much more than the current version of the source code.

The `.git` directory stores:

- Commit history
- Branches
- Tags
- Deleted files
- Historical secrets
- Configuration data
- Developer notes

Even if a sensitive file is removed later:

```bash
git rm .env
git commit
```

the contents remain recoverable from earlier commits.

As a result, secrets committed even briefly may remain accessible forever through Git history.

---

## Exploitation

### Step 1: Confirm Repository Exposure

Browse to:

```text
/.git/
```

Example:

```text
https://fa5c1182-4065-b-side-fab3a.challenges.webverselabs-pro.com/.git/
```

The repository metadata is publicly accessible.

---

### Step 2: Dump the Repository

Use `git-dumper` to reconstruct the repository locally.

```bash
git-dumper \
https://fa5c1182-4065-b-side-fab3a.challenges.webverselabs-pro.com/.git/ \
loot
```

Successful execution downloads the exposed repository.

---

### Step 3: Move Into the Repository

```bash
cd loot
```

The repository can now be examined like any normal Git project.

---

### Step 4: Search Commit History

Inspect all commits and patches for sensitive information.

```bash
git log -p --all | grep -iE "(flag|secret|WEBVERSE)"
```

---

## Understanding the Command

### `git log -p --all`

Displays:

```text
git log
```

- Complete commit history

```text
-p
```

- Includes code diffs and file modifications

```text
--all
```

- Searches every branch and reference

This produces a full view of every change ever committed.

---

### `grep -iE "(flag|secret|WEBVERSE)"`

Filters the output:

```text
grep
```

- Searches matching lines

```text
-i
```

- Case-insensitive matching

```text
-E
```

- Enables extended regular expressions

Pattern:

```text
(flag|secret|WEBVERSE)
```

Matches:

```text
flag
secret
WEBVERSE
```

This quickly identifies potentially sensitive information hidden within commit history.

---

## Proof of Exploitation

Searching historical commits reveals the challenge flag:

```text
WEBVERSE{....}
```

The flag exists in repository history even though it is no longer present in the current source tree.

---

## Impact

Successful exploitation allows an attacker to recover:

- Source code
- API keys
- Database credentials
- Environment variables
- Internal documentation
- Deleted files
- Historical secrets
- Developer comments

In real-world environments, exposed Git repositories frequently result in complete compromise because attackers can obtain credentials that provide direct access to production infrastructure.

---

## Root Cause

Two separate mistakes created the vulnerability.

### Exposed `.git` Directory

The deployment process copied the repository metadata into the public web root.

Example:

```bash
rsync -av . /var/www/html/
```

This unintentionally included:

```text
.git/
```

---

### Secrets Committed to Git

A sensitive file was committed:

```text
.env
```

Although later removed:

```bash
git rm .env
```

Git preserved the contents within historical commits.

The secret therefore remained recoverable.

---

## Mitigation

### Never Expose `.git`

Block access at the web server level.

Apache:

```apache
RedirectMatch 404 /\.git
```

Nginx:

```nginx
location ~ /\.git {
    deny all;
}
```

---

### Exclude Git Metadata During Deployment

Deploy only application files.

Example:

```bash
rsync --exclude='.git' ...
```

---

### Use Secret Scanning

Implement tools such as:

- GitLeaks
- TruffleHog
- GitHub Secret Scanning

to detect accidental credential commits.

---

### Remove Secrets Properly

Deleting a file is not sufficient.

Use history-rewrite tools:

```bash
git filter-repo
```

or

```bash
BFG Repo-Cleaner
```

to permanently remove secrets from repository history.

---

### Rotate Exposed Credentials

If a secret has ever been committed:

1. Assume compromise.
2. Revoke it.
3. Generate a replacement.
4. Audit associated systems.

---

## Real-World Insight

Exposed Git repositories remain one of the most common discoveries during web application reconnaissance. Bug bounty programs regularly award findings where attackers recover API keys, cloud credentials, database passwords, or internal source code from publicly accessible `.git` directories.

Many organizations remove secrets from current files but forget that Git preserves every previous version unless history is rewritten.

---

## Vulnerability Identification

This challenge is primarily an **Exposed Git Repository Information Disclosure** vulnerability.

### Classification Hierarchy

**OWASP Top 10:2025**

```text
A02 - Security Misconfiguration
 └── Source Code Exposure
      └── Exposed Git Repository
           └── Historical Secret Disclosure
```

### CWE Mapping

```text
CWE-538
Insertion of Sensitive Information into Externally-Accessible File or Directory
```

---

## Key Takeaway

Removing a secret from the latest version of a project does not remove it from Git history. If a `.git` directory becomes publicly accessible, attackers can reconstruct the repository, review historical commits, and recover sensitive information that developers believed had already been deleted.