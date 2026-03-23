# Path Traversal — Lab Write-ups

> **Topic:** Path Traversal (Directory Traversal)  
> **Platform:** PortSwigger Web Security Academy  
> **Labs Completed:** All  
> **Tools Used:** Burp Suite (Repeater), URL encoding

---

## What is Path Traversal?

Path traversal (also called directory traversal) lets you read files from the server's filesystem that the application wasn't meant to expose. By manipulating file path parameters with sequences like `../`, you can navigate outside the intended directory and access system files.

The classic target is `/etc/passwd` on Linux (or `C:\Windows\win.ini` on Windows), but in real engagements this can mean reading application source code, config files, private keys, or credentials.

---

## Lab 1 — Simple Path Traversal
**Level:** Apprentice

**What the lab is about:**  
Product images are loaded via a `filename` parameter. No sanitization. Goal: read `/etc/passwd`.

**What I did:**  
Original request:
```
GET /image?filename=17.jpg
```

Modified to:
```
GET /image?filename=../../../etc/passwd
```

The response returned the full contents of `/etc/passwd`. Each `../` moves one directory up from wherever the images are stored. Three levels was enough to reach the filesystem root.

**Key takeaway:**  
File path parameters are a primary target. `filename=`, `path=`, `file=`, `img=` — all worth testing with traversal sequences.

**Mitigation:**  
Canonicalize the resolved path and verify it starts with the intended base directory before opening the file.

---

## Lab 2 — Path Traversal with Absolute Path Bypass
**Level:** Practitioner

**What the lab is about:**  
The application strips `../` sequences but doesn't validate absolute paths. You can just provide a direct absolute path instead.

**What I did:**  
```
GET /image?filename=/etc/passwd
```

No traversal sequences needed — just supplied the full path directly. The application passed it straight to the filesystem.

**Key takeaway:**  
Stripping `../` isn't enough. You also need to block absolute paths. Many developers forget this case.

---

## Lab 3 — Path Traversal with Sequences Stripped Non-Recursively
**Level:** Practitioner

**What the lab is about:**  
The application removes `../` from input, but only does one pass — it doesn't recurse. So if you nest the sequences, the outer ones survive after stripping.

**What I did:**  
```
GET /image?filename=....//....//....//etc/passwd
```

After stripping `../` once:  
`....//` → `../`  (the middle `../` is removed, leaving the outer dots and slash)

After one pass of stripping, what remains is a valid traversal sequence.

**Key takeaway:**  
Always test nested/doubled sequences when basic traversal is blocked: `....//`, `..././`, `....\/`. Non-recursive stripping is a very common mistake.

---

## Lab 4 — Path Traversal with Superfluous URL-Decode
**Level:** Practitioner

**What the lab is about:**  
The application decodes the URL before stripping traversal sequences — or the stripping happens before URL decoding. Either way, URL-encoding the payload bypasses the filter.

**What I did:**  
URL-encoded `../` as `%2e%2e%2f`:
```
GET /image?filename=%2e%2e%2f%2e%2e%2f%2e%2e%2fetc/passwd
```

The WAF/filter checked the raw string (didn't find `../`), but the filesystem call happened after URL decoding, so it saw `../../../etc/passwd`.

**Key takeaway:**  
Double encoding also works: `%252e%252e%252f` (where `%25` is the encoded `%`). Filters that check before decoding are bypassable this way.

---

## Lab 5 — Path Traversal with Validation of Start of Path
**Level:** Practitioner

**What the lab is about:**  
The application requires the filename to start with `/var/www/images/`. It validates the prefix — but doesn't stop traversal after the prefix.

**What I did:**  
```
GET /image?filename=/var/www/images/../../../etc/passwd
```

The prefix `/var/www/images/` is present (passes validation), and then `../../../` navigates back up to the root and into `/etc/passwd`.

**Key takeaway:**  
A prefix check is useless without also checking the canonicalized resolved path. Always resolve the full path first, then check if it falls within the allowed base directory.

---

## Lab 6 — Path Traversal with Validation of File Extension
**Level:** Practitioner

**What the lab is about:**  
The application checks that the filename ends with a valid image extension like `.png`. You can still traverse — just add the extension as a null byte or append it after the traversal.

**What I did:**  
```
GET /image?filename=../../../etc/passwd%00.png
```

The null byte `%00` terminates the string at the OS level in some languages (particularly older PHP/C-based systems). The application sees `.png` at the end and passes validation, but the filesystem only reads up to the null byte.

**Key takeaway:**  
Null byte injection is an older technique but still relevant in legacy systems. Some modern runtimes strip null bytes, but it's always worth trying.

---

*Path traversal looks simple but the defense bypass techniques are surprisingly varied. The core lesson: always resolve paths to their canonical form before any validation or file access.*
